# zabbix_template_deploy — every task, block by block

Plain-language walkthrough of the whole role: what each file and line does,
why it is there, and whether it is really needed. Every "tested" or
"verified" note below was observed on the lab Zabbix server (API version
8.0.0) or read from the `community.zabbix` collection's source (4.2.0), not
assumed. Where something was not tested, it says so. Paths are relative to
this role's folder, and line numbers refer to the files as they are now.

Contents

1. What the role is, in one paragraph
2. How the pieces run
3. `vars/main.yml`
4. `tasks/main.yml`
5. The other files (`defaults`, `handlers`, `meta`)
6. What is inside a template file (`files/`)
7. What the import does on the server
8. Variables
9. The security model
10. Errors and messages
11. Known limitations
12. Why this role is necessary, and what it gives you

---

## 1. What the role is, in one paragraph

It runs **against the Zabbix server's web API** (over `httpapi`, no SSH) and
puts the **definition of each monitoring template** onto the server. A
template is a recipe: which items to collect, how often, which problems to
raise. This role only makes the recipes *exist* on the server. It does not say
which host uses which recipe (that is `zabbix_host_link`, which runs right
after) and it does not touch any monitored host.

## 2. How the pieces run

```
playbooks/deploy_zabbix.yml   Phase 1 (this role)   ->   Phase 2 (zabbix_host_link)

vars/main.yml     the list of 6 template names to import
files/*.yaml      the 6 template definitions (Zabbix export format)
tasks/main.yml    a loop: read each file, send it to the Zabbix API
```

The order matters. Phase 2 links a host to a template **by name**, so the
template has to be on the server first. That is why this role is Phase 1.

---

## 3. `vars/main.yml` — which templates to import

```yaml
zabbix_template_files:
  - scope_host_health
  - scope_storage_filesystem
  - scope_virtualisation
  - scope_containers
  - scope_host_access
  - scope_privileged_activity
```

- **Lines 4-7 (comment):** explains the list. The older `_rsyslog` variants of
  Host Access and Privileged Activity were drafts and are left out.
- **Lines 8-14:** six names. Each is a file in `files/` **without** the `.yaml`
  ending (`scope_host_health` means `files/scope_host_health.yaml`).
- **Why it is in `vars/`, not `defaults/`:** the list is a fixed fact about the
  role. It must match the files that ship inside it, and nobody should change
  it from the inventory. `vars/` outranks inventory values, so a stray
  `group_vars` entry cannot silently replace it. (`defaults/main.yml` is empty
  on purpose.)
- **Relevant?** Yes. It is the only place that decides what gets imported.
  Adding a seventh template means adding the file **and** its name here, and a
  matching entry in `zabbix_host_link`'s `vars/main.yml`. Otherwise nothing will
  ever link to it.

---

## 4. `tasks/main.yml` — the import loop

```yaml
- name: Import each requirement-cluster template into the Zabbix server
  community.zabbix.zabbix_template:
    template_yaml: "{{ lookup('file', item + '.yaml') }}"
    state: present
  loop: "{{ zabbix_template_files }}"
  loop_control:
    label: "{{ item }}"
```

| Line | What it does |
| --- | --- |
| 3 | The task's name, shown in the run output |
| 4 | `community.zabbix.zabbix_template`: the module that talks to the Zabbix API to create, update or import a template. It reaches the server through the `httpapi` connection set in the inventory, so there is no SSH and no shell. |
| 5 | `template_yaml`: the template's text, read from a file. `item` is the current name from the loop (for example `scope_host_health`). `item + '.yaml'` adds the ending. `lookup('file', ...)` reads that file **on the control node** (Ansible searches this role's `files/` folder for a bare name). The text is sent to Zabbix as data in an API call. Nothing is copied to the server's disk. |
| 6 | `state: present`: "make sure this template exists and matches this definition". If it is missing (deleted in the GUI, or a fresh server) it is created. If it exists it is updated to match the file. |
| 7 | `loop`: run the task once per name in the list, so six times. |
| 8-9 | `loop_control: label`: cosmetic. The output shows `scope_host_health` per line, not the whole file text (hundreds of lines per template). |

- **Why needed:** a host cannot be linked to a template that is not on the
  server. This is Phase 1 of `deploy_zabbix.yml` because Phase 2 depends on it.
- **Relevant?** Yes. It is the whole role.
- **Recovery:** because of `state: present`, deleting templates in the GUI and
  re-running rebuilds them.

---

## 5. The other files

- **`defaults/main.yml`:** empty. There is nothing here that should be
  overridden.
- **`handlers/main.yml`:** empty. No task in this role needs a restart or a
  follow-up action.
- **`meta/main.yml`:** describes the role (author, license, minimum Ansible
  2.14) and states that it needs the `community.zabbix` collection. A role's
  `dependencies:` field cannot list collections, so the requirement is tracked
  in the project's `requirements.yml`.

---

## 6. What is inside a template file (`files/scope_*.yaml`)

