# Fix a repository's issues, release, install

The user asks to fix a repository's issues ("fix the issues on phosphorus", "clear the board"), or reports problems or asks for changes in a repository's code ("git master is wrong", "widgets don't sync"). For a report the run's work is the user's points, each filed first, not the whole board. Pull requests, merges, branch deletes, releases and the mesh update never need a question. Ask only for what the request leaves open: which repository, if it's unclear, or a decision an issue itself needs.

Read `git.md` with this file: its API, board, filing and working-copy sections are the how; this file is the order. Reads are `git_read`, writes `git`, paths under `{r}` = `/repos/{owner}/{repo}`. `<user>` is the user you act as (`git.md`, Who you act as).

The repository the mesh runs is the one whose release writes the update feed: pmh/phosphorus unless `update_status` says otherwise. In it, commit bodies and pull request bodies say `Refs #n` where elsewhere they say `Closes #n`, and its issues are moved to Done only once their release is on the mesh (step 7). Other repositories end at step 4.

Keep a running list of each pull request (number, branch, issues, merge commit) and each issue's state; report from it.

## 1. Pick the work

1. File each problem or change the user raised as its own issue (`git.md`, Filing an issue) before any fix, and the same for points raised later in the conversation. These are assigned to `<user>`, so they are always the run's work.
2. In parallel, once a run: `git_read` `{r}/board?assignee=<user>` and `git_read` `{r}` (`default_branch`). The work is every issue in `doing` then `todo` whose assignees include `<user>`, alone or with others, oldest first within a column. An issue assigned only to others, or to no one, is not the run's work even when the user asks to clear the board: leave it and name it in the report. After this read, the only board calls are the moves below.
3. Working copy, once: `git_clone` where there is none, with `branch` the first group's branch (step 2), else the checkout here (`git.md`, Working copy). Read the repository's `CLAUDE.md` or README for its check commands.
4. Before starting an issue, `git_read` `{r}/issues/{n}` and `{r}/issues/{n}/comments`. Its `linked_pulls` holds an open pull request this run didn't open: someone's work in hand; skip it and say so. It needs a decision only the user can make (a design choice, a secret, missing facts): post the question as a comment, leave it in To do, go on.

## 2. Fix each issue on its group's branch

Group the issues before the first change: those in the same subdomain, crate, app or files share one pull request (issues touching the same files always do); a lone issue can be its own. Each group has its own `<kind>/<topic>` branch named for its change (`fix/git-board-issues`). Work the groups one at a time; make each one's branch as you start it: `git fetch origin && git checkout -B <default> origin/<default> && git checkout -b <branch>`.

For each issue in the group:

1. Move it to In progress: `git` `POST {r}/issues/{n}/board` `{ "column": "doing" }`. Make the fix, with a test only where the repository's rules ask for one.
2. Run the repository's checks for what changed; they pass before the commit.
3. Commit, one commit for the issue, its body ending `Closes #n` (`Refs #n` in the repository the mesh runs), and push the branch: `git push -u origin <branch>`. Don't merge.

