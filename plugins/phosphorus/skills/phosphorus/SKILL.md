---
name: phosphorus
description: "Operates the user's live, self-hosted Phosphorus mesh through the Phosphorus connector's tools. Use whenever the user names phosphorus or the mesh, one of its hosts, or asks about their own servers, homelab or tailnet, even without saying \"phosphorus\": a host up, down, slow or full; services or game servers on the mesh; installing, moving or removing a self-hosted service or app; files in mesh shares; running a command on a host; host logs; the mesh's git; fixing a mesh repository's issues through to a release and a mesh update; the mesh chat; its models and agents; a mesh update. Not for editing or building the phosphorus source code itself, apart from that issue run."
---

# Phosphorus

Phosphorus is a mesh of the user's own machines on one tailnet. Each machine running it is a **host**. A host can share folders, publish web services and game servers, run terminals, write logs, and run installed services (such as Immich) or a model. The Phosphorus connector covers the whole mesh from any host. Every call runs as the signed-in user with that user's rights; the connector's instructions list them, including admin status. Tool names may have a prefix (`mcp__phosphorus__`, `Phosphorus:`). Each tool's description is its full reference; this skill only covers how to combine tools.

This copy is shared by every user of the mesh. The connector signs in as the user and withholds what their role can't do; its instructions name what is withheld. Don't offer withheld actions; say an admin can do them.

If there are no Phosphorus tools, or the connector needs authentication, tell the user to add the connector for their mesh, `https://<their mesh domain>/api/mcp` (the Agents app, Setup, on their mesh desktop shows the exact URL): in claude.ai, Claude Desktop or Cowork, Customize, Connectors, Add custom connector; in Claude Code, `claude mcp add --transport http --scope user phosphorus https://<domain>/api/mcp`. The connector signs in with their mesh user name and password. This skill comes from the plugin marketplace `https://github.com/pmghartman/phosphorus-plugin` and updates with it. Then stop. The connector's instructions list what this sign-in can do: admin or not, saved host logins, and withheld tools. Don't offer withheld actions; say who can do them.

## Job files

Read the one that fits before the first call; a one-tool question needs none.

| request | file |
|---|---|
| how's the mesh, is everything up, check a host | `references/status.md` |
| something down, slow, full, failing or silent | `references/diagnose.md` |
| run a command or script on a host or every host | `references/run.md` |
| find, read, edit, copy, move or organize files | `references/files.md` |
| publish, change, remove or check a service, call its API; start or stop a game server | `references/service.md` |
| install, update, move or remove a service from the wizard; failover standbys; an optional app | `references/install.md` |
| update the mesh, follow, retry or cancel an update | `references/update.md` |
| read or post chat, message someone, wait for a reply (the Chat app) | `references/chat.md` |
| git repos, issues, the issues board, PRs, clone or push; set up a repo in a cloud session with none; commit or push in a checkout whose origin is the mesh git | `references/git.md` |
| fix a repository's issues end to end: fix, move on the board, merge, prune branches, release until it builds, update the mesh and health-check it | `references/issues.md` (and `git.md`) |

## Rules

- **Use Phosphorus tools for mesh resources.** Read and change hosts, shares, services, terminals, logs, git and chat only with Phosphorus tools. "Files" means mesh shares and "chat" means the mesh chat unless the user says otherwise. Other connectors are fine for other tasks (mail a summary, save a report).
- **Use real host names.** Get them from `mesh_hosts` once per session; never guess or shorten them. A device with `phosphorus: false` can't be managed; say so.
- **Batch and keep calls narrow.** Load the tools a job needs in one tool search and run independent reads in one parallel batch. Prefer the cheap call: `host_stats` with no host before one host, `files_list` `stat: true` before `files_read`, `files_search` over walking `files_list`, a `services` check over curl in `terminal_run`, `host_logs` filtered by `level`, `match`, `since`.
- **Read before writing.** `host_config` before `config_set` or `service_edit`; read the passage before `files_write` with old/new; `update_status` before `update_control`.
- **Act without asking.** The sign-in's grants scope what it may do, so a change the user asked for needs no confirmation step: `forbidden` is the only no. Use `dry_run` first where it exists to check what will change. `host_power` takes `confirm` equal to the host name.
- **No secrets.** Never ask for, accept or type a password, key or token. `terminal_run` uses the user's saved logins; report a missing login instead of asking for one.
- **Writes are audited** in `audit_log`. Report what changed, where, and how to undo it.

## Not covered by a job file

- Models: `models` shows which mesh host's model each agent (tavern while Tavern is installed) is assigned and answering with; `set_agent_model` assigns one (admin).
- The user's Phosphorus window: `open_file` opens a share's file in an editor tab, `open_terminal` opens a shared terminal tab.

## Errors

Failed calls return `code: message`.

- `bad_input`: fix the arguments from the message; never retry unchanged.
- `not_found`: re-list (`mesh_hosts`, `services`, `files_list`, `people`) instead of guessing.
- `forbidden`: the key lacks the right (share perm, admin, saved login); say so and stop.
- `unreachable`: the host is down or off the tailnet; check `mesh_hosts` `health: true`, and `host_logs` on another host.
- `key refused` (HTTP 401): the sign-in was revoked (Agents app, Access, Connectors) or invalidated by a password change. The user signs in again from the connector.

## Reporting

Lead with the answer. Use the exact host, service and path names phosphorus uses. For a change: what, where, before and after, how to undo. For a diagnosis: evidence lines verbatim, the cause, and the fix applied or proposed.

The connector's instructions name the mesh's address.
