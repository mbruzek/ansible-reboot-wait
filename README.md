# Ansible reboot and wait for boot

This is the smallest Ansible project with two tasks one to reboot systems, and
the wait for systems to boot.

I had significant trouble getting the task to wait for boot to complete without
error. I created this repository as the smallest example of my issue and took
my questions to #ansible on irc.freenode.net.

Many thanks to @halberom for working with me to resolve the issue that is
detailed below!

## Problem
I was testing Ansible with some raspberry pi machines and needed to reboot to
pick up changes that were made.

```yml
vars:
  ansible_connection: ssh
  ansible_ssh_user: pi
  ansible_ssh_pass: raspberry
```
Using the `local_action` and `wait_for` task did not work for my playbook.

```yml
- name: Wait for system to boot
  become: false
  local_action: wait_for
  args:
    host: "{{ inventory_hostname }}"
    port: 22
    state: started
    delay: 15
    timeout: 300
```

As it turns out this was because the `ansible_connection` was set to `ssh` for
the entire playbook. When the local_action tried to connect to the localhost
via ssh the task failed with the error:
```
TASK [Wait for system to boot] **************************************************************************
fatal: [192.168.1.159]: UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh: ssh: connect to host localhost port 22: Connection refused\r\n", "unreachable": true}
fatal: [192.168.1.125]: UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh: ssh: connect to host localhost port 22: Connection refused\r\n", "unreachable": true}
        to retry, use: --limit @/home/user/workspace/ansible/ansible-reboot-wait/playbook.retry
```

## Solution
The solution is to change `ansible_connection` to "local" for the `local_action`
action. This can be done by setting a task level variable or removing the
ssh set in the beginning.

```yml
- name: Wait for system to boot
  become: false
  vars:
    ansible_connection: local
  local_action: wait_for
  args:
    host: "{{ inventory_hostname }}"
    port: 22
    state: started
    delay: 15
    timeout: 300
```

## Research

* https://github.com/ansible/ansible/issues/14413
* https://gist.github.com/infernix/a968f23c4f4e1d6723e4
* https://stackoverflow.com/questions/23877781/how-to-wait-for-server-restart-using-ansible
