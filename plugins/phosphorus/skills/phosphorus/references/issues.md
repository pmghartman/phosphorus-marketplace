# Fix a repository's issues, release, install

The user asks to fix a repository's issues ("fix the issues on phosphorus", "clear the board"), or reports problems or asks for changes in a repository's code ("git master is wrong", "widgets don't sync"). For a report, the run's work is the user's points, each filed first (step 1.1), not the whole board. Pull requests, merges, branch deletes, releases and the mesh update never need a question. Ask no question for any step. Ask only for what the request leaves open: which repository, if it's unclear, or a decision an issue itself needs. Read `git.md` first; its rules on branches, commit identity and pushing all hold.

Every call is the `git` tool under `/repos/{owner}/{repo}` unless named otherwise. Keep a running list of each pull request (number, branch, issues, merge commit) and each issue's state; report from it at the end.

## 1. Pick the work

1. The user's points: before any fix, file each problem or change the user raised as its own issue, following `git.md` step 8 (search open issues first and comment on a match instead; a short plain title, the user's point and the evidence in a plain body, its subdomain, To do). Points raised later in the conversation are filed the same way when they come. These issues are the run's work, alongside any the user asked for from the board.
2. `GET /board` once and `GET /` (the repository: `default_branch`) in parallel. The board is read once a run; after that, the only board calls are moving each issue to In progress as you start it, and back to To do if it is dropped. The work is every open issue in `doing` (In progress) and `todo` (To do), In progress first, then oldest first. Skip an issue assigned to someone other than the user, and say so.
3. Set up the working copy once: `git_clone` where there is none, with `branch` the first group's branch (step 2), else the checkout already here. Read the repository's `CLAUDE.md` or README for its check commands.
4. Read each issue (`GET /issues/{n}` and `/issues/{n}/comments`) before starting it. An issue whose `linked_pulls` holds an open pull request this run didn't open is someone's work in hand: skip it and say so. An issue that needs a decision only the user can make (a design choice, a secret, missing facts): post the question as a comment (`POST /issues/{n}/comments`), leave it in To do, and go on to the next one.

## 2. Fix each issue on its pull request's branch

Fixes go through pull requests, grouped by area. Nothing merges until every group is done (step 3). Done issues go to the archive a day after they close.

Group the run's issues before the first change: issues in the same subdomain, crate, app or files go in one pull request, and a lone issue can be its own. Each group has its own branch, `<kind>/<topic>` named for the group's change (`fix/git-board-issues`, `fix/mesh-security-hardening`), never for the session or one tool. Work the groups one at a time. Make each group's branch off an up-to-date default branch as you start it: `git fetch origin && git checkout -B <default> origin/<default> && git checkout -b <branch>`. Then, for each issue in the group:

1. Move it to In progress: `POST /issues/{n}/board` `{ "column": "doing" }`, so the board shows what is being worked on. Make the fix on the group's branch, with a test only where the repository's rules ask for one.
2. Run the repository's own checks for what changed. They must pass before the commit. An issue dropped without a commit (checks you can't make pass, out of time, or a fix found too risky to merge unasked): comment on the issue with why (what was tried, the failing output), move it back to To do (`{ "column": "todo" }`), drop its uncommitted change (`git restore --staged --worktree <files>`, and remove files it added), go on.
3. Commit, one commit for the issue, with a message whose body ends `Closes #{n}`, and push the branch (`git push -u origin <branch>`). Don't merge.

When the group's issues are done, open its pull request: `POST /pulls` `{ "title": "...", "body": "...", "head": "<branch>", "base": "<default>" }`. The title is a short plain sentence for the group's change. The body is plain text: one line on what changes, then one `Closes #{n}` line per issue fixed in it. That links each issue to the pull request, and the board shows it In progress while the pull request is open. Note the pull request's `number`. A group whose issues were all dropped gets no pull request: delete its branch (`git checkout <default> && git branch -D <branch>`, and `git push origin --delete <branch>` if it was pushed).

Issues that touch the same files go in the same group, one after another. A problem found outside the issue in hand is filed as `git.md` step 8 says, not fixed in that commit.

## 3. Merge, then delete stale branches

Once every group is done and each has its pull request open, merge them one after another: `POST /pulls/{n}/merge` `{ "delete_branch_after_merge": true }`. It returns the merge commit's `sha`. Merging closes the linked issues and moves them to Done. A pull request the forge refuses (a conflict, or its branch behind): on its branch merge the up-to-date default branch in (`git fetch origin && git checkout <branch> && git merge origin/<default>`), resolve any conflict, re-run the repository's checks, `git push origin <branch>`, and merge it again. Never force push.

Then `git fetch origin --prune && git checkout -B <default> origin/<default>`, and delete each local branch whose pull request merged (`git branch -D <branch>`). Then `GET /branches`. Delete each branch with `merged: true` (its tip is already in the default branch): `DELETE /branches/{name}`. The default branch is never `merged`. Leave every branch with `merged: false`, which still holds unmerged work, and list them in the report with their last commit.

## 4. Release

Only when something merged in this run. Find the release workflow: a file under `.phosphorus/workflows/` (or `.github/workflows/`) with `on: workflow_dispatch` (`release.yml` in phosphorus). None: say so and stop after step 3.

1. `POST /actions/workflows/{file}/dispatches` `{ "ref": "<default>" }`, inputs left at their defaults. It returns the run's `id`.
2. Follow it every minute or two with `GET /actions/runs/{id}` (its `status`) until it is `success`, `failure` or `cancelled`. `queued` for over ten minutes means no runner matches its `runs-on`: report that.
3. `failure`: `GET /actions/runs/{id}/logs` for each job's status and log, and read the failing step.
   - Fails for the code: commit the fix on a new small `fix/<topic>` branch off the default branch (no issue needed), run the checks, push it and open its own pull request as in step 2, merge it as in step 3 once its checks pass, dispatch again.
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

A failure the release caused (a panic, an error that started with the new version, a service that stopped answering): file an issue (`POST /issues` with the log lines), fix it on a new small `fix/<topic>` branch with its own pull request as in step 2, merge it as in step 3 once its checks pass, and go round steps 4 to 6 again. Never roll a host back to an older version; fix forward. A failure outside the release (a host off the tailnet, a disk full): report it, don't fix it in code. The same failure after two fixes: stop and report.

## Report

Lead with the result: the version now on the mesh, or where the run stopped and why. Then one line per pull request (number, branch, issues, merge commit), one line per issue (number, title, its pull request, Done or why not), the branches deleted and the ones left, each release run and its outcome, each host's version and anything the health check found, with the evidence lines verbatim.
