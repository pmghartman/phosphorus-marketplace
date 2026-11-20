# Mesh update

Fetch, start, follow, retry or cancel a mesh update.

1. `update_status` `history: true` and `mesh_hosts` `health: true` in parallel.
2. Status: `fetch` shows the newest release fetched (each host checks the feed every few hours and pre-downloads new releases) and, per host, its version and whether that release is `fetched` there. Then the running update, who started it, each host's step and result, and the last finished update with its errors. To check for a new release: `update_control` `fetch`, then `update_status` a few seconds later.
3. Only with the user's explicit approval: `update_control` `start` (admin). Every host downloads the release first, then every host except the election leaders (failover actives, the git leader) and the one running the update installs and restarts at once; the leaders and that host follow once the rest are back, handing off to standbys already on the new version. Retry with `start` and `hosts` set to the ones that failed; `cancel`. Never start one while one runs.
4. Follow with `update_status` every 15 to 30 s until done or failed, reporting each host's change briefly. Done: `mesh_hosts` `health: true` shows every host's version.
5. A host failed: quote its error. `couldn't fetch wintun` means a Windows host failed to download the pinned tunnel driver from wintun.net; the host keeps running, and a retry with `hosts` set to it downloads again. Intel Macs stay on the last release built for them; report them as not updated and don't retry.
