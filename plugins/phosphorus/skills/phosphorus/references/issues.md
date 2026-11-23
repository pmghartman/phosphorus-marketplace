# Fix a repository's issues, release, install

The user asks to fix a repository's issues ("fix the issues on phosphorus", "clear the board"), or reports problems or asks for changes in a repository's code ("git master is wrong", "widgets don't sync", "redesign the agents page"): any change, a feature as much as a fix, runs this job to its end. For a report the run's work is the user's points, each filed first, not the whole board. Pull requests, merges, branch deletes, releases and the mesh update never need a question: they are the owner's standing instruction, so a report never ends asking for them. Ask only for what the request leaves open: which repository, if it's unclear, or a decision an issue itself needs. The Agents app's Fix issues button runs this same job on a mesh host for the issues its prompt names; such a run never asks at the keyboard (its questions are step 1's `Question:` comments).

A call the connector's scopes refuse (`forbidden`) is never worked around by another tool, path or sign-in: go on with the other issues and name what was refused in the report.

Read `git.md` and `agents.md` with this file: git's API, board, filing and working-copy sections are the how, agents' the claims, notes and runner turns other agents rely on; this file is the order. Reads are `git_read`, writes `git`, paths under `{r}` = `/repos/{owner}/{repo}`. `<user>` is the user you act as (`git.md`, Who you act as).

The repository the mesh runs is the one its update feed is read from: pmh/phosphorus unless the mesh was pointed elsewhere. Its release goes on to install on the mesh (step 6); other repositories end at step 6's release. In every repository commit and pull request bodies say `Closes #n`. Pull requests merge and release on a release train (`agents.md`, Release trains). Where the repository has a release workflow (`{r}`'s `release_workflow`), the merge leaves each issue In progress marked merged and it closes to Done once its release is verified on the mesh (step 7); elsewhere the merge closes it.

Every change is an issue. Keep a running list of each pull request (number, branch, issues, merge commit) and each issue's state; report from it.

## 1. Pick the work

1. File each problem or change the user raised as its own issue (`git.md`, Filing an issue) before any fix, and the same for points raised later in the conversation, with no `assignees`: they land unassigned with `<user>` as who asked for them, and taking one assigns it. `"column": "doing"` in the `POST {r}/issues` files and starts one in one call. These are the run's work.
2. In parallel, once a run: `git_read` `{r}/board?assignee=<user>&unassigned=true`, `work_board` `{ "repo": "<owner>/<repo>", "issues": [<the issues just filed>] }` (its `related` ranks the open issues related to them) and `git_read` `{r}` (`default_branch`). The work is every issue in `doing` then `todo` assigned to `<user>` (alone or with others) or to no one, oldest first within a column, less those another session's live or idle claim holds (`agents.md`): taking one of those means settling it with its holder first, so leave them unless the user asks for them, and name them in the report. An issue assigned only to other users is theirs to pick up: take it only when the user asks for it, and settle it with their agents; otherwise leave it and name it. After these reads, the only board calls are the moves below.
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

1. Move it to In progress: `git` `POST {r}/issues/{n}/board` `{ "column": "doing" }`; that claims it for this session and assigns it to `<user>`. Updates in its `work` (another holder, or another user assigned): settle each with them by `work_note` before editing for it (`agents.md`). A `conflict` naming notes waiting for this session: answer them, then move it again. Make the fix, with a test only where the repository's rules ask for one.
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

## 5. Board the train and merge at its turn

Each pull request through step 4 boards its repository's release train at once: `work_ready` `{ "repo": "<owner>/<repo>", "pull": n }`. Don't wait for the run's other groups, nor for anyone else's work: a train takes what is ready and the next train takes what is ready later. Its checks still running: it boards pending and holds the train only until that run ends. A failed run is refused: step 4 first.

Then wait on the train: `work_train` `{ "repo", "wait": 50, "phase": "<the last answer's phase>" }`, again after each answer, and do what its `next` says. With several pull requests on trains, wait on the repository's trains, not on each one. Every call moves the trains on; the first to board conducts (steps 6 and 7), and a live member takes over where it went quiet. A train stuck on something only a person can fix: say so in the report for a person admin to cancel it in the Agents app's Work page; your own pull request comes off a boarding train with `work_train` `cancel` and why. A train building with no release run, or installing, for over two hours ends by itself.

At a pull request's turn (`your_state` `merging`): on its branch `git fetch origin && git merge origin/<default>`, resolve, `git push origin <branch>`; run the checks on that merged head (step 3's `job_run`, `job_list` `wait`); green: `git` `POST {r}/pulls/{n}/merge` `{ "delete_branch_after_merge": true }`, noting the merge `sha` (the train records it and passes the turn on). The forge refuses a branch behind the default branch, or a run on its head queued, running or failed (409, naming it): merge the default branch in again and rerun the checks. The checks on the merged head fail: `work_train` `{ "repo", "leave": true, "why": "<the failing check>" }`, fix it on the branch (step 3), step 4, and `work_ready` again for a later train. Never force push.

Then `git fetch origin --prune && git checkout -B <default> origin/<default>`, remove the merged groups' worktrees (`git worktree remove ../wt/<topic>`) and delete the local branches whose pull requests merged (`git_clone`'s `finish` does the fetch and the branches). `git_read` `{r}/branches`: delete each with `merged: true` (`git` `DELETE {r}/branches/{name}`); list each `merged: false` one in the report with its last commit.

## 6. Release (the conductor)

