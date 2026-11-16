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

Define `chainsmith_controller` with exactly one certificate-generating host and
`chainsmith_nodes` with the certificate recipients. A host can belong to both
groups.

Inventory example:

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

Use the same inventory hostname for a machine in every group. The controller
can also be reached over SSH; its inventory name need not be `localhost`.
Existing service groups can be children of `chainsmith_nodes`, as shown above.

Run the role on both groups together:

```yaml
- name: Generate and deploy certificates
  hosts: chainsmith_controller:chainsmith_nodes
  become: true
  gather_facts: true
  roles:
    - pgvillage.chainsmith
```

The recipient users must already exist. Configure `chainsmith_users` and the
certificate paths and owners for the services on each recipient. All controller
and recipient hosts must be included in the same play batch, also when using
`--limit` or `serial`. Use the default `linear` execution strategy so certificate
checks finish on all recipients before generation and deployment.

Existing inventories must add these explicit groups: the role no longer selects
the controller from `localhosts` or recipients from `hacluster` and `backup`.

License
-------

PostgreSQL

Author Information
------------------

PgVillage is an Open Community.
Main contributor is Nibble-IT.
