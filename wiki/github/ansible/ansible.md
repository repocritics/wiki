# ansible/ansible

> Agentless IT automation — push YAML playbooks over SSH, no daemon on the managed node.

[GitHub repo](https://github.com/ansible/ansible) ·
[Official website](https://www.ansible.com/) ·
[License: GPL-3.0-or-later](https://github.com/ansible/ansible/blob/devel/COPYING)

## Overview

Ansible is a configuration-management and orchestration engine created by Michael DeHaan in 2012 and acquired by Red Hat in 2015[^1]. Its defining design choice is being **agentless**: instead of installing a persistent daemon on every managed host (the Puppet/Chef model), Ansible connects over the existing SSH daemon (or WinRM on Windows), pushes and runs modules, and disconnects. The only requirement on a managed Linux host is an SSH login and a Python interpreter; there is nothing to bootstrap and no open port to add.

The user-facing surface is **YAML playbooks** with **Jinja2** templating. Tasks call modules (`apt`, `copy`, `template`, `service`, cloud modules, etc.), and modules are expected to be **idempotent** — running the same play twice should converge to the same state and report `changed` only when it actually changed something. This makes Ansible readable to operators who are not programmers, which is the whole pitch: automation that "approaches plain English." The tradeoff is that YAML-plus-Jinja2 is not a real programming language, and non-trivial logic (loops, conditionals, dynamic data shaping) becomes awkward and verbose the moment you push past declarative config.

This repository is **ansible-core** (the engine, versioned 2.x), not the batteries-included `ansible` community package. Since the 2.10 reorganization (2020), the vast majority of modules and plugins were extracted into independently versioned **collections** distributed via Ansible Galaxy; ansible-core ships only a small set of builtin modules plus the execution engine[^2]. Understanding this split is the single biggest source of "why can't Ansible find my module" confusion for people coming from pre-2.10 monolithic Ansible.

## Getting Started

```bash
python -m pip install ansible-core   # the engine only
# or: pip install ansible            # engine + curated collections bundle
```

An inventory plus a playbook is the minimal working unit:

```ini
# inventory.ini
[web]
web1.example.com
web2.example.com
```

```yaml
# site.yml
- name: Configure web servers
  hosts: web
  become: true                       # privilege escalation via sudo
  tasks:
    - name: Ensure nginx is installed
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true

    - name: Deploy config from a template
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: reload nginx           # handler runs only if this task changed

  handlers:
    - name: reload nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
```

```bash
ansible-playbook -i inventory.ini site.yml
ansible web -i inventory.ini -m ping   # ad-hoc one-off command
```

## Architecture / How It Works

The control node (where `ansible-playbook` runs) does all the parsing and planning; managed nodes do nothing but execute:

1. **Inventory** — a static file (INI/YAML) or a dynamic plugin (AWS EC2, GCP, Azure, k8s, etc.) resolves the list of hosts and their group/host variables.
2. **Play → tasks** — for each play, Ansible connects to the targeted hosts (default connection plugin: `ssh`) and runs tasks in order, host-by-host in lockstep by default (the `linear` strategy — every host finishes task N before any host starts task N+1).
3. **Module shipping** — for each task, Ansible generates a self-contained Python module payload, copies it to the remote host (usually into a temp dir under the remote user's home), executes it with the remote Python interpreter, collects the JSON result, and removes the payload. This is why a Python interpreter is required on managed nodes and why interpreter discovery (`ansible_python_interpreter`) matters.
4. **Facts** — unless disabled, the `setup` module runs first and gathers host facts (OS, network, hardware) into variables. Fact gathering is a real per-host cost.
5. **Handlers** — tasks flagged with `notify` queue handlers that run once at the end of the play, and only if a notifying task reported `changed`.

**Idempotency lives in the modules, not the engine.** Well-written modules check current state before acting. But `command`, `shell`, and `raw` are escape hatches that run arbitrary commands and are *not* idempotent unless you add `creates:`/`removes:` guards or `changed_when:` logic. Overuse of `shell` is the classic anti-pattern that turns a "declarative" playbook into a fragile imperative script.

**Collections** are the distribution unit since 2.10: namespaced bundles (`community.general`, `ansible.posix`, `amazon.aws`, ...) with their own versioning and Galaxy dependency resolution via `requirements.yml`. Roles remain the in-repo reuse unit (a directory convention for tasks/handlers/templates/defaults).

**Connection and strategy plugins** are swappable: `ssh` is default, `local`/`docker`/`winrm`/`kubectl` exist; the `free` strategy lets fast hosts race ahead instead of waiting in lockstep.

## Production Notes

**Performance is the perennial complaint.** The default execution model is serial-ish and SSH-bound. Levers, roughly in order of impact:

- `forks` (default 5) controls parallel host count — raise it substantially for large fleets (`ansible.cfg` or `-f`). It is CPU/FD-bound on the control node.
- SSH **pipelining** (`pipelining = True`) removes a round trip per task by not writing the module to a temp file first; requires `requiretty` to be off in remote sudoers. Large speedup, off by default for compatibility.
- **Fact gathering** is expensive; set `gather_facts: false` when you don't need facts, or use a fact cache (`redis`/`jsonfile`).
- The **Mitogen** strategy plugin (third-party) can cut runtime several-fold by reusing interpreters and avoiding repeated SSH exec, but it is not officially supported and periodically breaks against new ansible-core releases — pin carefully.
- The `free` strategy avoids the slowest-host-blocks-everyone problem.

**Push model does not scale to enforcement.** Ansible converges state when you run it; it does not continuously enforce like an agent-based system. For scheduled/at-scale/audited operation teams reach for AWX (the upstream of Red Hat Ansible Automation Platform, formerly Tower) or `ansible-pull` (a cron-driven inversion where each host pulls and runs its own playbook).

**The 2.10 collections split breaks old assumptions.** Playbooks that used short module names now often need fully-qualified collection names (FQCN, e.g. `ansible.builtin.copy`), and modules you relied on may live in a collection you must install and version-pin separately. Reproducible runs require pinning both ansible-core and every collection in `requirements.yml`.

**YAML and Jinja2 footguns.** Bare booleans and unquoted strings get type-coerced surprisingly (`yes`/`no`/`on`/`off` → booleans; a leading zero or a colon can change parsing); `{{ }}` templating interacts with YAML quoting in ways that produce confusing errors. Complex data manipulation via Jinja2 filters becomes unreadable fast — a signal you may want a custom module or plugin instead.

**Managed-node Python.** Modules need Python on the target; minimal/container/appliance images that lack it require the `raw` module to bootstrap, or the `ansible.builtin` interpreter-discovery machinery. Python 2 removal and interpreter-discovery changes have caused real breakage across upgrades.

**Secrets** are handled by `ansible-vault` (symmetric encryption of vars files). It is adequate but not a secrets manager; integrations with HashiCorp Vault / cloud secret stores exist via lookup plugins.

## When to Use / When Not

**Use when:**
- You need agentless config management / app deployment over SSH with nothing to pre-install on hosts.
- Operators (not just developers) need to read and edit the automation.
- You want ad-hoc orchestration (rolling restarts, one-off fleet commands) alongside declarative config.
- You're doing multi-tier orchestration where ordering across host groups matters.

**Avoid when:**
- You need continuous, agent-enforced desired-state with drift correction (an agent-based tool fits better).
- You're provisioning cloud infrastructure lifecycle (VPCs, load balancers, DNS) as the primary job — that is Terraform/OpenTofu's domain; Ansible is weaker at stateful infra graphs.
- Your logic is genuinely programmatic — YAML+Jinja2 will fight you; a real language with an SDK is cleaner.
- You're running very large fleets where SSH-push latency dominates and you need event-driven reactions.

## Alternatives

- puppetlabs/puppet — agent-based pull model with continuous enforcement and a declarative DSL; use when you want hosts to self-correct drift on a schedule rather than converge only on push.
- saltstack/salt — agent (minion) or agentless SSH, event-driven bus; use when you need faster reaction at large scale and a message-bus architecture.
- chef/chef — Ruby-DSL, agent-based, strong for developer-owned infrastructure; use when the team is comfortable writing real code for config.
- hashicorp/terraform — declarative cloud-resource provisioning with state; use for standing up infrastructure, not for in-guest configuration.
- pyinfra — agentless Python-defined automation that executes over SSH much faster than Ansible; use when you want Ansible's shape but real Python and speed.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.0.1 | 2012-02 | First release by Michael DeHaan[^1]. |
| 1.0 | 2012 | Early stable line; playbooks + modules. |
| — | 2015-10 | Red Hat acquires Ansible, Inc.[^1] |
| 2.0 | 2016-01 | Execution-engine rewrite (blocks, new task/strategy internals). |
| 2.10 | 2020-08 | Collections split; most modules moved out of the repo[^2]. |
| 2.11 | 2021-04 | `ansible-base` renamed `ansible-core`[^2]. |
| 2.16 | 2023-11 | Dropped Python 3.9 control-node support; ongoing interpreter tightening. |
| 2.18 | 2024-11 | Recent stable line (ansible-core continues on 2.x)[^3]. |

## References

[^1]: Ansible project history and Red Hat acquisition (2015). https://www.redhat.com/en/about/press-releases/red-hat-acquire-ansible
[^2]: Ansible collections reorganization and `ansible-base` → `ansible-core` rename. https://docs.ansible.com/ansible/latest/reference_appendices/release_and_maintenance.html
[^3]: ansible-core changelogs / release list. https://github.com/ansible/ansible/releases
[^4]: Ansible documentation — architecture, strategies, and connection plugins. https://docs.ansible.com/ansible/latest/

## Tags

python, configuration-management, automation, devops, ssh, agentless, orchestration, infrastructure-as-code, yaml, red-hat
