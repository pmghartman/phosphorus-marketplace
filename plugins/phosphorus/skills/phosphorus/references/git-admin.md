# Git hosting and mirrors

Read `git.md` first; `{r}` is `/repos/{owner}/{repo}`.

- Push mirrors (a repository pushed to GitHub, GitLab, Codeberg or another forge): `git_read` `{r}/push_mirrors` lists each with its last push (`status.ok`, `status.log`). Only when the user asks: `git` `POST {r}/push_mirrors/sync` pushes all now, `{r}/push_mirrors/{name}/sync` one, returning each result; `POST`/`PATCH {r}/push_mirrors` adds or changes one with the details the user gives. Tokens are never returned.
- Actions: workflows in `.phosphorus/workflows/` or `.github/workflows/` run on hosts enabled as runners (Git › Administration › Hosts). A run stuck in `queued` means no runner matches its `runs-on`.
- `git_server`, only when the user asks: with no arguments, the hosts with a copy of the git database (`dbs`: up, order, `lead`), the site, the runners and `storage` (each copy and runner host's `appdata/git` by part, and `unused`: what the hourly sweep frees; never measure it with `du` or sqlite over `terminal_run`). Writes go to every copy and the most current one leads. `db` with `on` adds or removes a copy (not the last); `site` assigns `git.<domain>`; `runner` with `on` enables a runner.
- Forge users: `git_read` `/admin/users` lists them (admin, active, created, repositories owned); `git` `DELETE /admin/users/{name}` removes one gone from the mesh who owns no repositories, only when the user asks (409 with the reason otherwise).
