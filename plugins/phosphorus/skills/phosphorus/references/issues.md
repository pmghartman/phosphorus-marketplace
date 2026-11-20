# Fix a repository's issues, release, install

The user asks to fix a repository's issues ("fix the issues on phosphorus", "clear the board"). Merges, branch deletes, releases and the mesh update never need a question. Ask no question for any step. Ask only for what the request leaves open: which repository, if it's unclear, or a decision an issue itself needs. Read `git.md` first; its rules on branches, commit identity and pushing all hold.

Every call is the `git` tool under `/repos/{owner}/{repo}` unless named otherwise. Keep a running list of issue, branch, merge commit and state; report from it at the end.

## 1. Pick the work

1. `GET /board` once and `GET /` (the repository: `default_branch`) in parallel. The board is read once a run: issues close themselves from commit messages, so nothing on it needs moving. The work is every open issue in `doing` (In progress) and `todo` (To do), In progress first, then oldest first. Skip an issue assigned to someone other than the user, and say so.
2. Set up the working copy once: `git_clone` where there is none, else the checkout already here. Read the repository's `CLAUDE.md` or README for its check commands.
3. Read each issue (`GET /issues/{n}` and `/issues/{n}/comments`) before starting it. An issue that needs a decision only the user can make (a design choice, a secret, missing facts): post the question as a comment (`POST /issues/{n}/comments`), leave it in To do, and go on to the next one.

## 2. Fix each issue

No pull requests and no board calls: a fix merges straight into the default branch, and its commit's `Closes #{n}` closes the issue and moves it to Done when the default branch is pushed.

For each issue in turn, from an up-to-date default branch (`git fetch origin && git checkout -B <default> origin/<default>`):

1. Branch `fix/<topic>` (or `feat/`, `docs/`... for what the issue is), one branch per issue. Make the fix, with a test only where the repository's rules ask for one.
2. Run the repository's own checks for what changed. They must pass before a merge. Can't make them pass: comment on the issue with what was tried and the failing output, drop the branch, go on.
3. Commit with a message whose body ends `Closes #{n}`.
4. Merge it: `git fetch origin && git checkout -B <default> origin/<default> && git merge --no-ff <branch> && git push origin <default>`, then `git branch -D <branch>` (and `git push origin --delete <branch>` if it was pushed). A push refused as not fast-forward: fetch, merge the default branch again, re-run the checks, push again; never force. A merge conflict: resolve it on the branch, re-run the checks, merge again.

Issues that touch the same files go one after another.

## 3. Delete stale branches

Once the issues are done: `GET /branches`. Delete each branch with `merged: true` (its tip is already in the default branch): `DELETE /branches/{name}`. The default branch is never `merged`. Leave every branch with `merged: false`, which still holds unmerged work, and list them in the report with their last commit.

## 4. Release

Only when something merged in this run. Find the release workflow: a file under `.phosphorus/workflows/` (or `.github/workflows/`) with `on: workflow_dispatch` (`release.yml` in phosphorus). None: say so and stop after step 3.

1. `POST /actions/workflows/{file}/dispatches` `{ "ref": "<default>" }`, inputs left at their defaults. It returns the run's `id`.
2. Follow it every minute or two with `GET /actions/runs` (`workflow_runs`, newest first; that `id`'s `status`) until it is `success`, `failure` or `cancelled`. `queued` for over ten minutes means no runner matches its `runs-on`: report that.
3. `failure`: `GET /actions/runs/{id}/logs` for each job's status and log, and read the failing step.
   - Fails for the code: fix it as in step 2 (no issue needed), dispatch again.
   - Fails outside the code (a runner or host down, git between leaders, a network blip, a run stuck queued with no runner for its `runs-on`): dispatch again unchanged.
   - The same failure three runs in a row, or one only the user can fix (a secret, a host to bring back): stop and report it with the log lines.
4. `success`: the version is the newest release's `tag_name` (`GET /releases`, newest first), `v<version>`.

## 5. Install on the mesh

Only for the repository the mesh runs: its release writes the update feed (in phosphorus, the release's `update feed` step; the mesh is pmh/phosphorus unless `update_status` says otherwise). Other repositories end at step 4. Needs an admin key; without one, report the release and stop.

1. `update_status` and `mesh_hosts` `health: true` in parallel. An update already running: follow it to the end first, never start a second.
2. `update_control` `fetch`, then `update_status` every 15 to 30 s until `fetch` shows the new version and each host has it `fetched`. Not there after five minutes: fetch again once, then report.
3. `update_control` `start`. Note the time. Follow `update_status` every 15 to 30 s until it ends, reporting each host's change briefly.
4. A host failed: quote its error. A cause outside the code (host down, download blip): `update_control` `start` with `hosts` set to the failed ones, at most twice. `couldn't fetch wintun` and Intel Macs: see `update.md`. A failure in the release itself: treat it as a step 6 failure.

## 6. Health check

Before step 5 starts, keep a `services` snapshot of what is up. Once the whole update has finished, check once, in one parallel batch:

- `mesh_hosts` `health: true`: every phosphorus host up and on the new version.
- `host_logs` on each host with `level: "error"` and `since` the update's start: new errors, compared with the same window before the update where it's unclear whether they are new.
- `services`: every service that was up before the update still up.

A host still restarting or a service still starting: check that one again a minute later, once.

A failure the release caused (a panic, an error that started with the new version, a service that stopped answering): file an issue (`POST /issues` with the log lines), fix it as in step 2, and go round steps 4 to 6 again. Never roll a host back to an older version; fix forward. A failure outside the release (a host off the tailnet, a disk full): report it, don't fix it in code. The same failure after two fixes: stop and report.

## Report

Lead with the result: the version now on the mesh, or where the run stopped and why. Then one line per issue (number, title, merge commit, Done or why not), the branches deleted and the ones left, each release run and its outcome, each host's version and anything the health check found, with the evidence lines verbatim.
