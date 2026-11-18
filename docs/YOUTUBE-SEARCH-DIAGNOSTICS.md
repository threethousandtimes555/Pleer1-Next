# YouTube search and playback diagnostics

This document is the diagnostic contract for the Pleer1 companion. It records the known Build #14 evidence and the tests needed to investigate search without mixing it with stream playback.

## Product intent (not yet a claim about current implementation)

- An ordinary query such as `Concrete` should find standalone video tracks, not only playlists or Mixes.
- An explicit collection query such as `crystal castles playlist` should make collection results discoverable.
- Future UI may show tracks first and collections in separate sections, similar to YouTube. Do not implement or recommend a UI redesign as a prerequisite for fixing the current search defect.
- Classify items by YouTube renderer/type and IDs, not by title keywords alone.

## Evidence from physical iPhone Build #14 (2026-11-18)

Treat these as observations from the user's device, not as proof that the current Lab code has been built or tested.

### Query: `Concrete`

- InnerTube returned HTTP 200 with a response body of about 637,559 bytes.
- A diagnostic summary reported `videos=18, playlists=1, radios=0`.
- A nearby parse event reported `tracks=1`.
- HTML fallback returned HTTP 200 but did not expose `ytInitialData`.
- The old events did not have sufficient correlation IDs to prove that the summary and parse event describe exactly the same request.
- The user reported that the actual search experience was surfacing playlists/collections rather than the expected standalone tracks.

**Conclusion:** the old log does not establish why only one parsed track was reported, nor does the aggregate count prove that the standalone `Concrete` result was correctly surfaced. Do not claim this query is fixed.

### Query: `12 шагов до рая`

- InnerTube summary reported `videos=14, playlists=6, radios=0`.
- A parse event reported `tracks=6`.
- HTML fallback did not expose `ytInitialData`.

**Conclusion:** counts alone do not establish which six objects were displayed or whether the desired individual track was among them. Inspect typed result IDs and actual UI output in a correlated run.

### Query: `crystal castles playlist`

- InnerTube summary reported `videos=21, playlists=9, radios=0`.
- A parse event reported `tracks=9`.
- HTML fallback did not expose `ytInitialData`.
- The user selected the playlist `Crystal Castles PLAYLIST`, playlist ID `PLzDFYTz8KSyBUDaNaPOKVjomV2lCm5W4G`.
- The playlist loaded 31 tracks.
- First item: `CRYSTAL CASTLES - CHAR --official`, video ID `CnqzIYZR1GA`.
- Second item: `Crystal Castles - Concrete`, video ID `M7zxAI3GW2o`; the user selected it and attempted playback.

**Conclusion:** collection discovery and playlist expansion occurred on the device. This does not establish that ordinary search correctly returns standalone tracks.

### Query: `Kunteynir`

- InnerTube request ended with a network connection loss.
- HTML returned HTTP 200 but did not expose `ytInitialData`.
- No YouTube results were produced in that attempt.

**Conclusion:** this attempt cannot diagnose renderer classification or parser filtering. Repeat with a reliable connection and correlate all events by `searchID`.

## Playback failure observed in the same Build #14 session

- Stream resolution attempts reported visitorData/network connection failures.
- Player API requests returned HTTP 200 but reported `LOGIN_REQUIRED` for VISIONOS, ANDROID_VR and IOS clients, with a bot-verification/login message.
- No playable AVPlayer item/audio was established.
- The user reported that the VPN connection was malfunctioning during this test; in an earlier successful recovery, enabling a reliable VPN coincided with playback working.

**Conclusion:** VPN/network and YouTube anti-bot/player resolution are a separate playback investigation. Do not use these playback errors as evidence that search parsing or result classification is broken. The VPN correlation is meaningful but is not, by itself, proof of the exact cause.

## Source attribution guardrail

The track/title `Джем – я молодой вампир vamp` was confirmed by the user as a YouTube result, not Jamendo. The user has no Jamendo API key. A Jamendo-related symbol or code path in the project is not evidence that a specific result came from Jamendo. Attribute a source only when the result object or correlated diagnostics establish it.

## Required diagnostic fields for new runs

Every search attempt should have one stable `searchID` propagated through all relevant events:

- query and normalized query;
- InnerTube request start, HTTP status, response size, network error;
- raw candidate counts by renderer type;
- for each candidate: renderer/type, title, video ID or playlist/browse ID, and source endpoint (avoid logging private stream URLs or tokens);
- parser accepted/rejected counts with rejection reason;
- filtering counts and reason;
- final result count and IDs sent to the UI;
- HTML fallback status, whether `ytInitialData` was found, and parsed counts;
- UI result type/identifier actually displayed and selected.

Never log cookies, authorization headers, API tokens, visitor secrets, or private stream URLs.

## Minimal acceptance tests

1. Search `Concrete`: verify at least one genuine standalone `videoRenderer` is parsed and appears in the app's results; record its video ID and prove it is not a playlist/radio renderer.
2. Search `crystal castles playlist`: verify collection renderers are discoverable and the selected playlist ID is preserved.
3. Open a direct playlist URL and a direct RD Mix URL: verify each is recognized as its correct collection type.
4. Confirm a track whose title contains `джем` is not automatically classified as a Mix or Jam collection.
5. Keep playback validation separate: resolve and play a selected video under a known-good network/VPN condition; record visitorData/player API outcome and AVPlayer readiness independently.

For each test, report whether it ran in code inspection, automated test, CI simulator, or physical iPhone. Never imply that one test environment proves another.
