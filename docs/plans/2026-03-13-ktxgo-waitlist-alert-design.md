# KTXgo Waitlist Alert Registration Design

## Goal

Add a follow-up step after KTX waitlist registration so `ktxgo` also registers seat-assignment alerts for the same booking, including recipient phone number and available alert channel choices.

## Current State

- Login happens on `www.korail.com` through a real browser session.
- After login, `ktxgo` calls Korail mobile-style endpoints from the browser context with `fetch(..., credentials: "include")`.
- Waitlist booking already exists and is implemented as `TicketReservation` with `txtJobId=1102`.
- The current flow stops after waitlist success and does not perform any alert-registration follow-up call.
- Official Korail guidance describes seat-assignment alert registration as a separate action from reservation waitlisting, exposed from reservation history.

Relevant code:

- [ktxgo/browser.py](/home/chan/archive/ktx-srtgo/ktxgo/browser.py)
- [ktxgo/korail.py](/home/chan/archive/ktx-srtgo/ktxgo/korail.py)
- [ktxgo/cli.py](/home/chan/archive/ktx-srtgo/ktxgo/cli.py)

## User-Facing Requirement

When `ktxgo` successfully submits a waitlist request, it should also register seat-assignment alerts so the user receives the Korail notification when a seat is assigned. The configuration must cover:

- alert channel: SMS, KakaoTalk, or both if the upstream flow supports it
- destination phone number
- clear reporting when waitlist registration succeeds but alert registration fails

## Constraints

- The waitlist alert action is not currently documented as part of the existing reservation call.
- Official public guidance confirms the feature exists, but not the exact request contract.
- `ktxgo` currently does not store or prompt for a phone number dedicated to waitlist alerts.
- The alert flow may reuse the existing `www.korail.com` browser session, or it may require a mobile-domain KorailTalk session.

## Approaches Considered

### 1. Add a same-session follow-up API call

After `TicketReservation(1102)` succeeds, call a second endpoint from the existing Playwright browser session with the reservation number, alert channel, and phone number.

Pros:

- Fits the current architecture cleanly.
- Reuses `_api_call()` and existing authenticated browser state.
- Keeps the feature in the same `KorailAPI` abstraction as reservation and payment.

Cons:

- Depends on discovering the exact endpoint and parameters.
- May fail if the action is restricted to a different origin or session type.

### 2. Add a mobile-domain helper client for alert registration

Keep current waitlist booking logic as-is, but use a second KorailTalk-oriented client for the alert-registration follow-up step when the booking succeeds.

Pros:

- Good fallback if the alert endpoint requires `smart.letskorail.com` semantics or a different login flow.
- Keeps browser-session logic isolated from mobile-session logic.

Cons:

- Requires separate session management.
- Increases implementation complexity and maintenance cost.

### 3. Automate the reservation-history UI

Use Playwright to navigate to reservation history and click through the alert-registration form after waitlist success.

Pros:

- Can work even if the request contract remains unclear.

Cons:

- Most fragile option.
- Higher maintenance burden because DOM changes can break the feature.
- Slower and less suitable for headless automation.

## Recommended Design

Use a layered strategy:

1. Reverse-engineer the real seat-assignment alert request from Korail's reservation-history flow.
2. Implement a first-class `KorailAPI.set_waitlist_alert(...)` follow-up call that runs immediately after waitlist success.
3. Reuse the current browser session if the discovered endpoint works from that context.
4. If the endpoint requires a mobile-domain session, add a minimal fallback mobile client only for the alert-registration step.
5. Keep UI automation out of the product unless both API-based paths are blocked.

## Proposed Flow

1. User runs the normal `ktxgo` reservation command.
2. Search loop finds a sold-out train with waitlist availability.
3. `KorailAPI.reserve(..., waitlist=True)` succeeds and returns `h_pnr_no`.
4. `ktxgo` resolves alert settings:
   - phone number from CLI option first, then keyring default
   - alert channel from CLI option first, then keyring default
5. `ktxgo` attempts `set_waitlist_alert(pnr_no, channel, phone)`.
6. The CLI reports one of three outcomes:
   - waitlist success + alert registration success
   - waitlist success + alert registration skipped because settings are missing
   - waitlist success + alert registration failed
7. Telegram notification, if enabled, includes the alert-registration result.

## Configuration Design

Add explicit waitlist-alert settings rather than overloading existing payment or Telegram configuration.

Planned inputs:

- CLI option for alert phone number
- CLI option for alert channel
- keyring-backed defaults for both values

Resolution order:

1. CLI arguments
2. keyring saved values
3. if still missing, skip alert registration with a visible warning

This avoids breaking existing flows while still allowing persistent setup.

## Error Handling

- Waitlist booking success must not be rolled back if alert registration fails.
- Missing alert configuration should not abort the reservation loop after booking succeeds.
- Upstream failures should surface the exact Korail error message when possible.
- If the same-session call fails because the endpoint is unavailable or rejected by session/origin policy, that should be classified separately from validation failures such as an invalid phone number.

## Testing Strategy

Unit/integration coverage should focus on local logic we control:

- follow-up call is attempted only for successful waitlist bookings
- alert settings resolution order is correct
- partial-failure reporting is correct
- Telegram payload reflects alert outcome
- API parameter builder produces the exact discovered contract

Live network verification will still be required once the real Korail alert request is captured.

## Unknowns To Resolve Before Implementation

- exact endpoint name for waitlist alert registration
- exact parameter names and accepted values for:
  - phone number
  - SMS enablement
  - KakaoTalk enablement
  - reservation number / change number / additional identifiers
- whether the request works from the existing `www.korail.com` browser session
- whether KakaoTalk is actually available through the same flow as SMS or only in the native app

## Decision

Proceed with API-first implementation. The first implementation milestone is to capture the real Korail request contract, then wire it into `ktxgo` as a post-waitlist follow-up action with non-fatal failure handling.
