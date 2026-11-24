---
name: phosphorus
description: "Operates the user's live, self-hosted Phosphorus mesh through the Phosphorus connector's tools. Use whenever the user names phosphorus or the mesh, one of its hosts, or asks about their own servers, homelab or tailnet, even without saying \"phosphorus\": a host up, down, slow or full; services or game servers on the mesh; installing, moving or removing a self-hosted service or app; files in mesh shares; running a command on a host; host logs; the mesh's git; any change to a mesh repository's code, phosphorus's own included (a fix, a feature, a redesign, its issues), through merge, release and the mesh update; the mesh chat; its models and agents; a mesh update."
---

# Phosphorus

Phosphorus is a mesh of the user's own machines on one tailnet. Each machine running it is a **host**: it can share folders, publish web services and game servers, run terminals, write logs, and run installed services (such as Immich) or a model. The Phosphorus connector covers the whole mesh from any host and signs in as the user. Its instructions name the mesh's address, the user the key acts as (admin or not), the saved host logins, the scopes a connector was granted and the tools withheld from this key: don't offer a withheld action, say an admin can do it. Tool names may have a prefix (`mcp__phosphorus__`, `Phosphorus:`). Each tool's description is its reference; this skill says how to combine them.

No Phosphorus tools, or the connector asks to sign in: tell the user to add their mesh's connector, `https://<mesh domain>/api/mcp` (the Agents app, Setup, on their mesh desktop shows it): in claude.ai, Claude Desktop or Cowork, Customize, Connectors, Add custom connector; in Claude Code, `claude mcp add --transport http --scope user phosphorus https://<domain>/api/mcp`. It signs in with their mesh user name and password. Then stop. This skill comes from the plugin marketplace `https://github.com/pmghartman/phosphorus-plugin` and updates with it.

## Job files

Read the one that fits before the first call; a one-tool question needs none. Read it from the mesh: `guide` with `topic`, the file's name without `.md` (`status`, `git-admin`). That copy ships with the mesh, so it matches the mesh's tools. No `guide` tool (an older mesh): read the bundled copy under `references/`.

| request | file |
|---|---|
| how's the mesh, is everything up, check a host | `references/status.md` |
| something down, slow, full, failing or silent | `references/diagnose.md` |
| run a command or script on a host or every host | `references/run.md` |
| find, read, edit, copy, move or organize files | `references/files.md` |
| publish, change, remove or check a service, call its API; start or stop a game server | `references/service.md` |
| install, update, move or remove a service from the wizard; failover standbys | `references/install.md` |
| update the mesh, follow, retry or cancel an update | `references/update.md` |
| read or post chat, message someone, wait for a reply (the Chat app) | `references/chat.md` |
| git repos, issues, the board, pull requests, clone, commit or push (a mesh repository, or a checkout whose origin is the mesh git) | `references/git.md` |
| git push mirrors, actions runners, git database hosts and storage, forge users (admin) | `references/git-admin.md` |
| fix a repository's issues end to end, or the user reports problems or asks for any change in a mesh repository's code (a fix, a feature, a redesign) | `references/issues.md` and `references/git.md` |
| work alongside other agents: what others work on, claim work, notes between agents, runner turns (with any work in a mesh repository) | `references/agents.md` |

## Rules

