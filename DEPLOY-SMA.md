# DEPLOY-SMA — LibreTime tester-env

Baseline: `main @ 97290cf3a340e712d70843b235cb9ca03773970e` (2026-11-22).
Release 4.5.0 was skipped: 16 months old with EOL `php:7.4-fpm` / Debian
bullseye builders, while this SHA built all six bake targets in upstream's
Container workflow (run 44971708119).

## Quick start

```bash
./tester-env deploy   # build api/legacy/nginx from this checkout, start stack, migrate
./tester-env seed     # deterministic Riverbend Community Radio data set
./tester-env verify   # assert seeded rows, media files and legacy UI endpoints
./tester-env status   # compose state, HTTP probes, URL and credentials
./tester-env logs api # follow one service (postgres|rabbitmq|api|legacy|nginx)
./tester-env stop     # stop, keep volumes
./tester-env reset    # down -v: drop containers + postgres_data/rabbitmq_data/libretime_storage
```

Reset plus seed is the reproducible cycle:

```bash
./tester-env reset && ./tester-env deploy && ./tester-env seed && ./tester-env verify
```

`seed` clears the seeded tables first, so it is also re-runnable in place
(measured 3 s; reset + deploy + seed + verify 45 s with cached images).

Login: `admin` / `admin` (inserted by `libretime-api migrate` from
`api/libretime_api/legacy/migrations/sql/data.sql`). `seed` adds `dana` /
`dana` (Program Manager) and `miles` / `miles` (DJ).

## Seed data

Theme: a small volunteer-run community station. Implementation:
`seed/libretime-seed.sql` (rows) plus a media step in `./tester-env seed` that
writes one file per track under `/srv/libretime/seed/` in the
`libretime_storage` volume.

| Entity | Stable counts / names |
| --- | --- |
| Tracks | 27 files, 8 genres (Jazz 5, Soul 3, Indie Rock 4, Folk 4, Electronic 3, Blues 3, Ambient 2, Station ID/Bed 3), 10 artists, 2017-2026 |
| Track edge cases | `Cardboard Moon` has no album; `Sunday Morning Rain (Live at the Harbor)` is the long title; 3 station IDs are 0:41 |
| Playlists | `Morning Drive` 21:46 (6 tracks), `Late Night Jazz` 20:24 (5), `Weekend Folk Hour` 15:38 (4), `Station IDs and Beds` 2:03 (3) |
| Blocks | `Jazz Rotation` dynamic, genre contains Jazz, newest first, 30-minute limit; `Top of Hour Beds` static 1:22 |
| Shows | `Sunrise Broadcast` Mon 06:00-09:00, `After Hours Jazz` Fri 22:00-23:59, `Saturday Folk Session` Sat 19:00-21:00 (UTC), each linked to its autoplaylist |
| Schedule | 6 instances (current and next week) and 30 scheduled items |
| Webstreams | `Riverbend Live Studio Feed`, `World Service Overnight` |
| Users | admin (Admin), dana (Program Manager), miles (DJ) |

Determinism notes:

- Track metadata, upload times (`cc_files.utime`), playlist lengths and all
  names are fixed in the SQL; `verify` asserts the counts and lengths.
- Media files are N concatenated copies of the pinned upstream fixture
  `api/libretime_api/_fixtures/song.mp3` (40.81 s, 128 kbps, 44.1 kHz);
  `cc_files.filesize = N * 654105` and declared lengths are `N * 40.81 s`
  rounded to the second, so file duration and displayed length agree.
- Show instances are inserted for the current and next week, anchored to the
  Monday of the week in which `seed` runs (the calendar's default week/month
  view is wall-clock relative). Day-of-week, start times, durations and names
  are stable; only the calendar date follows the seed week.
- Instances are inserted explicitly because the legacy generator
  (`Show::createAndFillShowInstancesPastPopulatedUntilDate`) commits through a
  persistent Propel connection that the calendar leaves inside an open
  transaction (`ShowInstance::getShowHasAutoplaylist` begins one and never
  commits), so generated instances are rolled back on connection reuse.
  `shows_populated_until` is set three weeks out so the calendar does not try to
  generate more for the seeded range.
- `lang_tz_setup_complete` is set so the first-login language/timezone popup
  does not cover the UI.