Dropping an issue without a commit (checks you can't make pass, out of time, a fix too risky to merge unasked): comment why on it (what was tried, the failing output), move it back with `{ "column": "todo" }`, drop its change (`git restore --staged --worktree <files>`, and remove files it added), go on. A problem outside the issue in hand is filed (`git.md`, Filing an issue), not fixed in that commit.

When the group's issues are done, open its pull request: `git` `POST {r}/pulls` `{ "title", "body", "head": "<branch>", "base": "<default>" }`. The title is a short plain sentence for the group's change; the body plain text, one line on what changes, then one `Closes #n` (or `Refs #n`) line per issue fixed in it. That links each issue to it. Note its `number`. A group whose issues were all dropped gets no pull request: delete its branch (`git checkout <default> && git branch -D <branch>`, and `git push origin --delete <branch>` if pushed).

## 3. Merge, then delete merged branches

Nothing merges until every group's pull request is open. Then merge them one after another: `git` `POST {r}/pulls/{n}/merge` `{ "delete_branch_after_merge": true }`, noting each merge `sha`. `Closes #n` issues close and move to Done; `Refs #n` ones stay In progress. One the forge refuses (a conflict, or behind): `git fetch origin && git checkout <branch> && git merge origin/<default>`, resolve, re-run the checks, `git push origin <branch>`, merge again. Never force push.

Then `git fetch origin --prune && git checkout -B <default> origin/<default>` and delete the local branches whose pull requests merged (`git_clone`'s `finish` does both). `git_read` `{r}/branches`: delete each with `merged: true` (`git` `DELETE {r}/branches/{name}`); list each `merged: false` one in the report with its last commit.

## 4. Release

Only when something merged in this run. The release workflow is a file under `.phosphorus/workflows/` (or `.github/workflows/`) with `on: workflow_dispatch` (`release.yml` in phosphorus); none: say so and stop here.

1. `git` `POST {r}/actions/workflows/{file}/dispatches` `{ "ref": "<default>" }`, inputs left at their defaults. It returns the run's `id`.
2. Every minute or two, `git_read` `{r}/actions/runs/{id}` until `status` is `success`, `failure` or `cancelled`. `queued` for over ten minutes: no runner matches its `runs-on`; report it.
3. `failure`: `git_read` `{r}/actions/runs/{id}/logs` and read the failing step.
   - The code: fix it on a new small `fix/<topic>` branch off the default branch (no issue needed), run the checks, push, open its pull request and merge it as in steps 2 and 3, dispatch again.
   - Outside the code (a runner or host down, git between leaders, a network blip): dispatch again unchanged.
   - The same failure three runs in a row, or one only the user can fix (a secret, a host to bring back): stop and report it with the log lines.
4. `success`: the version is the newest release's `tag_name` (`git_read` `{r}/releases`), `v<version>`.

## 5. Install on the mesh

Only the repository the mesh runs; needs an admin key (the connector's instructions say), else report the release and stop. A request to fix its issues covers the mesh update; outside an issue run an update needs the user's yes.

1. In parallel: `update_status`, `mesh_hosts` `health: true`, `services` (the snapshot for step 6). An update already running: follow it to the end first; never start a second.
2. `update_control` `fetch`, then `update_status` every 15 to 30 s until `fetch` shows the new version and every host has it `fetched`. Not there after five minutes: fetch once more, then report.
3. `update_control` `start`, noting the time. `update_status` every 15 to 30 s until it ends, reporting each host's change briefly.
4. A host failed: quote its error. A cause outside the code (host down, download blip): `update_control` `start` with `hosts` set to the failed ones, at most twice. `couldn't fetch wintun` and Intel Macs: `update.md`. A failure in the release itself: step 7.

## 6. Health check

Once the update has finished, in one parallel batch:

- `mesh_hosts` `health: true`: every phosphorus host up and on the new version.
- `host_logs` on each host, `level: "error"`, `since` the update's start: new errors (compare the same window before the update where unclear).
- `services`: everything up in the snapshot still up.

A host still restarting or a service still starting: check it once more a minute later.

## 7. Close the released issues

Once the update finished on every host and the health check passed: for each issue whose fix merged in this run, `git` `POST {r}/issues/{n}/comments` `{ "body": "Released in v<version>, on the mesh." }`, then `POST {r}/issues/{n}/board` `{ "column": "done" }`. `git` refuses an earlier move (`conflict`, naming the hosts behind): finish steps 4 to 6 first. An issue whose fix didn't reach every host stays In progress; say why.

A failure the release caused (a panic, an error that started with the new version, a service that stopped answering): file it with the log lines, fix it on a small `fix/<topic>` branch with its own pull request (steps 2 and 3), and go round steps 4 to 6 again. Never roll a host back; fix forward. A failure outside the release (a host off the tailnet, a disk full): report it. The same failure after two fixes: stop and report.

## Report

Lead with the result: the version now on the mesh, or where the run stopped and why. Then one line per pull request (number, branch, issues, merge commit), one per issue (number, title, pull request, release, Done or why not), the issues filed and those left to others, branches deleted and left, each release run and its outcome, each host's version, and what the health check found, evidence lines verbatim.