- **Use Phosphorus tools for mesh resources.** Hosts, shares, services, terminals, logs, git and chat only through them. "Files" means mesh shares and "chat" the mesh chat unless the user says otherwise. Other connectors are fine for other tasks (mail a summary, save a report).
- **Use real host names** from `mesh_hosts`, once per session; never guess or shorten them. A device with `phosphorus: false` can't be managed; say so.
- **Batch and keep calls narrow.** Load the tools a job needs in one tool search, by name, and run independent reads in one parallel batch. Prefer the cheap call: `host_stats` with no host before one host, `files_list` `stat: true` before `files_read`, `files_search` over walking `files_list`, a `services` check over curl in `terminal_run`, `host_logs` filtered by `level`, `match`, `since`, `git_read` (read only) for every git read.
- **Read before writing.** `host_config` before `config_set` or `service_edit`; the passage before `files_write` with old/new; `update_status` before `update_control`.
- **Act without asking.** The sign-in's grants scope what it may do, so a change the user asked for needs no confirmation step; `forbidden` is the only no. Use `dry_run` first where it exists. `host_power` takes `confirm` equal to the host name.
- **Ship what you change.** Every change to a mesh repository's code, from an issue run or any other request (a feature, a redesign, a fix asked in passing), goes through `issues.md` to its end: pull request, merge and release on a release train and, in the repository the mesh runs, the mesh update, health check and verification. That is the owner's standing instruction and their go-ahead for each of those steps: never ask whether to merge, release or update, and never end a report on that question. A client's own permission check that holds one of them is the client's setting, not a decision left to the owner: name in one line what it held and where it stands, and go on with the rest.
- **No secrets.** Never ask for, accept or type a password, key or token. `terminal_run` uses the user's saved logins; report a missing login instead of asking for one.
- **File mesh repository problems.** A problem the user reports in a mesh repository's code is filed as an issue without asking (`git.md`, Filing an issue), unassigned with the user as who asked for it, and named in the report. So is one found along the way that the work in hand doesn't fix (a bug, a security gap, broken docs, a failing check): it lands in To do unassigned, for any user's agent to take, not in the work in hand. Taking an issue assigns it to the taker's user.
- **Work alongside other agents.** Any work in a mesh repository follows `agents.md`: read `work_board` before taking an issue or files, settle by `work_note` a take of an issue held or assigned elsewhere with its holder, keep the session's claim current, share findings and answer questions and handoffs with `work_note` (one to your claim holds your git writes there), wait your turn on the runners, merge and release on a release train (`work_ready`, `work_train`, `work_verified`), and `work_release` when done.
- **Keep sessions short.** In an issue run, give one issue or one same-files group to a fresh agent where the client can start subagents; write what you learn to the ticket (a `Scope:` comment) before fixing; wait on a run with `wait` rather than polling; never re-read what a ticket or an earlier answer already holds.
- **Writes are audited** in `audit_log`. Report what changed, where, and how to undo it.

## Not covered by a job file

- Models: `models` shows which mesh host's model each agent (tavern while Tavern is installed) is assigned and answering with; `set_agent_model` assigns one (admin).
- The user's Phosphorus window: `open_file` opens a share's file in an editor tab, `open_terminal` a shared terminal tab.
- Agent jobs (`ws_*`, `job_*`, `artifact_*`, `recipe_*`): `agents_hosts` describes them. `job_run` with `repo` and `ref` runs a command in a fresh checkout of a pushed ref of a mesh repository; wait on a job with `job_list` `job` and `wait`.
- Claude Code sessions on mesh hosts, as the Agents app starts them (Remote Control on, the full tool set, a clone or an empty folder): `session_start` starts one (on Opus 5.5 at medium effort unless model or effort say otherwise; `default` is the host's own; `name` lists it, `plan` makes it a planning session that files linked tickets and starts a session per ticket under itself; only hosts signed in to the user's own Claude account take one), `session_read` reads its state, its screen and its latest conversation (`ask` is a permission prompt it waits at), `session_send` sends it a message, answers its prompt, switches its mode, interrupts or ends it.
- What agents did on the mesh: `agent_sessions` (each session's calls and errors; with `session`, its events, each issue's steps and its cost by tool)).

## Errors

Failed calls return `code: message`.

- `bad_input`: fix the arguments from the message; never retry unchanged.
- `not_found`: re-list (`mesh_hosts`, `services`, `files_list`, `people`) instead of guessing.
- `forbidden`: final for this key. It lacks the right (share perm, admin, saved login, or a connector scope: `<tool> needs <scope>, not in this connector's scopes`), or a person denied the call (`denied by <who>`). Say what is missing in the report and stop; never work around it with another tool, path or sign-in.
- `unreachable`: the host is down or off the tailnet; check `mesh_hosts` `health: true`, and `host_logs` on another host.
- `conflict`: the message says what has to happen first; do that, don't retry blindly.
- `key refused` (HTTP 401): the sign-in was revoked (Agents app, Access, Connectors) or invalidated by a password change. The user signs in again from the connector.

## Reporting

Lead with the answer. Use the exact host, service and path names phosphorus uses. For a change: what, where, before and after, how to undo. For a diagnosis: evidence lines verbatim, the cause, and the fix applied or proposed.
