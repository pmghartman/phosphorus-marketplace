# Diagnose

Find the cause of a broken, slow, full or unresponsive host, service or game server from its health, stats and logs.

1. Resolve the target from `mesh_hosts` and `services`, in parallel. Ask only if it stays ambiguous.
2. Gather in parallel, as they apply:
   - host slow or full: `host_stats` on it (cpu, memory, disk with percents, parts, top processes, and `usage`: its phosphorus folder by state, cache and data folder, app and stack), `host_logs` `level: warn` `since: 2h`
   - host not answering: `mesh_hosts` `health: true` (answering? its tunnel up?), then `host_logs` `source: net` `since: 2h` on it and on the control host (the hub)
   - service: `services` `host` `name`, `host_config` (its entry in `services`), `host_logs` `match: <name>` `since: 2h`
   - game (the Games app installed): `games`, `game_log`
   - mesh tunnel down or waiting: `host_logs` `source: net` `since: 2h` shows the cause (hub response, tunnel device). A host whose UDP can't reach the hub on port 41641 falls back to the https relay at the mesh's domain. On Windows the tunnel driver, wintun, is downloaded into `state/cache/tools`; a `couldn't fetch` error clears with `host_power` `restart` once the host can reach wintun.net
   - domain or public URL: `mesh_hosts` `health: true` `edge` on each edge host (dns, https and public ports), `host_logs` `match: edge` `since: 2h` there. If the domain still points elsewhere, set the nameservers shown in Settings › Domain at the registrar; the certificate is issued after that delegation
   - hub (Settings › Domain): `mesh_hosts` `health: true` `control` (`url` is the mesh's domain, `host` the hub, `serving` its state), `host_logs` `source: net` `since: 2h` there. A host that left the mesh rejoins with a new `MESH_KEY` in its `.env` on next start
   - git: `git_server` for its database copies and runners, and `storage`: each copy's `appdata/git` by part and what the storage sweep will free (`unused`); `host_logs` `match: git` `since: 2h`
3. Narrow with more `host_logs` (`match` on the first error, an earlier `since`) or a read-only `terminal_run`: `systemctl status`, `ss -ltnp`, `journalctl -u ...`, and for a full Linux disk `df -h; du -xh --max-depth=2 / 2>/dev/null | sort -h | tail -20`.
4. Confirm a past error still matters before calling it the cause: a later success line, or the current `services` check or `mesh_hosts` health.
5. Report the cause, the evidence lines verbatim, and the fix. Apply one (`service_edit`, `config_set`, `host_power` `restart` for phosphorus or `reboot` for the machine) only on the user's word, `dry_run` first where it exists, then verify with a `services` check or `host_stats`.