Once every member of the train merged or left, the train is `building`. Only its conductor (`work_train` `conductor: true`) releases; the others wait on the train. The release workflow is `{r}`'s `release_workflow` (`release.yml` in phosphorus); a repository without one ends its train at the merges: say so and stop here. No question and no batching beyond the train: it is the batch.

1. Dispatch it as `next` says: `git` `POST {r}/actions/workflows/{file}/dispatches` `{ "ref": "<default>" }`, inputs left at their defaults, after the train's last merge (the release guard refuses an older commit). The train records the run. With `"following": true`, or a `conflict` naming a run already going (`releases/pending` `run`), that run is followed instead, never doubled; one that built a commit older than the train's tip is dispatched again once it ends.
2. Wait on the train (`work_train` `wait`), which follows the run, or on the run itself (`git_read` `{r}/actions/runs/{id}?wait=50&status=<last seen>`). `queued` for over ten minutes: no runner matches its `runs-on`; report it.
3. `failure` ends the train (`release failed`); its merges ride the next train's release. `git_read` `{r}/actions/runs/{id}/logs?failed=1` and read the failing step.
   - The code: fix it on a new small `fix/<topic>` branch off the default branch (no issue needed), run the checks, push, open its pull request and board it with `work_ready`; the next train releases it with these merges.
   - Outside the code (a runner or host down, git between leaders, a network blip): board nothing; the next train's release, or a release dispatched again once the runners are back, carries the merges (`releases/pending`).
   - The same failure three runs in a row, or one only the user can fix (a secret, a host to bring back): stop and report it with the log lines.
4. `success`: the train is `installing`, its `version` the new release (`v<version>`, `last_release.tag_name` of `releases/pending`).

### Install on the mesh

Only the repository the mesh runs; needs an admin key (the connector's instructions say), else report the release and stop. The conductor's release always goes on to the mesh update and the health check, with no question.

1. In parallel: `update_status`, `mesh_hosts` `health: true`, `services` (the snapshot for the health check). An update already running: follow it to the end first; never start a second.
2. `update_control` `fetch`, then `update_status` every 15 to 30 s until `fetch` shows the new version and every host has it `fetched`. Not there after five minutes: fetch once more, then report.
3. `update_control` `start`, noting the time. `update_status` every 15 to 30 s until it ends, reporting each host's change briefly.
4. A host failed: quote its error. A cause outside the code (host down, download blip): `update_control` `start` with `hosts` set to the failed ones, at most twice. `couldn't fetch wintun` and Intel Macs: `update.md`. A failure in the release itself: fix forward (step 7).

### Health check

Once the update has finished, in one parallel batch (a host still restarting or a service still starting: check it once more a minute later):

- `mesh_hosts` `health: true`: every phosphorus host up and on the new version.
- `host_logs` on each host, `level: "error"`, `since` the update's start: new errors (compare the same window before the update where unclear).
- `services`: everything up in the snapshot still up.

It passed: `work_train` `{ "repo", "installed": true, "version": "v<version>", "evidence": "<hosts and versions, errors found or none, services>" }`; the train goes on to `verifying` and every member is told. A repository the mesh doesn't run: call it at once after the release, with the release as its evidence.

## 7. Verify and close

When the train is `verifying` (`your_state` `merged`, also for a merge of an earlier cancelled or abandoned train that it carries), each member checks its own issues on the mesh, the hosts now running the new version: what each issue says is fixed, seen working (a call, a page, a log line, a screenshot). Then `work_verified` `{ "repo", "evidence": "<what was checked and what it showed>" }` (its issues default to its pull request's): each gets `Released in v<version>, on the mesh. <evidence>` and closes to Done. The conductor also verifies, with `"pull": n`, each merge its `next` names whose session went quiet. Issues `releases/pending` `merged` lists that no train carries (merged before trains, or by a person) get the same from whoever installed the release: `work_verified` `{ "repo", "issues": [...], "evidence" }` (it closes only issues a merge holds), or the comment by hand.

A failure the release caused (a panic, an error that started with the new version, a service that stopped answering, an issue not fixed after all): `work_verified` with `"ok": false` and what failed; the issues stay open. When the train's manifest pins it on one pull request, that one is reverted first: `git revert -m 1 <its merge sha>` on a `fix/<topic>` branch, its pull request (its body says `Reverts pull request #<n>`, never naming the issue, which stays open), checks, `work_ready` with `"revert": true` (a revert goes first on the next train); then fix forward with its own pull request saying `Closes #n` on a later train. Not pinned on one: file it with the log lines and fix forward. Never roll a host back. A failure outside the release (a host off the tailnet, a disk full): report it. The same failure after two fixes: stop and report.

## 8. Release the claims, with the evidence

Once the train is `done`: `work_release` `{ "repo": "<owner>/<repo>", "outcome": "done", "summary", "note" }`, the note an update for the next agents in the repository (what changed, what to watch); issues dropped were moved back to To do already. Each issue's closing comment carries its evidence: the test that covers it and what was checked on the mesh.

## Report

Lead with the result: the version now on the mesh, how many issues wait for a release, or where the run stopped and why. Then one line per train (number, members, conductor, release run, version, outcome), one per pull request (number, branch, issues, merge commit, checks), one per issue (number, title, pull request, release, Done or why not, and a scoped one's `Scope:` comment link), the issues filed and those left to others, each `Question:` asked, each call refused, the issues waiting for a release, branches deleted and left, each run cancelled, each release run and its outcome, each host's version, and what the health check found, evidence lines verbatim.
