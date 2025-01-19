---
title: on Ansible Inventories
date: 2025-01-19T19:43:57+01:00
draft: false

author: "Shan"
categories: ["Technology"]
tags: ["devops", "ansible"]

toc:
enable: true
auto: true
---
<!--more-->
# Structuring Ansible Inventories

As a part of working on [Komponist][1], working with Ansible Inventories
is essential to deploy the logic on multiple devices mentioned within these
inventories.

Most common way of creating an Ansible Inventory is using a `hosts` file
in __INI__ format. However, one can also create the inventories in __JSON__,
__YAML__. For the project I work with YAML just for consistency.

Assuming standard practice, one might be tempted to generated an inventory
in one single file either called `hosts` or `hosts.yml` file under a dedicated
directory. A TIL (Today I Learned) moment I had a while ago was using distinct
directories for hosts and groups in order to add maintain information without
messing with the main `hosts.yml` file.

## Example Inventory

We can create a specfic inventory that doesn't have to exist in `/etc/ansible` directory.

```bash
mkdir -p ~/komponist/inventory && touch ~/komponist/inventory/hosts.yml
```
Lets have 2 hosts and 1 group in our inventory:

```yml
---
testgroup1:
  hosts:
    host1:
    host2:
```

that should do it for the `hosts.yml` file.

## Host Variables

In order to specify the Host Variables and Ansible specific variables for the hosts,
we will create a base directory called `host_vars` in the `inventory` directory.

```bash
mkdir -p ~/komponist/inventory/host_vars
```
for `host1`, `host2` information we create specific directories under `host_vars` directory.

```bash
mkdir -p ~/komponist/inventory/host_vars/host{1,2}
```
now lets add `vars.yml` and `vault.yml` to each of the hosts directory.

`vars.yml` will contain host-specifc variables and `vault.yml` any sensitive values like
passwords, passkeys etc.

```bash
touch ~/komponist/inventory/host_vars/host{1,2}/{vars,vault}.yml
```

The final structure will be:

```bash
host_vars
├── host1
│   ├── vars.yml
│   └── vault.yml
└── host2
    ├── vars.yml
    └── vault.yml
```

### Adding Ansible Host Variables

assume that `host1` has `192.168.10.10` with user `h1user`, and `host2` has `192.168.10.20`
with user `h2user`.

The following content should exist in `host1/vars.yml` file:

```yml
---
ansible_host: 192.168.10.10
ansible_user: h1user
```

The following content should exist in `host2/vars.yml` file:

```yml
---
ansible_host: 192.168.10.20
ansible_user: h2user
```

Lets check how the current inventory might look like using:

```bash
ansible-inventory -i ~/komponist/inventory --list --yaml
all:
  children:
    testgroup1:
      hosts:
        host1:
          ansible_host: 192.168.10.10
          ansible_user: h1user
        host2:
          ansible_host: 192.168.10.20
          ansible_user: h2user
```

if you don't specify `-i`, `ansible-inventory` will always try resolving
the inventory in `/etc/ansible/` directory.

Similar to `vars.yml` file, any sensitive information for hosts can be placed
in `vault.yml` and it will be resolved with `ansible-inventory`.


## Group Variables

If you wish that both the hosts obtain variables, we can leverage on Group Variables,
similarly to Host Variables.

Similar to `host_vars`, create `group_vars` directory and the respective group name's
directory under `group_vars`.

```bash
mkdir -p group_vars/testgroup1/ && touch group_vars/testgroup1/{vars,vault}.yml
```
### Group Variables

As an example, lets add a variable called `priority` with an int value

The content of `testgroup1/vars.yml`:

```yml
---
priority: 1
```
Now checking it once again `ansible-inventory`:

```bash
ansible-inventory -i ~/komponist/inventory --list --yaml
all:
  children:
    testgroup1:
      hosts:
        host1:
          ansible_host: 192.168.10.10
          ansible_user: h1user
          priority: 1
        host2:
          ansible_host: 192.168.10.20
          ansible_user: h2user
          priority: 1
```

group variables trickle to each host in the group i.e., `priority: 1`


## Variable Precedence

It is worth noting that same variables may be overwritten if once doesn't
take care of variable precedence.

The predence is easy to remember:

{{< admonition type=info title="variable precedence rule" open=true >}}
Individual (host) precedes a Group. Individualism over Groupism.
{{< /admonition >}}

### Example

Let's play with the `priority` variable. Lets set the value to _10_ for `host1`
in the `host_vars/host1/vars.yml`.

The content of `host_vars/host1/vars.yml`:

```yml
---
ansible_host: 192.168.10.10
ansible_user: h1user
priority: 10
```

Lets check the value:

```bash
ansible-inventory -i . --list --yaml
all:
  children:
    testgroup1:
      hosts:
        host1:
          ansible_host: 192.168.10.10
          ansible_user: h1user
          priority: 10
        host2:
          ansible_host: 192.168.10.20
          ansible_user: h2user
          priority: 1
```

as mentioned, the `priority` variable value is overriden by the host's variable value
than that of the group variable.

If we wanted to set value to _all available groups_ we can add such a variable
to `group_vars/all/vars.yml` which will add it all hosts in all groups.

But remember, the `all` group will have the __LEAST__ precedence since it is the 
parent of all other groups. [Info][2]


[1]: https://github.com/shantanoo-desai/komponist
[2]: https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html#how-variables-are-merged
