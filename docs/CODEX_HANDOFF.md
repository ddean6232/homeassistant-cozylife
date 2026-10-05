# Codex Handoff: CozyLife TCP Reliability Fix

Date: 2026-10-05

## Mission

Finish and validate the focused CozyLife local-TCP reliability fix, then prepare the change for upstream review. The Home Assistant installation is production, so do not make changes to the live HA system until the code has passed local tests and the operator explicitly approves a controlled deployment step.

## Repository and branch

- Working repository: `~/code/homeassistant-cozylife`
- Fork: `ddean6232/homeassistant-cozylife`
- Upstream: `polaralias/homeassistant-cozylife`
- Working branch: `fix/stateless-cozylife-tcp`
- Draft upstream PR: `polaralias/homeassistant-cozylife#19`
- Current implementation commit: `cad840d`

## User constraints

- Scope only the CozyLife integration.
- Do not change indicator-mode behavior; it is explicitly deferred.
- Do not modify other Home Assistant integrations, networking, router settings, DNS, or unrelated automations.
- Treat Home Assistant as production.
- Do not install the branch into production without a backup and explicit approval for that deployment step.
- Prefer a small, reviewable upstream PR over a broad rewrite.

## Problem being fixed

CozyLife devices can silently close idle local TCP connections after roughly 30–60 seconds. The current upstream client retains a socket between operations, so later Home Assistant polls can use a stale connection and mark a device unavailable even while the CozyLife app continues to control it successfully.

Archived PR #7 and commit `651b967` identified the same failure mode and used connect/send/receive/disconnect with retries. Do not copy that change blindly: the current upstream client has better framing, sequence-number correlation, and command acknowledgement handling that must be preserved.

## Current code change

`custom_components/cozylife/tcp_client.py` currently:

- adds bounded retry constants;
- returns success/failure from `_initSocket()`;
- closes the socket after each `_exchange()`;
- retries failed exchanges on a fresh connection;
- preserves the current `_FRAME_TERMINATOR` framing;
- preserves top-level sequence-number matching;
- preserves `CMD_SET` acknowledgement validation through `res == 0`.

`tests/test_tcp_client_contract.py` adds coverage for:

- cleanup after an exchange;
- retrying after a stale connection.

## Required next work

1. Review the current diff for correctness and keep the change limited to the TCP client and its tests.
2. Add or improve tests if needed for:
   - failed connection establishment;
   - failed send followed by reconnect;
   - failed response followed by reconnect;
   - negative command acknowledgement;
   - fragmented responses;
   - unrelated sequence numbers;
   - ensuring no idle socket remains after success or failure.
3. Run the repository test command with the CI dependency versions:

   ```bash
   uv run --prerelease=allow \
     --with 'homeassistant==2025.1.4' \
     --with 'pytest==8.3.4' \
     --with 'pytest-asyncio==0.24.0' \
     python -m pytest -q
   ```

4. Run `git diff --check`.
5. Review the final diff manually.
6. Do not deploy to HA yet. Prepare a reversible production-test plan instead:
   - identify the installed CozyLife integration files/version;
   - back up only `custom_components/cozylife`;
   - install only the tested CozyLife files;
   - restart HA only with approval;
   - monitor both switches through polling and an idle period;
   - roll back the CozyLife backup if required.
7. Keep PR #19 as a draft until local and live validation evidence is available. Do not claim the issue is fixed based on unit tests alone.

## Definition of done for this handoff

- The integration-only change is tested locally.
- No indicator-mode code is changed.
- No production Home Assistant files have been changed without explicit approval.
- The branch is clean except for deliberate committed work.
- PR #19 contains an accurate summary, test evidence, and clearly labels live validation status.

## Important evidence

The upstream repository currently treats switches as potentially supported and has not validated switch reliability on real hardware. The app working normally does not disprove this bug because the app and the Home Assistant integration can use different connection/session paths.
