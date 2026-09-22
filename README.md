Zabbix Template Deploy
========================

Imports the 6 current requirement-cluster Zabbix templates into the Zabbix
server's template library, over its HTTP API.

This role only pushes template *definitions* onto the server — it does not
attach any host to a template. Attaching a specific host to the right
template(s) is `zabbix_host_link`'s job (a separate role, since "does this
template exist on the server" and "is this host linked to it" are two
different API operations against two different Zabbix objects). It also
never touches the monitored RHEL hosts themselves — that's
`zabbix_agent_deploy`, over a completely different connection (ssh, not
httpapi). See that role's README for the full purpose/approach split
between the two.

Purpose & Approach
-------------------

Each of the 6 requirement clusters (`host_health`, `storage`,
`virtualisation`, `containers`, `host_access`, `privileged_activity`) has
one master template, authored as a Zabbix 8.0 export YAML and checked in
under `files/`. This role loops `community.zabbix.zabbix_template` over
all 6, importing (or re-importing, on a later change) each one with
`state: present` — the same create-or-update semantics as using Zabbix's
own "Import" button in the UI, just automated and repeatable.

`files/` intentionally holds only the **current** version of each
template. Two clusters (`host_access`, `privileged_activity`) have an
earlier `_rsyslog`-suffixed draft in the source documentation project
(`~/zabbix/`) that was superseded by an audit-log-based approach — those
drafts are deliberately left out here.

Requirements
------------

- The `community.zabbix` collection installed on the control node. Not
  bundled with this role — tracked at the playbook/project level (see
  `meta/main.yml`'s `dependencies:` comment for why it can't live here).
- An `httpapi` connection to the target Zabbix server: `ansible_connection:
  httpapi`, `ansible_network_os: community.zabbix.zabbix`, plus the
  server's API port and URL path.
- Valid Zabbix API credentials reachable as `ansible_user` /
  `ansible_httpapi_pass` on whatever host/group this role runs against.

Role Variables
---------------

Defined in `defaults/main.yml`:

| Variable               | Default                                                                                                          | Purpose                                                          |
| ------------------------ | -------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `zabbix_template_files` | `[scope_host_health, scope_storage_filesystem, scope_virtualisation, scope_containers, scope_host_access, scope_privileged_activity]` | Which files in `files/` (without the `.yaml` extension) get imported, and in what order. |

Dependencies
------------

None (no other Ansible roles). The `community.zabbix` collection is a
hard requirement but isn't a role dependency — see Requirements above.

Example Playbook
------------------

    - hosts: zabbix_api_endpoint
      gather_facts: false
      roles:
        - zabbix_template_deploy

With the target defined in inventory, e.g.:

    zabbix_api:
      hosts:
        zabbix_api_endpoint:
          ansible_host: 192.168.10.160
          ansible_connection: httpapi
          ansible_network_os: community.zabbix.zabbix
          ansible_httpapi_port: 8081
          ansible_zabbix_url_path: "zabbix"

Known Limitations
-------------------

- This role only ever adds or updates templates — it never removes one.
  If a `scope_*.yaml` file is deleted from `files/` (or dropped from
  `zabbix_template_files`), the corresponding template stays on the
  Zabbix server until removed manually.
- Import order relative to `zabbix_host_link` matters: a template must
  exist on the server before any host can be linked to it, so this role
  needs to run (successfully) before `zabbix_host_link` in any playbook
  that uses both.

License
-------

MIT-0

Author Information
--------------------

Part of the `zabbix-roles` project — see `zabbix_agent_deploy/README.md`
for the project-wide purpose and the source documentation this role's
templates are built from.
