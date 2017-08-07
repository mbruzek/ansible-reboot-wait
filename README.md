# Ansible reboot and wait for boot

This is a small Ansible project with two tasks one to reboot systems, and wait
for systems to boot.

I had significant trouble getting the task to wait for boot to complete without
error. I created this repository as the smallest example of my issue and took
my questions to #ansible on irc.freenode.net.

Many thanks to [@halberom](https://github.com/halberom) for working with me to
resolve the issue that is detailed below!

### TL; DR
The `local_action` task waits for the systems to boot fails when forced to
use an ssh connection, it should be "local".

## Problem
I was testing Ansible with some Raspberry Pi machines and needed to reboot the
systems to pick up changes that were made.

Since a new Raspberry Pi does not have any authorized ssh keys, I configured
the playbook to use ssh with the default user and password as variables.

```yml
vars:
  ansible_connection: ssh
  ansible_ssh_user: pi
  ansible_ssh_pass: raspberry
```

The first task reboots the systems. This required using `async`, `poll` and
`ignore_errors` to execute the reboot command that will terminate the ssh
connection.

I found a verified solution on the Internet for a second task that used a
`local_action` and `wait_for` directive in the task. This did not work for my
playbook and I did not understand why. Here is the solution as it was written:

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

As it turns out this task failed because the `ansible_connection` was set to
`ssh` for the entire playbook. When the `local_action` tried to connect to the
localhost via ssh the task failed with the error:
```
TASK [Wait for system to boot] **************************************************************************
fatal: [192.168.1.159]: UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh: ssh: connect to host localhost port 22: Connection refused\r\n", "unreachable": true}
fatal: [192.168.1.125]: UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh: ssh: connect to host localhost port 22: Connection refused\r\n", "unreachable": true}
        to retry, use: --limit @/home/user/workspace/ansible/ansible-reboot-wait/playbook.retry
```

## Solution
The solution is to change `ansible_connection` to "local" for the `local_action`
task. This can be done by setting a task level variable or removing the
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
