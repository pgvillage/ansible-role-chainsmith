# pgvillage.chainsmith API

This document describes all variables of the `pgvillage.chainsmith` role, as defined in
[defaults/main.yml](../defaults/main.yml).

## How the role works

1. On the selected controller the `chainsmith` package is installed and a temporary output directory is created.
2. On every target node a list of certificate files (`chainsmith_cert_files`) is written to
   `chainsmith_cert_list`, and each file is checked for existence and for expiry within
   `chainsmith_expiry_days`.
3. If any node reports a missing or expiring certificate, `chainsmith issue` is run on the controller
   with the configuration in `chainsmith_body`, and the complete chain (server and client
   certificates and keys) is redeployed to all nodes.

The `chainsmith_controller` and `chainsmith_nodes` inventory groups are optional.
Without `chainsmith_controller`, the first host in `localhosts` generates the
chain. Without `chainsmith_nodes`, the deduplicated union of `hacluster` and
`backup` receives certificates. Each group independently overrides its default.
When present, the controller group must contain exactly one host and the
recipient group must not be empty. The resulting selection must always have
one controller and at least one recipient.

The controller can also be a recipient. Neither responsibility depends on its
inventory hostname or on whether Ansible connects locally or over SSH. No parent
inventory group is needed.

To perform certificate work, run the role on the selected controller and
recipients together, with facts available for recipients and the default
`linear` strategy. If `--limit`, `serial` or play host selection excludes any
required host from the current batch, the role prints a warning-style message
once for that batch, identifying the controller and missing hosts. It skips all
package installation, user discovery, certificate checks, generation, deployment
and cleanup. Other tasks and roles continue normally; existing certificate files
are untouched. Invalid inventory or effective host selection still fails
validation before this decision.