- Reset drops the postgres and storage volumes, so anything created in the UI
  (for example the `Smoke Test Playlist` from the browser smoke, or an
  `Untitled Playlist`) disappears; only the seeded state comes back.

## Verify checks

`./tester-env verify` runs 40 checks: seeded row counts and relationships,
playlist/block lengths, media files and byte totals in the storage volume, and
HTTP calls against the endpoints the legacy UI itself uses (admin login,
`/library/contents-feed` for the four library tabs, `/playlist/edit` for
planner and smart-block detail, `/user/get-user-data-table-info`,
`/schedule/event-feed` for the Radio Calendar, `/showbuilder/builder-feed` for
scheduled show content).

## Browser-visible surfaces

- `/showbuilder` (post-login landing page): left pane lists the 4 playlists, 2
  smart blocks and 2 webstreams; the `Tracks` tab lists 27 tracks with artist,
  album, genre, length and BPM. The right-hand schedule table only lists
  content for the default `now → +3h` window, so it can be empty when no show
  is on air.
- `/schedule` (Radio Calendar): the current week shows `Sunrise Broadcast`,
  `After Hours Jazz` and `Saturday Folk Session`, with filled-percentage
  badges; next week repeats them.
- Library tabs: Tracks 27, Playlists 4, Smart Blocks 2, Webstreams 2.
- Playlist planner: opening `Morning Drive` shows its 6 tracks with a 21:46
  total; `Jazz Rotation` shows the 30-minute dynamic block with its Genre
  criteria.
- Settings > Users: 3 rows (Admin, Program Manager, DJ).

## Stack

`docker-compose.tester-env.yml` builds three targets of the upstream root
`Dockerfile` and runs five services:

| Service | Image | Notes |
| --- | --- | --- |
| postgres | `postgres:17` | named volume `postgres_data` |
| rabbitmq | `rabbitmq:4.4-alpine` | named volume `rabbitmq_data` |
| api | `${IMAGE_TAG}-api` | gunicorn ASGI `:9001`, `/srv/libretime` storage |
| legacy | `${IMAGE_TAG}-legacy` | php-fpm `:9000` |
| nginx | `${IMAGE_TAG}-nginx` | publishes `$PORT` (default 18650) → `:8080` |

Omitted on purpose: `playout`, `liquidsoap`, `icecast`, `analyzer`, `worker`.
Stream/Live/Listener pages and async import status will show offline or pending
states; keep scenario assertions on library, playlist, schedule,
podcast/webstream, settings and users surfaces.

## Env / ports / state

- `PORT` (default `18650`) — the only host port.
- `RUN_ID` — suffix for the compose project, containers and volumes.
- `IMAGE_TAG` — image tag base, `-api`/`-legacy`/`-nginx` suffixes
  (default `tester-env-libretime:dev`); never contains `RUN_ID`.
- `TESTER_ENV_BROWSER_ORIGIN` — written into `tester-env.config.yml` as
  `general.public_url`. The PHP legacy app ignores `LIBRETIME_*` env overrides,
  so this file is regenerated by `deploy`/`start`; without the variable it
  falls back to `http://localhost:$PORT`.
- `tester-env.config.yml` is generated (api_key, secret_key, DB and RabbitMQ
  creds pinned). Do not edit it by hand.

## Mutation path

- `legacy/public/js` is bind-mounted read-only into nginx: JS edits are visible
  immediately with no rebuild.
- `legacy/application` is bind-mounted read-only into php-fpm: PHP/controller/
  view edits are visible after the nginx fastcgi cache window (5 s).
- `api/`, `shared/`, `worker/` code is baked into the image, so python-side
  mutations need a `${IMAGE_TAG}-api` rebuild.

Measured on this host: first full `deploy` 4 min 20 s (all three targets, 48
migrations); reset + deploy with cached images 41 s; stop + start 34 s.

## Caveats

- `public_url` must match the URL the browser actually uses; pass
  `TESTER_ENV_BROWSER_ORIGIN` when deploying for a browser session. On this host
  the browser container reaches the app at `http://172.16.0.1:<PORT>/`.
- RabbitMQ and postgres publish no host ports; everything but nginx stays on
  the compose network.
- `/` and `/login` return the Zend login page; successful login redirects to
  `/showbuilder`.
