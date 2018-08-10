# IOS Template
Contains many of the security best practices for router security, mainly targeted for internet facing routers.

This uses ansible and jinja2 templates, variables are in the playbook to simplify the ammount of files.

List of routers is in the inventory file, this is used for the hostname and filename.conf of the config.

## Installation

- Install ansiblem any version should work.
- Clone the repo
- Edit the files to your liking
- Run the playbook
`$ ansible-playbook playbook.yml -i inventory
ansible-playbook playbook.yml -i inventory

PLAY [all] ******************************************************************************************************************

TASK [write the config file] ************************************************************************************************
changed: [inet-rtr1]

PLAY RECAP ******************************************************************************************************************
inet-rtr1                  : ok=1    changed=1    unreachable=0    failed=0

localhost:~/projects/ios-template`