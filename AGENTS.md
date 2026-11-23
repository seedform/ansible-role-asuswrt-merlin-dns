# AGENTS.md

Ansible role that manages custom dnsmasq `address=` entries on Asus routers
running Asuswrt-Merlin, and exposes them to extra bridges/VLANs by symlinking
the shared config into each bridge's `dnsmasq-*.conf.add`.

## Layout
- `defaults/main.yml` – public, user-facing variables (`dns_*`), each with a
  short comment and a commented-out example.
- `vars/main.yml` – internal constants (`_dns_*`), e.g. paths under
  `/jffs/configs`.
- `tasks/main.yml` – single `block` that imports the numbered task files,
  sets `ansible_ssh_use_tty: false` and notifies the handler.
- `tasks/1-config.yml` – renders and deploys `dnsmasq.conf.add`.
- `tasks/2-bridges.yml` – reconciles the per-bridge symlinks.
- `handlers/main.yml` – `Restart dnsmasq` (`service restart_dnsmasq`).
- `molecule/<scenario>/` – Molecule test scenarios (see Testing).
- `molecule/resources/` – files shared by the scenarios: `fake_router.yml`
  and the task files in `tasks/`.

## Target environment constraints
- The router has no Python. Use only `ansible.builtin.raw` for remote work,
  plus controller-side modules (`set_fact`, `debug`). Do not introduce
  `copy`, `template`, `file`, `command`, `shell`, `stat`, etc.
- Remote shell is BusyBox `sh`: stick to POSIX commands and flags
  (`test`, `cat`, `find`, `grep`, `ln`, `rm`). No bash-isms.
- Only `/jffs` survives reboots; persistent config goes under
  `_dns_conf_dir`.
- Keep `ansible_ssh_use_tty: false` on the block, otherwise output uses
  `\r\n` and content comparisons break.
- Playbooks run with `gather_facts: false`; don't rely on facts.

## Task patterns to follow
- Always use FQCN (`ansible.builtin.*`).
- Idempotency is done by hand, since `raw` can't detect changes:
  1. Build the desired state with `set_fact` (`_dns_*_wanted`).
  2. Read the current state with a read-only `raw` task
     (`changed_when: false`, `check_mode: false`).
  3. Compare them in a `set_fact` flag or difference lists.
  4. Make the change with `raw`, `changed_when: true`, and a `when:` guard
     on the comparison result.
- Check mode support:
  - Read-only and `set_fact` tasks set `check_mode: false` so they still run.
  - Add a `debug` task with `when: ansible_check_mode` that summarises the
    pending change and sets `changed_when` to the comparison result.
- Read-only discovery tasks validate their output with `failed_when` where
  an unexpected result would be harmful (see the bridge lookup in
  `2-bridges.yml`).
- Use `>-`/`>` folded scalars for one-line shell commands, `|` for heredocs
  and multi-line Jinja.
- Put long Jinja filter chains one filter per line, starting with `|`.
- Changes that need dnsmasq to reload rely on the `notify` on the block in
  `tasks/main.yml`. Don't add a `notify` to each task.
- New task files are numbered (`3-<topic>.yml`) and imported from the block
  in `tasks/main.yml` with `ansible.builtin.import_tasks`.

## Naming
- Public variables: `dns_` prefix, defined in `defaults/main.yml`.
- Internal variables (`vars/`, `set_fact`, `register`): `_dns_` prefix.
- Registered command results end in `_cmd`
  (e.g. `_dns_add_conf_existing_cmd`).
- Task names are short, sentence-case, imperative descriptions
  (`Deploy config`, `Find existing ...`).

## Code style
- Keep lines to 80 characters wide or less, except when readability is
  affected.
- Use double quotes for YAML strings; single quotes inside Jinja
  expressions.
- Start every YAML file with `---`.
- Short inline comments only where behaviour isn't obvious
  (e.g. `# returns empty when file doesn't exist`).

## Validation
- Lint: `ansible-lint --offline` (rules skipped in `.ansible-lint`; don't
  add new skips without a reason). Without `--offline`, ansible-lint fails
  because it computes the role name `Seedform.asuswrt_merlin_dns` from
  `meta/main.yml`, and that name doesn't meet Galaxy's naming rules.
