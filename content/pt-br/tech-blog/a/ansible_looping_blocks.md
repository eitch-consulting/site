---
title: Ansible - Looping blocks
date: 2024-04-24T23:12:48-03:00
draft: false
language: pt-BR
featured_image: ../assets/images/featured/featured-ansible-logo.png
summary: |
  Precisei fazer um loop dentro de um bloco no Ansible, e descobri que era proibido usar dentro do bloco.
  A solução foi usar o `include_tasks` como se fosse um bloco em outro arquivo.
categories: Tech-Blog
tags:
- ansible
- automation
type: tech-blog
---

Precisei fazer um loop de um bloco completo no Ansible, e descobri que **não** era possível fazer algo assim:

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

É, **isso vai dar erro**, porque o loop não pode ser usado em um bloco. Então como fazer?

A solução que encontrei foi criar um arquivo de tasks separado, e usar um loop com o `ansible.builtin.include_tasks`. Em outras palavras, esse arquivo de tasks separado seria o equivalente à um bloco, mas dentro dele só vai ter um conjunto de tasks mesmo.

Exemplo que **deu certo**:

- Arquivo `main.yml`

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

- Arquivo `setup_vms.yml`

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

Pronto, agora ele vai "rodar" três vezes as duas tarefas dentro do `setup_vms.yml`, com os valores do loop. Note que:

- No `main.yml` eu usei um `loop_control` e `loop_var` para dizer que no outro arquivo, posso referenciar o valor do item como `vm`, ao invés do padrão que é `item`.
  
-  Também usei um loop dentro de um loop, como pode-se ver no `setup_vms.yml`. Isso foi possível principalmente porque associei o primeiro loop na variável `vm`, assim o segundo loop pode ser usado normalmente com `{{ item }}`.

- Por fim, usei o `loop_control` e `index_var` pra definir uma variável pro índice de cada item do loop. `disk` como `0`, `memory` como `1`, `network` como `2`. Usei isso no comando (detalhe que não dá para usar no `name` da task)

No final, executei 12 tasks ao todo:

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

## Referências

- [Ansible Docs - Loops - Defining inner and outer variable names with loop_var](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_loops.html#defining-inner-and-outer-variable-names-with-loop-var)