These are Zabbix "export" files. You do not edit them by running the role.
The top of `scope_virtualisation.yaml` shows the shape:

```yaml
zabbix_export:
  version: '8.0'
  template_groups:
    - uuid: b8a9b8c2d1e443a2b5c6d7e8f9000001
      name: Hypervisors
  templates:
    - uuid: b8a9b8c2d1e443a2b5c6d7e8f9000002
      template: Scope Virtualisation KVM
      name: 'Scope: Virtualisation (KVM)'
      description: '...'
      groups:
        - name: Hypervisors
      discovery_rules:
        - ... item_prototypes: ... trigger_prototypes: ...
```

| Field | Meaning |
| --- | --- |
| `version: '8.0'` | The Zabbix export format the file is written for |
| `template_groups` | The folder(s) the template is filed under in the Zabbix UI (`Linux servers`, `Hypervisors` or `Scope Templates`). This has nothing to do with the host group a host sits in. |
| `uuid` | Zabbix's permanent identity for that object. Importing the same UUID **updates** the template instead of creating a duplicate. |
| `template:` | The **technical name**. `zabbix_host_link` links hosts by exactly this string. Trigger expressions also contain it, for example `find(/Scope Virtualisation KVM/systemd.unit.info[...])`. Renaming a template means changing every expression too. |
| `name:` | The display name in the GUI. Not used for linking. |
| `discovery_rules` (LLD) | Rules that find things on the host (VMs, filesystems, containers) and create items for each one automatically. This is how the templates adapt to what is actually present on a host. |
| `item_prototypes`, `trigger_prototypes` | The items and alerts created once for each discovered thing |
| `items`, `triggers` | Fixed items and alerts that exist once per host |

**The six templates** (technical names, verified against the files):

| File | `template:` value |
| --- | --- |
| `scope_host_health.yaml` | Scope Host Health and Availability |
| `scope_storage_filesystem.yaml` | Scope Storage and Filesystem |
| `scope_virtualisation.yaml` | Scope Virtualisation KVM |
| `scope_containers.yaml` | Scope Containers |
| `scope_host_access.yaml` | Scope Host Access |
| `scope_privileged_activity.yaml` | Scope Privileged Activity |

These must match `zabbix_cluster_template_names` in
`zabbix_host_link/vars/main.yml` character for character.

**How the templates connect to the agent role.** Counted from the files, the
six templates hold about 53 active definitions (items, discovery rules and
prototypes), 17 dependent items and 2 calculated items. Most collection is
active. That has three consequences:
- **Active items:** the agent starts these itself and sends the data to the
  server. They need `ServerActive=` and a `Hostname=` that equals the Zabbix
  host name. Both are written by `zabbix_agent_deploy`.
- **Custom keys:** keys such as `kvm.vm.discovery`, `lvm.vg.free[...]` and
  `podman.status` do not exist in Zabbix itself. They are the UserParameters
  the agent role installs. Without them those items show "Not supported".
- **Log items:** `log[/path,...]` items open the file as the `zabbix` user, so
  the agent role grants that user read access first.

---

## 7. What the import does on the server

The module (`zabbix_template.py`) runs two API calls per template:
1. **Compare** (`configuration.importcompare`): "what would change if I imported
   this file?" If the answer is empty the module reports `ok`
   ("Template is up-to date"). If not, it goes on.
2. **Import** (`configuration.import`): the real change. The module reports
   `changed` ("Template import successful").

**The rules it sends (from the module source):**

| What | Rule |
| --- | --- |
| Items, triggers, discovery rules, graphs, dashboards | create missing, update existing, **delete missing** |
| The template itself | create missing, update existing |
| Template and host groups | create missing |
| Template linkage | create missing, and **delete missing** is forced on (line 400 of the module) |

**What "delete missing" means for you.** The file is the source of truth.
Anything added to these six templates in the Zabbix GUI (an item, a trigger)
and not present in the file is deleted on the next run. Make changes in the
`.yaml` file, then run the role.

**Why the role reports `changed` on every run (tested).** I ran the read-only
comparison (`configuration.importcompare`) with the real files against the live
server. All six templates came back non-empty, and the differences are cosmetic:

| Difference | Example from the comparison |
| --- | --- |
| The file writes a default value out explicitly, Zabbix stores it without | `delay: 1m` and `value_type: UNSIGNED` on many items |
| Same number, written differently | a multiplier `1.0E-9` in the file, `1e-09` in Zabbix |
| Triggers rewritten in another form | several triggers reported as removed and re-added |

There were **no entries about hosts or template links** in any comparison. So
`changed` is caused by how the files are written, not by anything wrong on the
server. A run is harmless, but it never reports a clean `ok`. To make `changed`
mean "something really changed", export the six templates once from Zabbix and
keep those exports as the files. That is optional and not done.

**About "the import unlinks every host".** An earlier FAQ entry (Q7) says the
import unlinks every host from every template until `zabbix_host_link` runs.
The comparison above shows no host or link changes, and I did not run the
real import on its own to test it, so treat that claim as **unproven**. The
safe habit is unchanged: run this role and `zabbix_host_link` together, in that
order, through `deploy_zabbix.yml`.

