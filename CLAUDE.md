# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Context

This is the `jbox-web` fork of `saltstack-formulas/fail2ban-formula`. The
parent directory's `CLAUDE.md` covers the fork collection, the shared toolchain
and the CI; this file only records what is specific to this formula.

## Commands

```bash
bundle install
bin/install-hooks                               # pre-commit + commit-msg hooks

bin/kitchen list                                # 3 instances: default-<platform>
bin/kitchen test default-debian-13-3008-2       # one instance, end to end
pre-commit run --all-files                      # yamllint, salt-lint, rubocop, ...
```

There is a single Kitchen suite, `default`. Its pillar is `pillar.example`
itself (`pillars_from_files` in `kitchen.yml`): editing the example changes what
the tests converge.

## Architecture — this formula predates the template layout

Do not apply the parent `CLAUDE.md`'s "Architecture of a formula" section here;
the differences are:

- **Flat states.** `fail2ban/init.sls` includes `install`, `config` and
  `service`, each a single file — no `package/`, `config/`, `service/`
  subdirectories.
- **No `tplroot`.** States import `"fail2ban/map.jinja"` literally. Only
  `_mapdata/init.sls` derives `tplroot`.
- **Two-level `map.jinja`, no YAML maps.** One inline `grains.filter_by` by
  `os_family` (FreeBSD, Gentoo, `default`) merged with `fail2ban:lookup`, then
  the whole `fail2ban` pillar merged on top. No `defaults.yaml` / `os*map.yaml`,
  no `libtofs.jinja`.
- **Cross-state links by name, not by SLS id.** Config files use
  `watch_in: service: {{ fail2ban.service }}` and the service uses
  `require: pkg: {{ fail2ban.package }}`; renaming the package or service in
  the map changes those references too.

### Pillar → files

`config.sls` turns four pillar keys into fail2ban `.local` overrides under
`{{ prefix }}/etc/fail2ban/`:

| Pillar key | File | Absent key |
|---|---|---|
| `fail2ban:config` | `fail2ban.local`, wrapped in a `[Definition]` section | file removed |
| `fail2ban:jails` | `jail.local`, one section per top-level key | file removed |
| `fail2ban:actions:<name>:config` | `action.d/<name>.local` | nothing |
| `fail2ban:filters:<name>:config` | `filter.d/<name>.local` | nothing |

All four render through the generic INI template `fail2ban/files/config.jinja`
(sections and keys sorted, list values joined on indented continuation lines).
Setting `source_path` on any of them ships that file verbatim instead.
`enabled: false` on an action or filter removes its file; `fail2ban:enabled:
false` stops and disables the service.

## Tests

`test/integration/default/controls/packages.rb` is the only InSpec control: it
checks that the package is installed, nothing about the rendered files or the
service. The `_mapdata` dump runs in `state_top` but there are no reference
dumps under `test/integration/default/files/` to compare it to.
`test/salt/states/misc/fake_log_files.sls` touches `/var/log/auth.log` so the
example `ssh` jails can start in the container.
