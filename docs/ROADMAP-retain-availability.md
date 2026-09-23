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

**Stage 0: fork + roadmap in the repo.** `gh repo fork wez/govee2mqtt --clone` to `/config/tools/govee2mqtt`
(outside `custom_components`, outside HA's git: I'll add it to `.gitignore` in `/config`). Remote `upstream` = wez.
Write `docs/ROADMAP-retain-availability.md` in the fork (this plan: root cause, decisions, stages) and commit
it to the fork's main. *Checkpoint:* the fork exists and the roadmap is in it.

**Stage 1: the patch on a clean branch (for upstream).** Branch `fix/retain-availability` from `upstream/main`,
containing only `src/service/hass.rs`:
- a `publish_retained` helper (or a `retain` parameter) used only for `availability_topic()`/`"online"` in `register_with_hass`
- `client.set_last_will(availability_topic(), "offline", QoS::AtMostOnce, true)`
- a code comment explaining why (the race with HA subscribing)

There's no local Rust (Alpine/musl, and we don't compile locally). Compile + tests run in the fork's CI:
push the branch → PR *inside the fork* (branch → miczu71:main) → `pr.yml` + `build.yml` (build without push)
must be green. *Checkpoint:* diff + green CI.

**Stage 2: fork identity + release.** On the fork's `main`: merge `fix/retain-availability`, commit the identity
changes (the list above), then push main → CI publishes `ghcr.io/miczu71/govee2mqtt:latest`. Bump
`addon/config.yaml` version = `2026.MM.DD-<sha8>-miczu71` → **published** GH release with the same tag and full
release notes → the tag job builds `ghcr.io/miczu71/govee2mqtt-{amd64,aarch64}:<version>`.
**Manual step for you:** GHCR creates new packages as *private* and there's no API to change that. In GitHub →
Packages, set `govee2mqtt`, `govee2mqtt-amd64` and `govee2mqtt-aarch64` to **Public** so Supervisor can pull them.
*Checkpoint:* the image tags exist and are public (`docker manifest`/`gh api` check).

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
