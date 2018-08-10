# IOS Template
Contains many of the security best practices for router security, mainly targeted for internet facing routers.

This uses ansible and jinja2 templates, variables are in the playbook to simplify the ammount of files.

List of routers is in the inventory file, this is used for the hostname and filename.conf of the config.

## Installation
- Install ansible any version should work.
- Clone the repo
- Edit the files to your liking
- Run the playbook

```
$ ansible-playbook playbook.yml -i inventory

PLAY [all] ******************************************************************************

TASK [write the config file] ************************************************************
changed: [inet-rtr1]

PLAY RECAP ******************************************************************************
inet-rtr1                  : ok=1    changed=1    unreachable=0    failed=0

$ cat inet-rtr1.conf

! Good for caching small packets like terminal sessions
service nagle
service tcp-keepalives-in
service tcp-keepalives-out
!
! Show copious timestamps in our logs
service timestamps debug datetime msec show-timezone localtime
service timestamps log datetime msec show-timezone localtime
! Ensures all passwords and secrets are obfuscated when showing configuration
service password-encryption
no service dhcp
no ip bootp server
!
hostname inet-rtr1
!
logging buffered 16384 debugging
no logging console
! ’secret’ ensures MD5 is used when ‘service password encryption’ is used.
enable secret Ch@ngeM3S00n!
no enable password
!
! Use TACACS+, local fallback for AAA
! aaa new-model
! aaa authentication login default group tacacs+ local-case
! aaa authentication enable default group tacacs+ enable
! aaa authorization commands 15 default group tacacs+ local
! aaa accounting exec default stop-only group tacacs+
! aaa accounting commands 15 default stop-only group tacacs+
! aaa accounting network default stop-only group tacacs+
!
! tacacs-server host 10.1.1.1
! tacacs-server key tacacs_key123
!
! Use local (case sensitive) for AAA when theres no tac server
aaa authentication login default group local-case
aaa authorization exec default group local
!
ip dhcp bootp ignore
! In the event that TACACS+ fails, use case-sensitve local authentication.
username RAdmin privilege 15 secret Ch@ngeM3v3ryS00n!
!
! Logging and archive the commands and changes
archive
 log config
  logging enable
  logging size 500
  notify syslog
  hidekeys
  path flash:backup-
  write-memory
  maximum 8
!
! Ensure TCL doesn’t use an initilizaion file where available.
no scripting tcl init
no scripting tcl encdir
!
! Enable the netflow top talkers feature. You can see the top N talkers (50 in this example)
ip flow-top-talkers
 top 50
 sort-by packets
!
! Don’t run the HTTP server.
no ip http server
no ip http secure-server
!
! Allow us to use the low subnet and go classless
ip subnet-zero
ip classless
!
! Disable noxious services
no service pad
no ip source-route
no ip finger
no ip bootp server
no ip domain-lookup
!
! Block brute force login attempts while maintaining access for legitimate source addresses.
login block-for 100 attempts 15 within 100
login quiet-mode access-class 100
login on-failure log
login on-success log
!
! Catch crash dumps; very important with a “security router.”
! ip ftp username rtr-ftp-user
! ip ftp password rtr-ftp-passwd
! Give our core dump files a unique name.
exception core-file secure-router01-core
exception protocol ftp
exception dump 10.1.1.1
!
! Fire up CEF for both performance and security.
ip cef
!

clock timezone UTC 0 0
ntp update-calendar
ntp server 10.1.1.1
!
! Configure the loopback0 interface as the source of our log  messages.
! int loopback0
! ip address 10.10.10.10 255.255.255.255
! no ip redirects
! no ip unreachables
! no ip proxy-arp
!
! Configure null0 as a place to send bad packets
interface null0
 no ip unreachables
!
interface Ethernet2/0
 description External Internet Interface
 ip address 209.209.209.2 255.255.255.252
 ! Apply our template ACL
 ip access-group 2010 in
 ! Allow UDP to occupy no more than 2 Mb/s of the pipe.
 rate-limit input access-group 150 2010000 250000 250000 conform-action transmit exceed-action drop
 ! Allow ICMP to occupy no more than 500 Kb/s of the pipe.
 rate-limit input access-group 160 500000 62500 62500 conform-action transmit exceed-action drop
 ! Allow multicast to occupy no more than 5 Mb/s of the pipe.
 rate-limit input access-group 170 5000000 375000 375000 conform-action transmit exceed-action drop
 ! Don’t send redirects.
 no ip redirects
 ! Don’t send unreachable. NOTE WELL that this may break PMTU discovery.
 no ip unreachables
 ! Don’t propogate smurf attacks.
 no ip directed-broadcast
 ! Don’t pretend to be something you’re not.
 no ip proxy-arp
 ! Do not reveal our netmask
 no ip mask-reply
 ! Log bad access attempts
 ip accounting access-violations
 !
 ! Keep flow data for analysis. If possible, export it to a cflowd server.
 ip route-cache flow
 ! When you configure anything to do with ntp on an IOS box, it will start listening on all interfaces.
 ntp disable
 ! Disable Maintenance Operations Protocol on all interfaces
 no mop enable
!
interface Ethernet2/1
 description Internal Interface
 ip address 10.1.1.1 255.255.255.0
 ip access-group 115 in
 no ip redirects
 no ip unreachables
 no ip directed-broadcast
 no ip proxy-arp
 ip accounting access-violations
 no ip mask-reply
 ip route-cache flow
 no mop enable
!
!
! Netflow config
! ip flow-export source loopback0
! ip flow-export destination 192.0.2.34 2055
! ip flow-export version 5 origin-as
!
! Log anything interesting to the loghost. Capture all of
! the logging output with FACILITY LOCAL5.
logging trap debugging
logging facility local5
! Syslog config
! logging source-interface Loopback0
! logging 10.1.1.1
!
access-list 100 remark Management Access ACL
access-list 100 permit tcp 10.1.1.0 0.0.0.255 host 0.0.0.0 range 22 23 log-input
access-list 100 deny ip any any log-input
!
!
! Configure an ACL that prevents spoofing from within our network.
! This ACL assumes that we need to access the Internet only from the
access-list 115 remark Anti-spoofing ACL
access-list 115 permit ip 209.209.209.0 0.0.0.255 any
! Now log all other such attempts.
access-list 115 deny ip any any log-input
!
! Rate limit (CAR) ACLs for UDP, ICMP, and multicast.
access-list 150 remark CAR-UDP ACL
access-list 150 permit udp any any
access-list 160 remark CAR-ICMP ACL
access-list 160 permit icmp any any
access-list 170 remark CAR-Multicast ACL
access-list 170 permit ip any 224.0.0.0 15.255.255.255
!
! Deny any packets from the RFC 1918, IANA reserved, test,
! multicast as a source, and loopback netblocks to block
! attacks from commonly spoofed IP addresses.
access-list 2010 remark Anti-bogon ACL
! Claims it came from the inside network, yet arrives on the
! outside. Do not use this if CEF has been configured to take care of spoofing.
access-list 2010 deny ip 209.209.209.0 0.0.0.255 any log-input
access-list 2010 deny ip 209.209.209.0 0.0.0.255 any log-input
! Drop all ICMP fragments
access-list 2010 deny icmp any any fragments log-input
access-list 2010 permit ip any 209.209.209.0 0.0.0.255
access-list 2010 deny ip any any log-input
!
! Do not share CDP information
no cdp run
! SNMP is VERY important, particularly with MRTG.
! Treat the COMMUNITY string as a password – keep it difficult to guess.
! For SNMP versions 1-2
snmp-server community d0nt3v3nTry RO 100
!
! Introduce ourselves with an appropriately stern banner.
banner motd %
Access to this device or the attached
networks is prohibited without express written permission.
Violators will be prosecuted to the fullest extent of both civil
and criminal law.

%
!
line con 0
 exec-timeout 5 0
 transport input none
line aux 0
 exec-timeout 5 0
line vty 0 4
 access-class 100 in
 exec-timeout 5 0
! Enable SSH connectivity.
 ip domain-name edge-router.io
! Disable SSHv1
 ip ssh version 2
 transport input ssh
!
! End of the configuration.

```