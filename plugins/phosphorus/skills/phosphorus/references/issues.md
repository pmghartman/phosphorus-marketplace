# Fix a repository's issues, release, install

The user asks to fix a repository's issues ("fix the issues on phosphorus", "clear the board"), or reports problems or asks for changes in a repository's code ("git master is wrong", "widgets don't sync", "redesign the agents page"): any change, a feature as much as a fix, runs this job to its end. For a report the run's work is the user's points, each filed first, not the whole board. Pull requests, merges, branch deletes, releases and the mesh update never need a question: they are the owner's standing instruction, so a report never ends asking for them. Ask only for what the request leaves open: which repository, if it's unclear, or a decision an issue itself needs. The Agents app's Fix issues button runs this same job on a mesh host for the issues its prompt names; such a run never asks at the keyboard (its questions are step 1's `Question:` comments).

A call the connector's scopes refuse (`forbidden`) is never worked around by another tool, path or sign-in: go on with the other issues and name what was refused in the report.

Read `git.md` and `agents.md` with this file: git's API, board, filing and working-copy sections are the how, agents' the claims, notes and runner turns other agents rely on; this file is the order. Reads are `git_read`, writes `git`, paths under `{r}` = `/repos/{owner}/{repo}`. `<user>` is the user you act as (`git.md`, Who you act as).

The repository the mesh runs is the one its update feed is read from: pmh/phosphorus unless the mesh was pointed elsewhere. Its release goes on to install on the mesh (step 6); other repositories end at step 6's release. In every repository commit and pull request bodies say `Closes #n`, and the merge closes each issue and moves it to Done.

Every change is an issue. Keep a running list of each pull request (number, branch, issues, merge commit) and each issue's state; report from it.

## 1. Pick the work

