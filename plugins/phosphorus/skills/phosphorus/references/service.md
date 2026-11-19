# Services

Publish, change, remove or check a web service, call a service's API, and start or stop a game server.

Each host detects web servers listening on it and lists them for an admin to publish (as discovered apps). Publishing stores the port as the host's record in the mesh store, behind the phosphorus login. Removed records stay removed. Pages for wizard-installed services are managed with `stacks` and `stack_control` instead of `service_edit`. With a domain and certificate (Settings › Domain) a service gets `<name>.<domain>`, or a path if another host already uses that name. Check `services` first.

- Read first, in parallel: `services`, `host_config` on the host.
- Publish or change: `service_edit` `host`, `name` (lowercase, digits, -), `port`, `public`; the scheme is sniffed. `dry_run: true` first, show the diff, then apply once the user agrees. `public: true` puts it on the internet; only when asked. Either way the proxy tells the service who is signed in: `X-Phosphorus-User` (and `X-Phosphorus-Admin: 1`), absent for visitors not signed in.
- Remove: `service_edit` `remove: true`, dry run first.
- Verify: `services` `host` `name` until it listens and is published, then the URL from `services`.
- Game (the Games app installed; its page says which games exist): `games` to find it, `game_power` `start` or `stop`, then `game_log` if it doesn't come up.
- API: `service_request` `service`, `path`, `method`, `body`; `headers` for the service's own token only when the user gave one.
