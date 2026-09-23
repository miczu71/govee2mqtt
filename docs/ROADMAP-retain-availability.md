# govee2mqtt: fix at the source (fork + upstream PR) — retained availability

## Context

After an HA Core restart, `light.lampa_kuchnia` (and all 13 entities of the Govee H6076) stays `unavailable`
until the Govee to MQTT Bridge app (`b9845f46_govee2mqtt`) is restarted by hand. Last time: 2026-09-23 15:48.

### Root cause (confirmed with evidence)

In `wez/govee2mqtt` `src/service/hass.rs`, **every** publish is QoS 0 with `retain=false`: the discovery
configs, `gv2mqtt/availability=online`, the state, and the last-will `offline`. `mosquitto_sub --retained-only`
on `gv2mqtt/#` returns nothing. After a restart, HA only gets the one burst govee sends 15 s after the HA birth
message: configs → `sleep(10 ms × 15 = 150 ms)` → `online` → state.

Today HA was loaded during startup ("No ACK from MQTT server in 10 seconds" ×3). HA created the entities from
the 15:53:09 configs, but it subscribed to `gv2mqtt/availability` **after** govee had published `online`.
**The evidence is the MQTT debug info for the device:** all 13 entities were subscribed (they received the
LWT `offline` at 15:53:41), but none has an `online` message from 15:53:09. The history holds only 2 messages
out of a limit of 10. A non-retained `online` never arrives again, so the entities stay unavailable. The
upstream issues are still open: #110 and #572. There is a related open PR, #711 (debounce), but it doesn't fix this.

### Decisions (from the interview)

- **Fix at the source**, not with an HA automation: fork `miczu71/govee2mqtt` + a PR to upstream.
- **Patch A: retain only the availability.** `online` retain=true, last-will `offline` retain=true. HA gets the
  retained `online` right after subscribing. The state catches up by itself within ~30 s (periodic publish,
  seen in the debug info). Configs and state stay non-retained, which keeps the diff minimal for upstream.
- **Image on GHCR under miczu71**, built by GitHub Actions with `GITHUB_TOKEN`, the same way upstream does it.
- **Fork 1:1** (same layout, only the identity changes), so syncing is just `git merge upstream/main`.
- **Version suffix.** ⚠ Correcting my earlier suggestion: `+miczu71` **won't work**, because the version is the
  Docker image tag and a tag can't contain `+`. It will be **`YYYY.MM.DD-<sha8>-miczu71`** instead.

## Fork's technical details (verified in the upstream repo)

- `addon/Dockerfile`: `FROM ghcr.io/wez/govee2mqtt:latest`. **This has to point to miczu71**, otherwise the
  add-on image would pull in upstream's unpatched binary. It's the most important line of the whole fork.
- `.github/workflows/build.yml`: `IMAGE: ghcr.io/wez/govee2mqtt` → `ghcr.io/miczu71/govee2mqtt`. The workflow
  builds the binary (cross, amd64+arm64) on push to main, and the add-on (`home-assistant/builder`) on a `20*` tag push.
- `addon/config.yaml`: `image: ghcr.io/wez/govee2mqtt-{arch}` → miczu71, plus `url` and `name` ("… (miczu71)").
  `slug` stays `govee2mqtt`. Supervisor prefixes it with a hash of the repo URL, so there's no collision.
- `addon/build.yaml`: cosign `identity` + `org.opencontainers.image.source` → miczu71.
- `repository.yaml`: name/url/maintainer → miczu71.
- `scripts/apply-tag.sh`: CI builds the add-on version from the commit (`%cd-%h`). In the fork, `TAG_NAME` =
  the name of the pushed tag (`$GITHUB_REF_NAME`), so **`addon/config.yaml` version == the release tag**, as
  your release checklist requires.
- Host: x86_64 → `amd64`. `gh`: logged in as miczu71 with `repo`+`workflow`. The fork doesn't exist yet.

