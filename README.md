# arpa-iac

Ansible-managed state for the `home.arpa` homelab.

[`home.arpa`](https://codeberg.org/arpatek/home.arpa) documents the lab — what each component
is, why it was built that way, and what will trip you up. **This repo deploys it.** Docs
describe; code enforces. Neither is generated from the other.

---

## Why this exists

`home.arpa` accumulated config copies that were hand-maintained records of what had been
deployed, with nothing connecting them to the hosts. On 2026-09-14 that produced three findings
in a single evening:

| Finding | Nature |
| --- | --- |
| `smb.conf` section dividers were 79 chars on both Pis, 80 in the repo | silent drift |
| `nas-health` ran on edgerunner only — netrunner's RAID1 was unprobed | coverage gap |
| `alloy` and `node_exporter` ran on netrunner only — edgerunner unmonitored | coverage gap |

Nothing was broken. Everything was invisible. Documentation is a claim; this repo makes the
claim testable.

---

## Layout

```
arpa-iac/
├── ansible.cfg              # run from the repo root
├── inventories/homelab.ini  # every lab host, grouped
├── group_vars/
│   └── all/
│       ├── vars.yml         # plaintext — maps names onto vault_* vars
│       └── vault.yml        # ansible-vault encrypted
├── host_vars/<host>/        # per-host overrides
├── roles/<role>/            # tasks, handlers, templates, defaults
└── playbooks/               # site.yml and per-component plays
```

Host names match the [`portal-22`](https://codeberg.org/arpatek/portal-22) aliases in
`~/.ssh/config.local`, so SSH supplies the key and Ansible does not duplicate that config.

---

## Usage

```bash
ansible-inventory --graph                    # verify groups resolve
ansible practice -m ping --ask-vault-pass    # reachability
ansible-playbook playbooks/site.yml --check --diff --limit practice
```

`--check --diff` is the drift check: run it against `prod` and it reports what has changed
underneath you without touching anything.

---

## Conventions

- Run from the repo root — `ansible.cfg` is only picked up there.
- FQCN module names (`ansible.builtin.copy`), tags on every task, handlers for restarts.
- **Never add NOPASSWD sudoers rules to make Ansible convenient.** Become passwords live in the
  vault. Every host requires a sudo password, deliberately.
- Practice VMs before production, always. Roles are proven by snapshot revert and re-run, not by
  a clean second run.

## Host notes

- **`hal`** has no `python3` and uses `doas` — needs a `raw` bootstrap before any module works.
- **`mikoshi`** is the FreeIPA server. It joins the inventory for reachability and gets roles
  last; a careless play against an IPA server does real damage.
- **`glados` / `tars`** sit on darwin's UTM shared network and are reachable only from darwin.
