---
title: "Deep Dive with Ansible: Patching an Ansible Collection"
date: "2024-04-19T13:21:16+02:00"
draft: false

author: "Shan"
tags: ["devops", "ansible"]
categories : ["Technology"]

toc:
  enable: true
  auto: true
---
<!--more-->
## Problem

Currently, I have been developing a lot with Ansible at work for Automation logic
for servers and industrial hardware. It is also a fantastic tool to performing testing
of provisioned systems. One such test was checking whether a Grafana container was configured
correctly through a Docker Compose project. The Grafana container was behind a Traefik reverse-proxy
configured with TLS-termination for some internal self-signed certificates, where it wasn't required
to check certificate validation.

There exists a nice [`community.grafana`][1] Ansible Collection that can help do lookup of some
provisioned Grafana Dashboards.

The usage based on the Documentation was quite simple:

```yaml
- name: Gather Information about current Grafana Dashboards
  ansible.builtin.set_fact:
    existing_grafana_dashboards: |
      {{ lookup(
        'community.grafana.grafana_dashboard',
        'grafana_url=https://{{ ansible_host }}/grafana grafana_user=admin grafana_password={{ grafana_admin_pass }} search=fooDash'
      }}
 - ansible.builtin.debug:
     var=existing_grafana_dashboards
```

This plugin would simply connect to each Grafana dashboard instance in my inventory and try to search for existence of dashboards
with name `fooDash` and store them in a local Ansible Fact for the Playbook called `existing_grafana_dashboards`.

This could have worked out of the box, but as it turned out since the URL was HTTPS and my instance was hosted with a self-signed
certificate, I got URI verification failures, which is quite common when working with Ansible and URIs.

Instincts told me to just add a `validate_certs=false` parameter to the lookup plugin. So a refactor would look something like:

```yaml
- name: Gather Information about current Grafana Dashboards
  ansible.builtin.set_fact:
    existing_grafana_dashboards: |
      {{ lookup(
        'community.grafana.grafana_dashboard',
        'grafana_url=https://{{ ansible_host }}/grafana grafana_user=admin grafana_password={{ grafana_admin_pass }} search=fooDash'
        validate_certs=false
      }}
 - ansible.builtin.debug:
     var=existing_grafana_dashboards
```

However it turns out I still got the same SSL verification error and upon inspecting the Upstream code, the plugin did not have an
implementation for the `validate_certs` logic, which is quite common in Ansible's URI lookup plugin.

With a proper [Issue report][2] at the collection's repository, I tried to figure out how to patch this requirement to the lookup plugin.

## Local Setup for Ansible Collections

Based on the Repository's documentation, the easiest way would be to have the collection cloned in to the `COLLECTIONS_PATH` on a work-machine
and have the collection reflect in a project to test the changes out.

The [`COLLECTIONS_PATH`][3] is an Ansible Configuration which points to a directory where all other collections exists. If not explicitly set
it can vary from where the collections exists in the system.

If Ansible is installed via `pip install --user ansible` then chances are the path to the collections is 

```bash
~/.local/lib/python<version>/site-packages/ansible_collections
```

If Ansible is installed globally either via some distro package manager it might be somewhere in:

```
/usr/local/lib/python<version>/dist-packages/ansible_collections
```

{{< admonition tip >}}
Use `ansible-galaxy collection list` to figure out where are the collections pointed at when using ansible.

The first line shows you the path where the collections are located on the machine.
{{< /admonition >}}

Since `community.grafana` already exists on the work machine, it would be wise not to clone the repository in the same path
as where all the collections already exist.

### Overriding the Collections Path for Development

If we read the documentation for the `COLLECTIONS_PATH` again, it mentions:

> if COLLECTIONS_PATHS includes '{{ ANSIBLE_HOME ~ "/collections" }}', and you want to add my.collection to that directory,
> it must be saved as '{{ ANSIBLE_HOME} ~ "/collections/ansible_collections/my/collection" }}'

Here `ANSIBLE_HOME` would be `~/.ansible` directory. So in my case, it would be the following:

```bash
~/.ansible/collections/ansible_collections/community/grafana
```

So here are the steps to setting up the override logic for the Ansible Collection logic locally:

0. Create a fork of the upstream repository on GitHub
1. Create the base collections directory if it doesn't exist:
    ```bash
    mkdir -p ~/.ansible/collections/ansible_collections
    ```
2. Clone to fork in such a way that the contents of the repo are placed under `ansible_collections/community/grafana`
    ```bash
    git clone <FORK_URL> ~/.ansible/collections/ansible_collections/community/grafana
    ```
    this could be generally described for other ansible collections as follows:
   ```bash
   git clone <URL> ~/.ansible/collections/ansible_collections/<namespace>/<collection>
   ```

Now you can make changes to the codebase in this particular repository and test it somewhere else locally to see if the changes
work as expected.

### Setting Up a Local Test Base

In order to test the changes create a simple directory where your development codebases exist e.g. `~/Development/ansible/grafana_patch`

In the particular directory add the following Ansible configuration file called `ansible.cfg`:

```ini
[defaults]
collections_path = ~/.ansible/collections/ansible_collections
```

This is a local override to tell ansible, that add the collections that now exist in our `~/.ansible/collections/ansible_collections`
for the local project too.

This is verified by performing:

```bash
$ pwd
/home/User/Development/ansible/grafana_patch
$ ansible-galaxy collection list
# /home/User/.ansible/collections/ansible_collections
Collection        Version
----------------- -------
community.grafana 1.8.0
# /usr/local/lib/python3.8/dist-packages/ansible_collections
Collection                    Version
----------------------------- -------
### Other collections omitted for brewity
```

Now a simple playbook called `test_dashboard_lookup.yml`

```yaml
- hosts: localhost
  tasks:
    - ansible.builtin.set_fact:
        local_dashboard: |
          {{ lookup(
            'community.grafana.grafana_dashboard',
            'grafana_url=https://localhost:3000 grafana_user=admin grafana_password=admin search=foo',
            validate_certs=false
          }}
    - ansible.builtin.debug:
        var=local_dashboards
```
can be used to verify if things are working locally with a default Grafana Container setup with self-signed certs using:

```bash
ansible-playbook test_dashboard_lookup.yml -vvvv
```

## Result

Based on the [Issue][2] filed by me, there now exists [a patch][4] that provides the necessary feature for validation / not validating
SSL certificates.

New Tool, New Environment, New way to Patch something => Better to document!


[1]: https://github.com/ansible-collections/community.grafana
[2]: https://github.com/ansible-collections/community.grafana/issues/346
[3]: https://docs.ansible.com/ansible/latest/reference_appendices/config.html#collections-paths
[4]: https://github.com/ansible-collections/community.grafana/pull/356
