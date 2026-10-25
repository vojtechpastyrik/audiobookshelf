# Patched fork of audiobookshelf

Tracks upstream [advplyr/audiobookshelf](https://github.com/advplyr/audiobookshelf) and
carries a small set of local fixes on the `patches` branch.

Everything here is temporary. A patch is dropped as soon as upstream ships the fix.

## Layout

- `patches` — the only branch that matters. Upstream release tag + patch commits + a tip
  commit carrying `.upstream-base`.
- `.upstream-base` — the upstream tag the patches currently sit on. Read by both workflows.
- `master` — untouched mirror of upstream.

## Patches

| commit | what | upstream |
|---|---|---|
| `fix(server): keep transcode streams resolvable after session close` | A backgrounded client keeps requesting HLS segments from the session it had open. Upstream tears the stream down immediately, so every request is a 404 and playback never resumes. Keeps the stream resolvable for `STREAM_GRACE_PERIOD_MS` (default 10m) after close, with the transcode stopped and ffmpeg never restarted for a lingering stream. | [#5612](https://github.com/advplyr/audiobookshelf/issues/5612) |
| `fix(server/managers/ApiCacheManager.js): resolve model name for afterUpsert` | Sequelize passes `afterUpsert` a `[instance, created]` tuple, so `getModelName` returned `'Array'`, missed `highChurnModels`, and cleared the whole API cache on every playback session upsert. | not reported |

## Runtime switches

| env | default | effect |
|---|---|---|
| `STREAM_GRACE_PERIOD_MS` | `600000` | How long a closed session's stream stays servable. `0` restores upstream behaviour without a rebuild. |

## Image

`ghcr.io/vojtechpastyrik/audiobookshelf:latest` — built from `patches`, `linux/amd64` only.

Also tagged `:<upstream-version>` and `:<upstream-version>-<run>` so a bad build can be
pinned back.

## Automation

- **Sync upstream** — daily 04:00 UTC. Finds the newest non-prerelease upstream tag,
  rebases `patches` onto it, amends the `.upstream-base` tip commit, force-pushes.
  On conflict it aborts and opens an issue; `latest` keeps pointing at the last good build.
- **Build image** — on every push to `patches`.

## Rebasing by hand

```sh
git fetch upstream --tags
git switch patches
git rebase --onto <new-tag> "$(cat .upstream-base)" patches
echo <new-tag> > .upstream-base
git commit --amend -am "chore: track upstream <new-tag>"
git push --force-with-lease origin patches
```

Turn on `git config rerere.enabled true` so a conflict resolved once is reapplied next time.