1. File each problem or change the user raised as its own issue (`git.md`, Filing an issue) before any fix, and the same for points raised later in the conversation, with `"assignees": ["<user>"]`. `"column": "doing"` in the `POST {r}/issues` files and starts one in one call. These are the run's work.
2. In parallel, once a run: `git_read` `{r}/board?assignee=<user>&unassigned=true`, `work_board` `{ "repo": "<owner>/<repo>" }` and `git_read` `{r}` (`default_branch`). The work is every issue in `doing` then `todo` assigned to `<user>` (alone or with others) or to no one, oldest first within a column, less those another session's live or idle claim holds (`agents.md`): leave those and name them in the report. An issue assigned only to other users is theirs, even when the user asks to clear the board: leave it and name it. After these reads, the only board calls are the moves below.
3. Read each issue once, all in parallel: `git_read` `{r}/issues/{n}` and `{r}/issues/{n}/comments`. Its `linked_pulls` holds an open pull request this run didn't open, or another session's claim holds it: someone's work in hand; skip it and say so. It needs a decision only the user can make (a design choice, a secret, missing facts): post one comment that asks it, leave it in To do, go on (a Fix issues run's question shows in the Agents app's Needs you as choices). The comment is plain text: `Question: <header, at most 12 characters>`, the question on its own lines, a blank line, `Pick one:` (`Pick any:` only when the choices aren't exclusive), then 2 to 4 lines `1. <label> - <what it means>`, the recommended one first with ` (Recommended)` ending its label; no Other choice, the page adds one. With no real choices, write the `Question:` line and the question alone. An issue whose newest comment starts `Question:` waits for its owner: skip it. One with an answer below its `Question:` is ready (any later comment; the page writes `Answer: <label>`, labels joined by commas for `Pick any:`, or the person's own words, maybe more after a blank line): read the answer and do the work.
4. Working copy, once: `git_clone` where there is none, with `branch` the first group's branch (step 2), else the checkout here (`git.md`, Working copy). Read the repository's `CLAUDE.md` or README for its check commands.

## 2. Plan the run

Group the issues before the first change: those in the same subdomain, crate, app or files share one pull request (issues touching the same files always do); a lone issue can be its own. Each group has its own `<kind>/<topic>` branch named for its change (`fix/git-board-issues`). Claim each group's issues, branch and paths with `work_claim` as the plan settles, and settle each overlap it reports by `work_note` before editing those paths; a subagent working a group claims it in its own session, so the main session lets those issues go first (`drop_issues`).

- A clear issue (one crate or page, no design choice, no change to stored state, a protocol or an API others use) is fixed directly, with no scoping pass.
- A broader or unclear issue is scoped before any change: find where and what, and post it as a comment that starts `Scope:` with the files and functions, the cause or the change, what must not change, and the checks to run. An issue that already has a `Scope:` comment, or a body that names its files, is not explored again: trust it and check as you open files.
- The client can start subagents (Claude Code): give each group to a fresh subagent in its own worktree (`git fetch origin && git worktree add ../wt/<topic> -b <branch> origin/<default>`), at most three at once. Its brief is the issues' text, their `Scope:`, the repository's rules file, and steps 3 and 4 of this job; it scopes a broader issue that has no `Scope:` first. The main session keeps the running list and reads no code: one long session costs far more per step than several short ones.
- It cannot: work one group at a time, making each branch as you start it (`git fetch origin && git checkout -B <default> origin/<default> && git checkout -b <branch>`), and re-read nothing already read.

## 3. Fix each issue on its group's branch

For each issue in the group:

1. Move it to In progress: `git` `POST {r}/issues/{n}/board` `{ "column": "doing" }`; that claims it for this session. A `conflict` or `forbidden` answer means it is not this run's to take: leave it, name it in the report, go on. Make the fix, with a test only where the repository's rules ask for one.
2. Commit, one commit for the issue, its body ending `Closes #n`. Push the branch (`git push -u origin <branch>`) once the group's commits are ready rather than after each one: every push starts the shared runners. Don't merge.

Checks belong to the mesh, not to where the agent runs. Before the pull request, run them on the pushed branch: `job_run` `{ "repo": "<owner>/<repo>", "ref": "<branch>", "command": "<the checks>" }` (`host` or `labels` to choose the host; needs `git:code`), then `job_list` `{ "job", "wait": 50, "status": "<last seen>" }` until `success`, `failure`, `cancelled` or `lost`. `failure`: read `job_log`, fix in a new commit, push, run again. The job runs in a fresh checkout that is removed when it ends; `PHOS_REPO_KEEP` outlives it and holds `CARGO_TARGET_DIR` unless `env` sets one, so builds stay warm. No `job_run` on this key (a Fix issues run already works on a mesh host): run them in the checkout.

The repository has a pull request workflow (a file under `.phosphorus/workflows/` or `.github/workflows/` with `on: pull_request`; phosphorus has none): run only the quick checks for what changed (the touched crate's or page's lint and tests); the workflow runs the whole set by itself on the mesh's runners once the pull request is open. It has none: run the whole set from the repository's rules.

Dropping an issue (checks you can't make pass, out of time, a fix too risky to merge unasked): comment why on it (what was tried, the failing output), move it back with `{ "column": "todo" }`, drop its change (`git restore --staged --worktree <files>`, and remove files it added; a pushed commit: `git revert` it and push), go on. A problem outside the issue in hand is filed as an issue of its own (`git.md`, Filing an issue), not fixed in that commit, and not taken up in this run.

When the group's issues are done, open its pull request: `git` `POST {r}/pulls` `{ "title", "body", "head": "<branch>", "base": "<default>" }`. The title is a short plain sentence for the group's change; the body plain text, one line on what changes, then one `Closes #n` line per issue fixed in it. That links each issue to it. Note its `number`. A group whose issues were all dropped gets no pull request: delete its branch (`git checkout <default> && git branch -D <branch>`, and `git push origin --delete <branch>` if pushed).

## 4. Checks and review

1. For each pull request, all in parallel, `git_read` `{r}/pulls/{n}`: `checks` lists the runs on its head commit (`id`, `name`, `status`), started by the forge when it opened. Wait on each with `git_read` `{r}/actions/runs/{id}?wait=50&status=<last seen>` until `success`, `failure` or `cancelled`; never poll faster. No `checks` two minutes after opening (no pull request workflow, or its paths do not cover the change): the checks you ran before opening are the gate; go on.
   - `failure`: `git_read` `{r}/actions/runs/{id}/logs?failed=1`, fix on the branch, quick checks, push; the new commit starts a new run. The same failure three runs in a row: leave the pull request open and report it with the log lines.
   - A run whose other jobs passed and whose one job has been queued over ten minutes has no runner up for it (a laptop asleep): cancel the run (`git` `POST {r}/actions/runs/{id}/cancel`) and say so in the report.
2. Review only a broad change (several parts at once), or a change to access control, stored formats or a protocol. Where the repository has a reviewer role (`.claude/agents/verifier.md`, or one its rules file names) and the client can start subagents, the main session runs it on the pull request's diff as an agent that did not write the change, and posts its report as a comment on the pull request (`git` `POST {r}/issues/{n}/comments`). Fix what it ranks as must fix before merging; answer a finding you disagree with in a comment and leave it for the user.

## 5. Merge, then delete merged branches

Nothing merges until every group's pull request is open and through step 4. Then merge them one after another: `git` `POST {r}/pulls/{n}/merge` `{ "delete_branch_after_merge": true }`, noting each merge `sha`. Each issue its pull request names with `Closes #n` (or `Refs #n`) closes and moves to Done. The forge refuses a merge (409, naming the run) while a run on the head commit is queued, running or failed: step 4 clears it. One it refuses for a conflict (`conflicts in <files>`): `git fetch origin && git checkout <branch> && git merge origin/<default>`, resolve, re-run the checks, `git push origin <branch>`, step 4, merge again. Never force push.

Then `git fetch origin --prune && git checkout -B <default> origin/<default>`, remove the merged groups' worktrees (`git worktree remove ../wt/<topic>`) and delete the local branches whose pull requests merged (`git_clone`'s `finish` does the fetch and the branches). `git_read` `{r}/branches`: delete each with `merged: true` (`git` `DELETE {r}/branches/{name}`); list each `merged: false` one in the report with its last commit.

## 6. Release train

After the merges, `git_read` `{r}/releases/pending`: `merged` lists the issues whose merged pull requests are not yet in the release every host runs (closed already; always empty outside the repository the mesh runs), `run` the id of a release run already queued or running. Release whenever something merged in this run, or `merged` is not empty: no question, no batching; a run already queued or running (`run`) is followed instead of dispatching another. The release workflow is a file under `.phosphorus/workflows/` (or `.github/workflows/`) with `on: workflow_dispatch` (`release.yml` in phosphorus); none: say so and stop here.

1. `git` `POST {r}/actions/workflows/{file}/dispatches` `{ "ref": "<default>" }`, inputs left at their defaults. It returns the run's `id`. With `"following": true` it is another session's run of the same workflow: follow that run and don't dispatch again. A `conflict` names the run that holds the runners' turn (`agents.md`): wait on it as in step 4, then dispatch, unless it was a release of this repository that already took in this run's merges (`releases/pending` `merged` empty).
2. Wait on the run as in step 4 (`?wait=50&status=<last seen>`) until `success`, `failure` or `cancelled`. `queued` for over ten minutes: no runner matches its `runs-on`; report it.
3. `failure`: `git_read` `{r}/actions/runs/{id}/logs?failed=1` and read the failing step.
   - The code: fix it on a new small `fix/<topic>` branch off the default branch (no issue needed), run the checks, push, open its pull request and merge it as in steps 3 to 5, dispatch again.
   - Outside the code (a runner or host down, git between leaders, a network blip): dispatch again unchanged.
   - The same failure three runs in a row, or one only the user can fix (a secret, a host to bring back): stop and report it with the log lines.
4. `success`: the version is `last_release.tag_name` of `git_read` `{r}/releases/pending`, `v<version>`.

### Install on the mesh

Only the repository the mesh runs; needs an admin key (the connector's instructions say), else report the release and stop. This job's release always goes on to the mesh update and the health check, with no question.

1. In parallel: `update_status`, `mesh_hosts` `health: true`, `services` (the snapshot for the health check). An update already running: follow it to the end first; never start a second.
2. `update_control` `fetch`, then `update_status` every 15 to 30 s until `fetch` shows the new version and every host has it `fetched`. Not there after five minutes: fetch once more, then report.
3. `update_control` `start`, noting the time. `update_status` every 15 to 30 s until it ends, reporting each host's change briefly.
4. A host failed: quote its error. A cause outside the code (host down, download blip): `update_control` `start` with `hosts` set to the failed ones, at most twice. `couldn't fetch wintun` and Intel Macs: `update.md`. A failure in the release itself: fix forward (Released issues).

### Health check

Once the update has finished, in one parallel batch (a host still restarting or a service still starting: check it once more a minute later):

- `mesh_hosts` `health: true`: every phosphorus host up and on the new version.
- `host_logs` on each host, `level: "error"`, `since` the update's start: new errors (compare the same window before the update where unclear).
- `services`: everything up in the snapshot still up.

### Released issues

Once the update finished on every host and the health check passed: for every issue `merged` listed before the release (the list empties once every host runs it), not only this run's, `git` `POST {r}/issues/{n}/comments` `{ "body": "Released in v<version>, on the mesh." }`. They closed and moved to Done when they merged; nothing waits on the release. A fix that didn't reach every host: say which in the report.

A failure the release caused (a panic, an error that started with the new version, a service that stopped answering): file it with the log lines, fix it on a small `fix/<topic>` branch with its own pull request (steps 3 to 5), and release, install and check again. Never roll a host back; fix forward. A failure outside the release (a host off the tailnet, a disk full): report it. The same failure after two fixes: stop and report.

## 7. Release the claims

Once the run is over: `work_release` `{ "repo": "<owner>/<repo>", "outcome": "done", "summary", "note" }`, the note an update for the next agents in the repository (what changed, what to watch); issues dropped were moved back to To do already.

## 8. Evidence

Each issue's comment once its pull request merges, or its released comment, carries its evidence that it is fixed: the test that covers it, or what was checked on the mesh. An issue another run merged gets the released comment alone.

## Report

Lead with the result: the version now on the mesh, how many issues wait for a release, or where the run stopped and why. Then one line per pull request (number, branch, issues, merge commit, checks), one per issue (number, title, pull request, release, Done or why not, and a scoped one's `Scope:` comment link), the issues filed and those left to others, each `Question:` asked, each call refused, the issues waiting for a release, branches deleted and left, each run cancelled, each release run and its outcome, each host's version, and what the health check found, evidence lines verbatim.
