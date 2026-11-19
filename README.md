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

By default, the first host in `localhosts` generates certificates and the union
of `hacluster` and `backup` receives them. Existing inventories using these groups
can keep their host selection without adding new groups.

Optionally define `chainsmith_controller` with exactly one generator and/or
`chainsmith_nodes` with the recipients to override those defaults independently.
A host can belong to both groups. No parent group is required.

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

With explicit groups, run the role on both together:

```yaml
- name: Generate and deploy certificates
  hosts: chainsmith_controller:chainsmith_nodes
  become: true
  gather_facts: true
  roles:
    - pgvillage.chainsmith
```

The recipient users must already exist. Configure `chainsmith_users` and the
certificate paths and owners for the services on each recipient. To perform
certificate work, the selected controller and every recipient must be in the
same play batch. Use the default `linear` strategy so checks finish on all
recipients before generation and deployment.

If `--limit`, `serial` or the play's host selection leaves required hosts outside
the current batch, Chainsmith prints a warning-style message listing those hosts
and skips its package installation, user discovery, certificate checks,
generation, deployment and cleanup. Other tasks and roles continue normally.
This also applies when the optional inventory groups are absent and the legacy
defaults are used. Invalid controller or recipient configuration still fails
validation.

Serial maintenance runs therefore skip certificate work in every incomplete
batch. Run certificate management separately with the controller and all
recipients together; a limited run containing that complete set works normally.
Existing certificates are untouched by an incomplete run.


Testing
-------

GitHub CI runs `molecule test --all` for both scenarios: `default` covers the
legacy inventory defaults with a separate controller, and `controller_node`
covers an explicitly selected controller that also receives certificates.
The configured sequences check convergence, idempotence, certificate chains,
matching private keys, file ownership and permissions, and temporary-file cleanup.

Limited and serial runs, continuation after a skipped role, and renewal in a
later complete run were checked during development.

License
-------

PostgreSQL

Author Information
------------------

PgVillage is an Open Community.
Main contributor is Nibble-IT.
