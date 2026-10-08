# StudyHub — repository notes

## What this is
A **static site**: `index.html` (app shell, inline CSS + JS, study questions, practice
tests, leaderboard UI) and `leaderboard.html` (standalone class leaderboard page).
No backend, no build step, no package manager, no framework, no dependencies.
All state lives in the browser's `localStorage` (`studyhubProgress`, `studyhubScores`,
`studyhubLeaderboard`, `studyhubUsername`).

`index.html` has one optional integration: `STUDYHUB_SHARED_LEADERBOARD_URL` (near the
top of its `<script>` block). It is an empty string, which disables the shared/remote
leaderboard — the app then uses the local per-device leaderboard only. Filling it in
would require an external JSON endpoint; nothing else in the repo talks to the network.

## Running it (Base44 sandbox)
```bash
docker compose -f docker-compose.base44.yml up -d
```
- `web` — `nginx:alpine` on host port **3000**, serving the repo bind-mounted read-only
  at `/usr/share/nginx/html`. Editing `index.html` / `leaderboard.html` shows up on the
  next page load (no rebuild, no restart); call `reload_preview` to refresh the iframe.
- `.base44/nginx.conf` is mounted at `/etc/nginx/nginx.conf`. It must set `user root;`
  because the sandbox bind-mounts the repo read-only with root-only permissions, and
  nginx's default unprivileged worker otherwise fails with `403 (13: Permission denied)`.
  It also denies dotfile paths so `.git/` and `.base44/` are not served publicly.
- Healthcheck probes `GET http://127.0.0.1/index.html` (explicit IPv4 — busybox wget
  tries `::1` for `localhost` and nginx listens on IPv4 only); verify with
  `docker compose -f docker-compose.base44.yml ps`.

## Known caveats
- `index.html` calls `ensureUsername()` at the end of its script, which opens a **native
  `prompt()`** when no username is stored. A real browser shows that dialog and the page
  renders normally once it is answered. An automated/headless browser cannot answer it,
  so it blocks the main thread and the page never becomes responsive — automated checks
  on `index.html` should expect that and seed `studyhubUsername` in `localStorage` first
  (there is no way to do that before the page's own script runs on a fresh load).
- `leaderboard.html` asks for the username with an in-page modal (`#usernameInput` +
  `#save`) instead, so it is fully automatable.
- Because storage starts empty, both leaderboards legitimately show their empty states.

## Verifying the app works
- `curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/` → `200`, and the page
  title is `StudyHub — High School Study Center`.
- `http://localhost:3000/leaderboard.html` → `200`.
- No credentials or environment variables are required.