## Stages (each ends with a checkpoint; I won't start the next one without a go-ahead)

**Stage 0: fork + roadmap in the repo.** ✅ Done. Forked to `miczu71/govee2mqtt`, cloned to
`/data/home/dev/govee2mqtt` (deviation from the original plan's `/config/tools/govee2mqtt`: this is a
whole Rust project, not HA config, so it lives outside `/config` entirely rather than being gitignored
inside it). `upstream` remote = wez. This roadmap committed to the fork's `main`.

**Stage 1: the patch on a clean branch (for upstream).** ✅ Done. Branch `fix/retain-availability` off
`upstream/main`, `src/service/hass.rs` only: a `publish_retained` helper used for the `online` publish in
`register_with_hass`, and `set_last_will(..., true)`. PR #1 opened *inside the fork* (branch → miczu71:main).

CI result: `pr.yml` (`cargo build/test/fmt --check`) green. `build.yml`'s `build (linux/amd64)` and
`build (linux/arm64)` (cross-compile of the actual patched binary) green on 2 consecutive runs.
`test-addon` failed both times with a cosign "no signatures found" error validating HA's own
`ghcr.io/home-assistant/{amd64,aarch64}-base-debian:bookworm` images — confirmed pre-existing and
unrelated: upstream's own PR CI runs have been failing/`action_required` the same way since mid-August
(checked via `wez/govee2mqtt` Actions history), and this branch never touches `addon/`. Also note:
`test-addon` doesn't run on a tag push anyway (`if: ! (push && tag)`), so it won't block Stage 2's release.
**Checkpoint accepted** on the two build jobs. Squash-merged as PR #1.

**One-time manual step surfaced here:** GitHub disables Actions on a freshly forked repo behind a
browser-only consent banner ("workflows aren't being run on this forked repository") that isn't exposed
via the REST API or `gh` CLI at all — 0 workflow runs, even after a push/PR, until you click it once on
https://github.com/miczu71/govee2mqtt/actions. Done by you mid-Stage-1.

**Important finding — upstream's stance on retain (affects Stage 5 only, not Stages 0-4):** the maintainer
(wez) has twice rejected retain=true fixes on principle — closed PR #452 and PR #581 (a near-identical
patch to ours) with: *"Retained messages should not be required at all, as home assistant is supposed to
broadcast a message when it (re)starts... If that is not functioning correctly, I'd rather see some effort
and a PR that runs that down and resolves that, than turning on retained messages."* He then favored a
different approach in open PR #711 (debounce the birth/will re-registration handler so overlapping
`online`/`offline` triggers don't race each other's publishes) — a different race from the one we found
(subscribe-after-publish ordering, evidenced via HA's MQTT debug info showing entities subscribed but
missing the `online` message from the registration burst) and one #711's debounce doesn't fix. Decision
(with the user): **submit the PR anyway with our concrete evidence** — worst case it's closed again like
#452/#581, at no cost to us; our fork keeps the fix regardless of upstream's decision.

**Stage 2: fork identity + release.** ✅ Done. Identity commit (Dockerfile ×2/build.yml/config.yaml/
build.yaml/repository.yaml/README/docker-compose.yml → miczu71) pushed to `main`, publishing
`ghcr.io/miczu71/govee2mqtt:latest` (binary image). Deviations from the original plan, in order hit:

- **GHCR packages turned out public by default** this time (anonymous-token manifest fetch returned 200
  right away) — the plan's "manual step: set packages Public in GitHub UI" wasn't needed.
- **The `addon` release job failed** with `Error: no signatures found` / `Invalid base image
  ghcr.io/home-assistant/{amd64,aarch64}-base-debian:bookworm` — `home-assistant/builder`'s `--cosign`
  flag also verifies HA's own base-image signature, and that verification is currently broken for
  everyone (confirmed: `wez/govee2mqtt`'s own Container Build runs have been failing/`action_required`
  the same way since mid-August, before this fork existed). Fixed by adding `--no-cosign-verify` (added
  upstream in `home-assistant/builder@2025.09.0`, present in our pinned `2026.02.1`) to skip *only* the
  base-image check; `--cosign` stays on so our own output image is still signed. Flagged by the safety
  classifier as disabling verification (reasonable, out of context) — surfaced to the user, approved,
  applied.
