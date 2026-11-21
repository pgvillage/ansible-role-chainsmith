Chainsmith
=========

Chainsmith is a tool for generating tls chains with server and client certificates.
This role installs Chainsmith and runs it to generate and distribute a chain ready to be used by PostgreSQL for encryption in transit and authentication.
This role is part of PgVillage, which is an opinated PostgreSQL deployment for Virtual Machines.

Requirements
------------

This role aims at using an RPM from the MannemSolutions repo.

Role Variables
--------------

Please see the [API documentation](docs/api.md) for a description of all variables.
The defaults can be found in [defaults/main.yml](defaults/main.yml).


Dependencies
------------

No dependencies


Example Playbook
----------------

By default the first host in `localhosts` generates the certificates and the
hosts in `hacluster` and `backup` receive them. To choose other hosts, define the
`chainsmith_controller` group (exactly one host) and/or the `chainsmith_nodes`
group; a host can be in both:

```yaml
all:
  children:
    chainsmith_controller:
      hosts:
        node-01:
          ansible_connection: local
          ansible_python_interpreter: "{{ ansible_playbook_python }}"
    chainsmith_nodes:
      children:
        hacluster:
    hacluster:
      hosts:
        node-02:
        node-03:
        node-04:
```

Run the role on the controller and all recipients together. The recipient users
(`chainsmith_users`) must already exist.

```yaml
- name: Generate and deploy certificates
  hosts: chainsmith_controller:chainsmith_nodes
  become: true
  gather_facts: true
  roles:
    - pgvillage.chainsmith
```

With `--limit` or `serial` the role skips the certificates with a warning, see
[`--limit` and `serial`](docs/api.md#--limit-and-serial).

License
-------

PostgreSQL

Author Information
------------------

PgVillage is an Open Community.
Main contributor is Nibble-IT.