Each incomplete serial batch skips certificate work, so perform certificate
management in a separate complete batch. A limit that includes the controller
and every recipient runs normally. This behavior applies to both explicit
groups and legacy defaults.
See the [inventory and playbook example](../README.md#example-playbook).

## Inventory selection

| Variable | Default | Description |
| --- | --- | --- |
| `chainsmith_controller_host` | First host in `chainsmith_controller`, otherwise first in `localhosts` | The single inventory host that installs and runs Chainsmith. May also receive certificates. |
| `chainsmith_nodes` | Hosts in `chainsmith_nodes`, otherwise `hacluster + backup` (deduplicated) | Inventory hosts that receive certificates and are checked for expiry. |

The controller creates the temporary configuration, generates the complete chain
when needed, and removes its temporary configuration and output directory in an
`always` block. Generation runs without privilege escalation. Recipients read
OS user information, check their certificate files, and deploy server and client
certificates, trust chains and private keys with the configured ownership. Client
certificate users are selected through `chainsmith_users` and must already exist.
The role does not reload services after deployment.

## Installation

| Variable | Default | Description |
| --- | --- | --- |
| `chainsmith_package_state` | `present` | State of the chainsmith package(s) (`present`, `latest`, `absent`). Installation only runs on the controller. |
| `chainsmith_package_names` | `[chainsmith]` | List of packages to install to provide the chainsmith binary. |
| `chainsmith_cmd` | `/usr/bin/chainsmith` | Path to the chainsmith executable on the controller. |

## Expiry check

| Variable | Default | Description |
| --- | --- | --- |
| `chainsmith_cert_list` | `/tmp/chainsmith_cert_list.txt` | Path on each node where the list of deployed certificate files is written. Used for the expiry check. |
| `chainsmith_expiry_days` | `30` | Certificates expiring within this number of days are considered invalid, which triggers regeneration and redeployment of the full chain. |
| `chainsmith_cert_files` | *(derived)* | List of dicts (`dest`) of certificate files checked on a node: the client cert and root cert of every user in `chainsmith_users` for the node's groups, plus `server.crt` and `root.crt` in `chainsmith_server_cert_path`. |

## Chain configuration

| Variable | Default | Description |
| --- | --- | --- |
| `chainsmith_subject` | PgVillage / Mannem Solutions subject | Subject for the root CA and intermediates. Keys are X.509 subject attributes (`C`, `ST`, `L`, `O`, `OU`, `CN`). |
| `chainsmith_tmpdir` | `{{ chainsmith_tmpfile.path }}` | Directory on the controller where chainsmith writes its output. Defaults to a temporary directory created by the role. |
| `chainsmith_server_intermediate` | *see below* | Intermediate CA that issues the server certificates. |
| `chainsmith_client_intermediate` | *see below* | Intermediate CA that issues the client certificates. |
| `chainsmith_body` | *see below* | Full chainsmith configuration, written as JSON and passed to `chainsmith issue`. |

Default `chainsmith_subject`:

```yaml
chainsmith_subject:
  C: NL/postalCode=2403 VP
  ST: Zuid Holland
  L: Alphen aan den Rijn/street=Weegbreestraat 7
  O: Mannem Solutions
  OU: PgVillage
  CN: postgres
```

Default `chainsmith_server_intermediate`:

```yaml
chainsmith_server_intermediate:
  name: server
  keyUsages: [keyEncipherment, dataEncipherment, digitalSignature]
  extendedKeyUsages: [serverAuth]
  servers: "{{ chainsmith_servers }}"
```

Default `chainsmith_client_intermediate`:

```yaml
chainsmith_client_intermediate:
  name: client
  clients: "{{ chainsmith_internal_clients + chainsmith_external_clients }}"
  keyUsages: [keyEncipherment, dataEncipherment, digitalSignature]
  extendedKeyUsages: [clientAuth]
```

Default `chainsmith_body`:

```yaml
chainsmith_body:
  subject: "{{ chainsmith_subject }}"
  tmpdir: "{{ chainsmith_tmpdir }}"
  intermediates:
    - "{{ chainsmith_server_intermediate }}"
    - "{{ chainsmith_client_intermediate }}"
```

## Servers

| Variable | Default | Description |
| --- | --- | --- |
| `chainsmith_hosts` | *(derived)* | List of FQDNs of all `chainsmith_nodes` (from facts). |
| `chainsmith_ips` | *(derived)* | List of the first IPv4 address of all `chainsmith_nodes` (from facts). |
| `chainsmith_servers` | *(derived)* | Dict of server certificates to generate: FQDN -> list of SAN entries. By default every server cert is valid for all hosts and IPs (`chainsmith_hosts + chainsmith_ips`), which is a requirement for stolon_proxy. |

For server certificates which correspond to one host only, override `chainsmith_servers`:

```yaml
chainsmith_servers: |
  {% set nodes = {} %}
  {% for node in chainsmith_nodes %}
  {% do nodes.update({hostvars[node].ansible_facts.fqdn: hostvars[node].ansible_facts.all_ipv4_addresses }) %}
  {% endfor %}
  {{ nodes }}
```

## Server certificate deployment

| Variable | Default | Description |
| --- | --- | --- |
| `chainsmith_server_cert_path` | `/etc/pki/{{ chainsmith_server_cert_owner }}` | Folder where `server.crt`, `server.key` and `root.crt` are deployed. |
| `chainsmith_server_cert_owner` | `postgres` | OS user owning the server certificate folder and files. |
| `chainsmith_server_folders` | `[{dest: chainsmith_server_cert_path, owner: chainsmith_server_cert_owner}]` | Folders to create for the server certificate files (mode `0700`). |
| `chainsmith_server_cert_files` | `server.crt` and `root.crt` | List of dicts (`dest`, `owner`, `content`) of server certificate files to deploy (mode `0600`). `root.crt` contains the client chain, used by the server to verify client certificates. |
| `chainsmith_server_key_files` | `server.key` | List of dicts (`dest`, `owner`, `content`) of server private key files to deploy (mode `0600`). |

## Clients

| Variable | Default | Description |
| --- | --- | --- |
| `chainsmith_users` | *see below* | Dict of inventory group -> list of OS users that get a client certificate. Certs are generated for all users, but only deployed on nodes in the corresponding group, into the user's home directory. |
| `chainsmith_internal_clients` | *(derived)* | Flattened list of all users in `chainsmith_users`, used as client certificate names. |
| `chainsmith_external_clients` | `[applicatie]` | Additional client certificate names to generate which are not deployed by this role (e.g. for applications). They are available in the chainsmith output. |
| `chainsmith_client_cert_sub_folder` | `.postgresql` | Folder (relative to the user's home directory) where client certificates are deployed. |
| `chainsmith_client_cert_filename` | `postgresql.crt` | Filename of the client certificate (libpq default). |
| `chainsmith_client_pkey_filename` | `postgresql.key` | Filename of the client private key (libpq default). |
| `chainsmith_client_root_filename` | `root.crt` | Filename of the root certificate (server chain) used by clients to verify the server. |

Default `chainsmith_users`:

```yaml
chainsmith_users:
  hacluster: [postgres, avchecker, pgquartz, pgfga]
  router: [pgroute66]
  backup: [rustfs]
```

Until v0.2.4 the default backup user was `minio`. Set `chainsmith_users` in your inventory to keep
deploying client certificates for `minio`.

## Facts set by the role

After generation, the following facts are set on every certificate recipient,
including the controller if it is a recipient, and are referenced by the
deployment variables above:

| Fact | Description |
| --- | --- |
| `chainsmith_results_server_cert` | Server certificate for the node's FQDN. |
| `chainsmith_results_server_key` | Server private key for the node's FQDN. |
| `chainsmith_results_server_chain` | Server chain (deployed to clients as root cert). |
| `chainsmith_results_client_certs` | Dict of client name -> client certificate. |
| `chainsmith_results_client_chain` | Client chain (deployed to servers as root cert). |
| `chainsmith_results_client_keys` | Dict of client name -> client private key. |