- **`TAG_NAME: ${{ github.ref_name }}` didn't get main's `addon/config.yaml` version updated** — it only
  patches the file inside the ephemeral tag-triggered checkout that `home-assistant/builder` reads from,
  never commits back. Supervisor reads `main`'s `config.yaml` when it adds the repo, so that has to carry
  the real version too. Fixed by following upstream's own pattern (their `9158353 "Tag
  2026.03.25-ab9deb66"` commit): bump `version:` on `main` first, tag with the identical string, retag
  after the cosign fix. Landed on **`2026.09.23-miczu71`** (dropped the `-<sha8>` — date+suffix is
  distinct enough from upstream's own version strings for a single low-frequency fork, and sidesteps the
  chicken-and-egg of a commit needing to know its own future short hash).

Final state, verified: `ghcr.io/miczu71/govee2mqtt-{amd64,aarch64}:2026.09.23-miczu71` both public and
pullable (anonymous GHCR token, HTTP 200 on the manifest); `main`'s `addon/config.yaml` version matches;
[GitHub release](https://github.com/miczu71/govee2mqtt/releases/tag/2026.09.23-miczu71) published (not
draft). *Checkpoint met.*

**Stage 3: switching the app (with a rollback option).** Through the MCP: add the repository
`https://github.com/miczu71/govee2mqtt` to Supervisor → install the fork app → copy the options from the current
app (API key, MQTT host/port/user/pass, temperature_scale) → **stop the upstream app and set boot=manual (don't
uninstall it: that's the rollback)** → start the fork. Tests:
1. `light.lampa_kuchnia` is available and the lamp responds (on/off)
2. `mosquitto_sub --retained-only gv2mqtt/availability` → `online` (retained)
3. stop the fork app → the retained value changes to `offline` and the entities go `unavailable` → start → `online`

*Checkpoint:* all 3 tests pass. Rollback = stop the fork, start upstream.

**Stage 4: real-world test: an HA restart (done by you, I don't restart HA).** After the restart, without any
manual step: the lamp is available within ~1 min of startup, the MQTT debug info shows a retained `online`
received during subscription, and the state catches up within ~30 s. If it holds up, and with your approval,
uninstall the upstream app. *Checkpoint.*

**Stage 5: PR to upstream + documentation.** A PR `miczu71:fix/retain-availability` → `wez:main` with the evidence
(timeline, debug info, the 1-line mechanism) and references to #110/#572/#711. In `/config`: a note in CLAUDE.md
(govee2mqtt comes from the fork, why, and how to sync with upstream), and a memory note about the race and
how to diagnose it through the MQTT debug info. When upstream merges and releases the fix → go back to the
upstream app and archive the fork.

## Risks

- **A retained `online` that goes stale.** If govee dies without sending its LWT (the broker was down at that
  moment), HA would show `available`. That's rare, and stage 3 test 3 checks the normal path.
- **The switch creates a new slug and `addon_config`.** govee will build its cache from scratch (a single query to
  the Govee API). The entities keep their `unique_id` `gv2mqtt-…`, so entity_id/automations/scenes don't change.
- **Two apps at once would conflict on the MQTT topics**, so the upstream one is always stopped before the fork starts.
- Nothing in HA Core is restarted by me. The only live operation is the app switch in stage 3, with rollback.

## Verification (summary)

- Stage 1: green CI in the fork (compile + tests).
- Stage 3: retained `online`/`offline` on the broker + the lamp works.
- Stage 4: after a real HA restart, the lamp is available with no manual steps (this is the actual acceptance test).
