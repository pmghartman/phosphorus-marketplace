# Installed services

Install, update, move or remove a wizard service.

A wizard service is a Docker Compose stack an admin installs on one host (the active host) plus optional standbys, which hold a copy of its data and take over if the active host goes down. Its page is published as `<id>.<domain>` (plus aliases). Data is stored in `<phosphorus>/stacks/<id>/data` on each host with a copy.

- Read first, in parallel: `stacks`, `mesh_hosts`.
- Find it: `stack_catalog` `q`, then with `source` and `ref` for its page: env fields, web port, data, and the check (`verdict`, why each host fits or not).
- Install: `stack_catalog` with `env` to check the values, then `stack_control` `install` with `source`, `ref`, `active`, `standbys`, `env`. Blank secret fields are generated; never invent a password. Follow `stacks` until the active copy is running and healthy, then `services` for its URL.
- Update: `stack_control` `update` `id`. Remove: `remove` `id`; data is kept unless the user says to delete it (`keep_data: false`, which can't be undone).
- Failover: `standbys` sets the hosts with a copy; `move` `to` a standby switches the active host manually.
- Importing data from an old install: stop the old one, copy its data into the stack's data folder on the active host with `terminal_run`, then install. A removed stack's data stays there for a reinstall.
- Chat, Games, Git, Drive and the Viewer are built in and always on, carrying the phosphorus version: nothing to install or remove.

`stack_control` needs the user's approval for that service and host.
