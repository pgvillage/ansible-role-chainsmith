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

Please see [defaults](https://github.com/pgvillage/ansible-role-chainsmith/blob/main/defaults/main.yml) for all variables


Dependencies
------------

No dependencies


Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - hosts: servers
      roles:
         - pgvillage.chainsmith

License
-------

PostgreSQL

Author Information
------------------

PgVillage is an Open Community.
Main contributor is Nibble-IT.