- Run the automated tests with Molecule (see Testing).
- To check against a real router, do a dry run first:
  `ansible-playbook <playbook> --check --diff`, then a real run, then a
  second real run that should report no changes.

## Testing
The tests use Molecule with Podman. Each scenario starts an Alpine
container, which has BusyBox `sh` and no Python, like the router.
`molecule/resources/fake_router.yml` sets the container up like a router:
- `/jffs/configs` exists.
- `/etc/dnsmasq-{2,3,4,5}.conf` are dnsmasq instances for
  `br52`, `br53`, `br54` and `br520`.
- A `service` stub appends each restart to `/tmp/service.log`.

### Dependencies
- Podman, with a running machine on macOS (Podman Desktop, or
  `podman machine start`).
- A project venv in `.venv` with the Python tools:
  ```sh
  python3.14 -m venv .venv
  .venv/bin/pip install ansible-core ansible-lint molecule \
    "molecule-plugins[podman]"
  ```
- Ansible collection `containers.podman` (>= 1.8.1). Install it with
  `ansible-galaxy collection install containers.podman` if
  `ansible-galaxy collection list containers.podman` doesn't show it.

### Running
Run all commands from the role root, with the venv activated
(`source .venv/bin/activate`).
- All scenarios: `molecule test --all`. There is no `default` scenario, so a
  plain `molecule test` won't work. With `--all`, Molecule first prints
  `CRITICAL 'molecule/default/molecule.yml' glob failed`. That's harmless:
  the run continues and the exit code is still 0.
- In the `bridges` verify step, an `[ERROR] ... failed_when` line is
  expected. It's the unknown-bridge case, which the test catches with
  `rescue`.
- One scenario: `molecule test -s records`.
- Debugging a scenario:
  1. Run `molecule converge -s <scenario>` to create, prepare and converge,
     keeping the container.
  2. Then use `molecule verify -s <scenario>` or `molecule login -s <scenario>`.
  3. Clean up with `molecule destroy -s <scenario>`.

### Scenarios
| Scenario  | What it tests |
| --------- | ------------- |
| `records` | Adding, changing and removing `address=` records, from `dns_entries` and `dns_inventory_groups`, then removing every record. |
| `bridges` | Linking the config into the selected bridges' dnsmasq instances. A bridge with no dnsmasq instance must fail the run. |
| `cleanup` | Unexposing some bridges, removing stale links, then unexposing all bridges. |

Each scenario runs this sequence:
1. `prepare` – sets up the fake router, and may also apply the role to
   create a "previous" state.
2. `check` – a dry run.
3. `converge` – applies the role.
4. `idempotence` – a second converge, which must report no changes.
5. `verify` – asserts on the state. It may also apply the role again for
   follow-up cases, then call `meta: flush_handlers`.

### Writing tests
- Apply the role with `include_role` and
  `name: "{{ playbook_dir | dirname | dirname }}"`, so the tests don't
  depend on the role being installed.
- Read state with `../resources/tasks/read_state.yml`. It registers
  `_dns_test_conf_cmd`, `_dns_test_links_cmd` and `_dns_test_service_cmd`.
- Use `../resources/tasks/reset_restarts.yml` to flush handlers and clear
  the restart log. `_dns_test_service_cmd` then shows whether the next run
  restarted dnsmasq, which means it reported a change.
- Every test play sets `module_defaults` for `ansible.builtin.raw` to
  `executable: /bin/sh`. The `containers.podman.podman` connection runs
  commands without a shell, so `&&`, pipes, redirects and heredocs would
  break. Over SSH, the router's login shell does this job.
- Test-only variables use the `_dns_test_` prefix.
- Shared task files go in `molecule/resources/tasks/`. Otherwise
  ansible-lint parses them as playbooks.
- Keep platforms on `pre_build_image: true`. Building an image would
  install Python and hide accidental use of Python-based modules.

## Documentation
- When adding or changing a public variable, update `defaults/main.yml`
  (with a commented example) and the variable table and example
  configuration in `README.md`.
