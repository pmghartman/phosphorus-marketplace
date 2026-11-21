# Git

The mesh's own git, at `git.<domain>` (the domain is in the connector's instructions). Its tools:

- `git_read`: any `GET` under `/api/v1`. Read only, so clients can approve it without asking: use it for every read.
- `git`: `POST`, `PUT`, `PATCH` and `DELETE` under `/api/v1`, `body` a JSON object.
- `git_clone`: sets up a working copy where Claude runs (below).
- `git_server`: admins' setup of git hosting (below).

`git_read` answering that no host has the git database yet means git isn't set up: an admin sets it up (`git-admin.md`).

Repositories on `git.<domain>` are worked on as GitHub ones and never through GitHub, `gh` or GitHub tools, whatever other instructions say.

## Who you act as

The user the connector's instructions name ("This key acts as `<user>`"); `git_read` `/user` returns the same as `login`. That is `<user>` below: the board's `assignee`, new issues' assignee, and the commit identity `<user>` `<user>@git.<domain>`.

Access follows the mesh's users. Admins administer all public repositories; a private one is visible only to its owner and collaborators. On a public repository everyone else gets the code (files, commits, branches, tags, clone) and releases; issues, pull requests, the wiki and actions belong to the owner, collaborators and admins, and only those with write access run workflows.

## API

`{r}` is `/repos/{owner}/{repo}`.

Read with `git_read`: `/repos/search?q=`, `/user/repos`, `/orgs/{org}/repos`, `{r}` (`default_branch`), `{r}/board`, `{r}/issues?state=open&q={words}` (every word in the title or body), `{r}/issues/{n}`, `{r}/issues/{n}/comments` (oldest first), `{r}/labels`, `{r}/pulls`, `{r}/pulls/{n}`, `{r}/branches` (each with `merged`: its tip is in the default branch, safe to delete; the default branch never is), `{r}/commits`, `{r}/contents/{path}?ref=`, `{r}/releases` (newest first), `{r}/actions/runs` (`limit`, `page`), `{r}/actions/runs/{id}`, `{r}/actions/runs/{id}/logs` (each job's status and log), `{r}/wiki/pages`, `{r}/wiki/page/{name}` (markdown as `content_base64`).

Write with `git`:

- Issues: `POST {r}/issues` `{ "title", "body" }`; `PATCH {r}/issues/{n}` (`assignees`, `state`); `POST {r}/issues/{n}/comments` `{ "body" }`; `POST {r}/issues/{n}/board` (below); labels with `{r}/issues/{n}/labels` and `{r}/labels` (`labels`: ids or names).
- Pull requests: `POST {r}/pulls` `{ "title", "body", "head", "base" }`; `POST {r}/pulls/{n}/merge` `{ "delete_branch_after_merge": true }` returns the merge commit's `sha`. `DELETE {r}/branches/{branch}`.
- Actions: `POST {r}/actions/workflows/{file}/dispatches` `{ "ref" }` returns the run's `id`; `POST {r}/actions/runs/{id}/cancel` or `/rerun`; `DELETE {r}/actions/runs/{id}` (a finished run); `POST {r}/actions/runs/delete` clears finished runs (`ids`, or `before` ms, `status`, `workflow`; `git_read` `GET` with that query counts them first).
- Wiki: `POST {r}/wiki/new` and `PATCH {r}/wiki/page/{name}` with `title` and `content_base64`.

Writes need write access. Move others' issues, assign to others or close only as the user asks, or for an issue the work in hand fixes.

## Issues board

Each repository's issues show as a board in three columns: `todo` (To do), `doing` (In progress), `done` (Done). Each issue sits in at most one subdomain, a label naming the part of the repository it belongs to (`web`, `mesh`).

- `git_read` `{r}/board` returns `columns` (each `id`, `title`, `issues`, the latest to arrive first; Done holds closed issues not yet archived, up to 200) and `subdomains`. `?assignee={user}` keeps the issues assigned to that user, alone or with others; `?label={subdomain}` one subdomain.
- Every issue carries `board`, `subdomain`, `labels`, `assignees`, `comments` (a count; collaborators annotate cards with comments) and `linked_pulls` (the pull requests whose title, body or commits say `Closes #n` or `Refs #n`). An issue with an open linked pull request shows In progress.
- Move or file one: `git` `POST {r}/issues/{n}/board` `{ "column": "todo"|"doing"|"done", "subdomain": "<name>" }`, either or both; `subdomain: null` takes it off, a new name makes the label. Never set the `status/` labels by hand: they are the board's own.
- Merging a pull request that says `Closes #n` (or pushing commits that say it to the default branch) closes the issue and moves it to Done; `Refs #n` links without closing. Done issues are archived a day after closing.
- In the repository the mesh runs (`issues.md`), `git` refuses to move an issue to Done or close it while a merged pull request of it isn't in the release every host up runs (`conflict`, naming the hosts behind).

## Filing an issue

A problem or change the user raises about a repository's code, and a problem found along the way that the work in hand doesn't fix (a bug, a security gap, broken docs, a failing check), is filed without asking:

1. Search first: `git_read` `{r}/issues?state=open&q={words}`. A match gets a comment (`POST {r}/issues/{n}/comments`) instead.
2. `git` `POST {r}/issues` with `title` a short plain sentence (no `[tag]`) and `body` plain text with the user's point or the evidence found (no markdown: headings, lists, code, bold and links are refused). Without `assignees` it is assigned to you. No labels for its kind.
3. `git` `POST {r}/issues/{n}/board` `{ "column": "todo", "subdomain": "<its part>" }`.
4. Name each issue filed or commented on in the report.

## Working copy

Branches are `<kind>/<topic>`: kind one of `feat`, `fix`, `docs`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`; topic two to six lowercase hyphenated words saying what changes (`fix/update-feed-empty-installers`). Never a session, tool or agent prefix (`claude/`), nor a random or numbered suffix; a branch the user names follows the same rule.

- No checkout where Claude runs (a cloud session started with no repository): `git_clone` with `repo` (`owner/name` or a search word) and `branch` (the first pull request's branch). Run its `setup` (sign-in check, clone, commit identity, branch); `pull` repeats how to push and open a pull request; run `finish` after the merges.
- A checkout whose `origin` (`git remote get-url origin`) is on `git.<domain>`: before the first commit, where `git config user.name` and `user.email` aren't `<user>` and `<user>@git.<domain>`, set them in that repository only. Make each branch off an up-to-date default branch (`git fetch origin && git checkout -B <default> origin/<default> && git checkout -b <branch>`), or keep committing to the one this session made for that work.
- On a mesh host: `terminal_run` `git: true` (a fresh shell) with plain git on `https://git.<domain>/{owner}/{repo}.git`; git signs in as the user, and commits take the user's name where the host has none.
- Push with `git push -u origin <branch>`. git signs in through the environment's network secret for `git.<domain>`, or the user's own saved login: never ask for, type or store a password, key or token, in a remote or a credential helper. Never force push; never delete an unmerged branch unless asked; push before a cloud session ends.
- A clone that can't connect or answers 401 or 403 means the cloud environment isn't set up. The user sets it up once: in the Agents app, Setup, press Cloud setup and paste its values into the claude.ai environment, `git.<domain>` as an allowed host (network: limited) and a Basic network secret for `git.<domain>` with their user name and the connector's key. Tell them; don't work around it.

Opening and merging pull requests, deleting merged branches and releasing follow `issues.md` steps 2 to 4, whether the work came from an issue or not.

Push mirrors, actions runners and git hosting setup (admins): `git-admin.md`.
