# pgvillage.chainsmith API

This document describes all variables of the `pgvillage.chainsmith` role, as defined in
[defaults/main.yml](../defaults/main.yml).

## How the role works

1. On `localhost` the `chainsmith` package is installed and a temporary output directory is created.
2. On every target node a list of certificate files (`chainsmith_cert_files`) is written to
   `chainsmith_cert_list`, and each file is checked for existence and for expiry within
   `chainsmith_expiry_days`.
3. If any node reports a missing or expiring certificate, `chainsmith issue` is run on `localhost`
   with the configuration in `chainsmith_body`, and the complete chain (server and client
   certificates and keys) is redeployed to all nodes.

The play must therefore include `localhost` as well as all hosts in `chainsmith_nodes`.

## Installation

| Variable | Default | Description |
| --- | --- | --- |
| `chainsmith_package_state` | `present` | State of the chainsmith package(s) (`present`, `latest`, `absent`). Installation only runs on localhost. |
| `chainsmith_package_names` | `[chainsmith]` | List of packages to install to provide the chainsmith binary. |
| `chainsmith_cmd` | `/usr/local/bin/chainsmith` | Path to the chainsmith executable on localhost. |

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
| `chainsmith_tmpdir` | `{{ chainsmith_tmpfile.path }}` | Directory on localhost where chainsmith writes its output. Defaults to a temporary directory created by the role. |
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
| `chainsmith_nodes` | `{{ groups['hacluster'] + groups['backup'] }}` | Inventory hosts that receive certificates and are checked for expiry. |
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
  backup: [minio]
```

## Facts set by the role

After generation, the following facts are set on every node (except localhost) and are
referenced by the deployment variables above:

| Fact | Description |
| --- | --- |
| `chainsmith_results_server_cert` | Server certificate for the node's FQDN. |
| `chainsmith_results_server_key` | Server private key for the node's FQDN. |
| `chainsmith_results_server_chain` | Server chain (deployed to clients as root cert). |
| `chainsmith_results_client_certs` | Dict of client name -> client certificate. |
| `chainsmith_results_client_chain` | Client chain (deployed to servers as root cert). |
| `chainsmith_results_client_keys` | Dict of client name -> client private key. |