---

## 8. Variables

| Variable | Where it is set | Used by |
| --- | --- | --- |
| `zabbix_template_files` | `vars/main.yml` (fixed) | the loop in `tasks/main.yml` |
| `ansible_connection`, `ansible_network_os`, `ansible_httpapi_port`, `ansible_zabbix_url_path`, `ansible_httpapi_use_ssl`, `ansible_httpapi_validate_certs` | `zabbix-deploy/inventory/host_vars/zabbix_api_endpoint.yml` | how the play reaches the API |
| `ansible_user`, `ansible_httpapi_pass` | the same file | the Zabbix API login |

The play sets `gather_facts: false`, because there is no operating-system shell
behind an API to gather facts from.

---

## 9. The security model

**What the role can do.** With its API login it can create and change
templates and template groups on the Zabbix server. It has no access to any
monitored host, and it sends no commands to them.

**What was found on the lab setup (worth fixing before real use)**

| Finding | Why it matters | Better |
| --- | --- | --- |
| The login is `Admin` with the password `zabbix`, which is Zabbix's factory default. It works against the API (tested). | Anyone who knows Zabbix can log in as a super admin | Change the password. Use a dedicated API user whose role allows only what the import needs (API access plus write access to template groups). |
| The password sits in plain text in `host_vars/zabbix_api_endpoint.yml` | Anyone who can read the repo can read it | Encrypt that file with `ansible-vault` |
| The API is reached over plain `http://...:8081` and certificate checks are off (`use_ssl: false`, `validate_certs: false`) | The login and the token cross the network unencrypted | Serve Zabbix over HTTPS, then set `use_ssl: true` and leave `validate_certs` at its default. The default of `validate_certs` is `true`, so it only matters once `use_ssl` is on. |

**Data flow:** the template text travels in the body of an API request. The
files contain monitoring definitions only, with no secrets.

---

## 10. Errors and messages

Messages marked **observed** were produced on the lab setup while writing this
page (safe, read-only tests). No failure from this role was reported to me
during the normal runs.

| Message (search for this) | Cause | Fix |
| --- | --- | --- |
| `The lookup plugin 'file' failed: Unable to access the file 'scope_not_here.yaml': File not found.` (**observed**) | A name in `zabbix_template_files` has no matching file in `files/` | Fix the name or add the file |
| `Incorrect user name or password or account is temporarily blocked.` (**observed**, API response) | The API login in `host_vars` is wrong, or the account is locked after repeated failures | Correct the password and wait a moment |
| `Could not connect to http://192.168.10.160:9/zabbix/api_jsonrpc.php: [Errno 113] No route to host` (**observed**, with a wrong port on purpose) | Wrong address, port or URL path (`ansible_httpapi_port`, `ansible_zabbix_url_path`) | Check the values in `host_vars` |
| The run shows `changed` for all six templates every time (**observed**) | Cosmetic differences, see section 7 | Nothing to fix. Optionally use exported files. |

---

## 11. Known limitations

- **All or nothing.** Every run imports all six templates, whichever hosts use
  them. Only the linking phase is scoped per host.
- **The files are the source of truth.** GUI edits to these templates are
  deleted on the next import.
- **Always `changed`.** See section 7. It hides real changes among the noise.
- **Default credentials, plain text, plain HTTP.** See section 9.
- **Run it with `zabbix_host_link`.** Phase 2 needs these templates to exist,
  and I did not run the real import on its own to see what it does to host
  links (section 7). The supported way is the full `deploy_zabbix.yml`.
- **UUIDs are identity.** If a template's `uuid` in the file is changed, Zabbix
  treats it as a different template.

---

## 12. Why this role is necessary, and what it gives you

**The problem it solves.** Templates have to exist on the Zabbix server before
anything can be monitored with them. Importing them by clicking through the
Zabbix UI does not scale: it is slow, easy to forget, and different on every
server. The working instructions say the same: "manually clicking import in
the Zabbix UI doesn't scale", and ask for the import to be part of the
existing Ansible automation.

**What you get from it**

| Benefit | How the role delivers it |
| --- | --- |
| **Repeatable** | The same six templates land on any Zabbix server the same way, from files in git |
| **Rebuildable** | Deleted the templates in the GUI? Run it again and they come back |
| **One source of truth** | The `.yaml` files define the templates. Changes are reviewed in git, not made by hand |
| **Ordered and linked** | It is Phase 1, so the templates are in place before hosts are linked to them |
| **Adapts to each host** | The templates use discovery rules, so they show only what exists on a host (VMs, filesystems, containers) without editing a template per host |
| **Small** | One task and one list. Adding a template is one file and one line |
| **Documented** | The comparison results and error messages above make the behaviour predictable |

**In one line:** it turns "click Import six times on every Zabbix server" into
one repeatable step, so the templates the agent role prepares each host for
are always there and identical.
