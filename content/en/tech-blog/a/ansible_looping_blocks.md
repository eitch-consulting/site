---
title: Ansible - Looping blocks
date: 2024-04-24T23:12:48-03:00
draft: false
language: en
featured_image: ../assets/images/featured/featured-ansible-logo.png
summary: |
  I needed to do a loop inside a block in Ansible, but I could not do it - it was forbidden to use inside
  the block. The solution was to use `include_tasks` to mimic the block in another file.
categories: Tech-Blog
tags:
- ansible
- automation
type: tech-blog
---

I needed to do a loop inside a full block in Ansible, and I discovered that it was **not** possible to
do something like this:

```yaml
- name: this is a block
  block:
    - name: task one from the block, loop {item}
      ansible.builtin.debug:
         msg: "This is a block, indeed. Block: {item}."

    - name: task two from the block, loop {item}
      ansible.builtin.debug:
         msg: "This is a block, indeed. Block: {item}."
  loop:
    - "one"
    - "two"
```

Yep, **this won't work and will give me an error** because the `loop` clause can't be used inside a block. What I should do?

The solution that I found was to create a separate tasks file, and use a loop with `ansible.builtin.include_tasks`.
This tasks file would be equivalent to a block, but inside it we'll have just a set of tasks instead of a block.

An example which **works**:

- File `main.yml`

```yaml
---
- name: setup VMs
  ansible.builtin.include_tasks:
    file: setup_vms.yml
  loop:
    - "webserver"
    - "database"
    - "appserver"
  loop_control:
    loop_var: vm
```

- File `setup_vms.yml`

```yaml
---
- name: VM {{ vm }} - create
  ansible.builtin.command:
    cmd: vm create {{ vm }}

- name: VM {{ vm }} - configure
  ansible.builtin.command:
    cmd: vm config {{ vm }} set {{ idx }} {{ item }}
  loop:
    - "disk"
    - "memory"
    - "network"
  loop_control:
    index_var: idx
```

Now this loop will "run" three times, doing the two tasks inside `setup_vms.yml`, with the values from the loop. Also, notice that:

- In `main.yml`, I used the `loop_control` and `loop_var` to specify that in the other file, I can reference the value of the loop item as `vm`, instead of the default `item`.

- I also used a loop inside the loop, which you can see in the file `setup_vms.yml`. This was possible mainly because I associated the first loop with the variable `vm`, and the second loop will use the default `{{ item }}`.

- The `loop_control` and `index_var` in the `setup_vms.yml` was used to define a variable to the index of each loop's item. In this case, `disk` as index `0`, `memory` as `1`, and `network` as `2`. I used this index number in the command (also note that we can't use this index variable in the task's name).

As a result, 12 tasks were executed:

```plain
VM webserver - create
- vm create webserver

VM webserver - configure
- vm config webserver set 0 disk

VM webserver - configure
- vm config webserver set 1 memory

VM webserver - configure
- vm config webserver set 2 network

VM database - create
- vm create database

VM database - configure
- vm config database set 0 disk

VM database - configure
- vm config database set 1 memory

VM database - configure
- vm config database set 2 network

VM appserver - create
- vm create appserver

VM appserver - configure
- vm config appserver set 0 disk

VM appserver - configure
- vm config appserver set 1 memory

VM appserver - configure
- vm config appserver set 2 network
```

## References

- [Ansible Docs - Loops - Defining inner and outer variable names with loop_var](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_loops.html#defining-inner-and-outer-variable-names-with-loop-var)
