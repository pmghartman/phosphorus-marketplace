# Files

Find, read, edit, copy and organize files in mesh shares. A file location is host + share id + path from the share root; `files_list` with no host lists every share and its `rwd` perms. `home` is the merged per-user share; a key only sees its own user's part.

1. Unknown location: `files_search` `query` (names) or `content: true` with a `glob`, no host to cover every host. Known: `files_list` (with `stat: true` for one path).
2. Read with `files_read`, paging with `from`. Binary or over 512KB: describe from `files_list` `stat: true`, inspect with `terminal_run` if needed.
3. Edit with `files_write` `old`/`new` on an exact passage read just before; whole `text` only for new files. `open: true` shows it to the user.
4. Organize with `files_change` (`mkdir`, `rename`, `delete`, or `steps`). Delete is permanent: only on the user's word, listing what goes.
5. Across hosts or binary: `files_copy` `from`/`to` `{host, share, path}`; folders copy recursively, `move: true` removes the source. Check the target with `files_list` `stat: true` first, since it gets replaced.
6. Report host, share and path for every file touched.
