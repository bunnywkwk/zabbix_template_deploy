# zabbix_template_deploy — the task, block by block

Plain-language walkthrough of this role's single task: what each line
does, why it is there, and whether it is really needed. Paths are
relative to this role's folder.

## What this role is for (one paragraph)

It runs **against the Zabbix server's API** (over `httpapi`, not SSH)
and puts the **definition of each monitoring template** onto the
server. A template is the "what to collect and when to alarm" recipe:
items, discovery rules, triggers. This role only makes the recipes
*exist* on the server. It does not say which host uses which recipe —
that is `zabbix_host_link`, which runs right after.

## The three moving parts

```
vars/main.yml        the list of template file names to import
files/*.yaml         the 6 template definitions (Zabbix export format)
tasks/main.yml       the loop that imports each file
```

---

## 1. `vars/main.yml` — which templates to import

```yaml
zabbix_template_files:
  - scope_host_health
  - scope_storage_filesystem
  - scope_virtualisation
  - scope_containers
  - scope_host_access
  - scope_privileged_activity
```

- A plain list of six names. Each is a file in `files/` **without** the
  `.yaml` extension (`scope_host_health` → `files/scope_host_health.yaml`).
- **Why it lives in `vars/`, not `defaults/`:** this list is a fixed
  fact about the role — it must match the files that physically ship
  inside it. Nobody should be tuning it from inventory. `vars/` has
  higher precedence than inventory values, so a stray `group_vars`
  entry cannot silently shadow it. (`defaults/main.yml` is now
  intentionally empty.)
- **About the superseded `_rsyslog` drafts:** the comment in
  `vars/main.yml` says the old `_rsyslog` variants of Host Access and
  Privileged Activity were deliberately left out because they were
  prior drafts. Only the six current files are in `files/` today; the
  list is what would keep any leftover draft from reaching the server.
- **Relevant?** Yes — it is the only place that decides what gets
  imported. Adding a seventh template means adding the file **and**
  adding its name here (and a matching entry in `zabbix_host_link`'s
  `zabbix_cluster_template_names`, or nothing will ever link to it).

---

## 2. `tasks/main.yml` — the import loop

```yaml
- name: Import each requirement-cluster template into the Zabbix server
  community.zabbix.zabbix_template:
    template_yaml: "{{ lookup('file', item + '.yaml') }}"
    state: present
  loop: "{{ zabbix_template_files }}"
  loop_control:
    label: "{{ item }}"
```

Line by line:

**`community.zabbix.zabbix_template:`**
The module that talks to the Zabbix API to create, update or import a
template. It comes from the `community.zabbix` collection and reaches
the server through the `httpapi` connection set up in inventory (see
`zabbix-deploy`'s docs) — there is no SSH and no shell involved.

**`template_yaml: "{{ lookup('file', item + '.yaml') }}"`**
- `item` is the current name from the loop, e.g. `scope_host_health`.
- `item + '.yaml'` glues the extension on → `scope_host_health.yaml`.
- `lookup('file', ...)` reads that file's text from this role's
  `files/` folder **on the control node**. (Ansible searches a role's
  `files/` directory automatically for relative names.)
- The module then sends that YAML text to Zabbix's import API. So the
  templates are shipped to the server as data in an API call — nothing
  is copied to the server's disk.

**`state: present`**
"Make sure this template exists, matching this definition." If it is
missing (deleted in the GUI, fresh server) it is created; if it
exists it is updated to match the file. This is why re-running the
role after deleting templates in the GUI simply rebuilds them.

**`loop: "{{ zabbix_template_files }}"`**
Runs the task once per name in the `vars/` list — six times. Each pass
sets `item` to the next name.

**`loop_control: label: "{{ item }}"`**
Cosmetic. Output shows `scope_host_health` per line instead of dumping
the whole file contents into the terminal (which would be hundreds of
lines per template).

- **Why needed:** a host cannot be linked to a template that is not on
  the server. This is Phase 1 of `deploy_zabbix.yml` precisely because
  Phase 2 depends on it.
- **Relevant?** Yes — it is the entire role.

---

## 3. What is inside a `files/scope_*.yaml`

You never edit these by running the role; they are Zabbix "export"
files. Three fields matter for understanding how the pipeline fits
together:

| Field in the file | Example | What uses it |
| --- | --- | --- |
| `template:` | `Scope Virtualisation KVM` | The template's **technical name**. `zabbix_host_link` links hosts by exactly this string. |
| `name:` | `'Scope: Virtualisation (KVM)'` | The display name shown in the Zabbix GUI. Not used for linking. |
| `uuid:` | `b8a9b8c2d1e443a2b5c6d7e8f9000002` | Zabbix's stable identity for the template. Re-importing the same UUID **updates** it instead of creating a duplicate. |

The six technical names, verified against the files:

| File | `template:` value |
| --- | --- |
| `scope_host_health.yaml` | Scope Host Health and Availability |
| `scope_storage_filesystem.yaml` | Scope Storage and Filesystem |
| `scope_virtualisation.yaml` | Scope Virtualisation KVM |
| `scope_containers.yaml` | Scope Containers |
| `scope_host_access.yaml` | Scope Host Access |
| `scope_privileged_activity.yaml` | Scope Privileged Activity |

These must match `zabbix_cluster_template_names` in
`zabbix_host_link/vars/main.yml` character for character. If one
changes, the other has to change with it.

---

## 4. The one behavior that surprises people

Every run reports `changed`, even when nothing was edited, and the
import **unlinks every host from every template** until
`zabbix_host_link` runs again. Cause: the module hard-codes the import
rule `templateLinkage.deleteMissing = True`, and the template files
(correctly) say nothing about hosts. So after the first link ever
happens, the server has links the file doesn't list, and the import
removes them.

Practical rule: **never run this role on its own against a live
server.** Always run it in the same execution as `zabbix_host_link`
(that is exactly what `playbooks/deploy_zabbix.yml` does). Full
explanation is in `zabbix-deploy/docs/FAQ.md` (Q7) and
`zabbix-deploy/docs/zabbix_template_deploy.md`.

## 5. Requirements

- `community.zabbix` collection on the control node.
- An `httpapi` connection to the Zabbix server with credentials that
  may import templates.
- Because it targets an API (no shell to gather facts from), the play
  that calls it sets `gather_facts: false`.

## Quick "is it really needed?" summary

| Piece | Needed? | Why |
| --- | --- | --- |
| `vars/main.yml` list | Yes | The single source of "what gets imported" |
| The import loop | Yes | Templates must exist before anything can link to them |
| `state: present` | Yes | Makes the run repeatable and rebuilds deleted templates |
| `loop_control` label | No (cosmetic) | Keeps terminal output readable |
