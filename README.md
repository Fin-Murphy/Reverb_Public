
# Visit the Reverb landing page at  https://getreverb.vercel.app/

# Reverb_ Demo: [https://youtu.be/SP142rXVtIM](https://youtu.be/SP142rXVtIM)

# Demo raw audio: [https://youtu.be/XIxWi8a-220](https://youtu.be/XIxWi8a-220)

This a public version of the REVERB repository. 
Since the Reverb codebase contains potential sensitive info, it is not available at this time

Some interesting technical bits & bobs:

### Splicing TTS audio at the byte level

The most interesting problem in the app. The TTS Worker takes roughly five seconds per email; a ten-newsletter brief sent as one request blows past the request timeout. So the brief is split into independently speakable segments — greeting, calendar block, each email, outro — synthesized **in parallel**, and the returned WAV blobs are spliced back into one file client-side.

`WAVConcatenator` (`Shared/Services/AudioBriefingService.swift`) does the splice without pulling in AVAssetExportSession — it operates directly on the RIFF container:

- **Generic chunk walking, not a fixed 44-byte header assumption.** It iterates `id`/`size` pairs from offset 12, honours the even-byte padding rule, and locates `fmt ` and `data` wherever they actually land.
- **Tolerates streaming headers.** The Worker emits WAVs with a `0xFFFFFFFF` placeholder in the `data` chunk size field, because a streaming encoder doesn't know the final length when it writes the header. Since `data` is always terminal in a WAV, the parser takes the remainder of the buffer whenever the declared size is implausible.
- **Derives silence from the format chunk.** An 0.8 s gap is inserted between segments so consecutive stories don't run together. The gap length in bytes is computed from the sample rate, channel count, and bit depth read out of `fmt ` — not hardcoded — so it stays correct if the Worker's voice model changes.
- **Rebuilds the RIFF header** with correct `data` and `RIFF` payload sizes for the merged stream.

The result: wall-clock synthesis time is bounded by the *slowest single segment* rather than the sum, and a partial failure is attributable to one story instead of the whole brief.

---

### Turning newsletter HTML into plaintext

Marketing email is the worst input format there is, and it has to be clean before it reaches an LLM. `GmailLoader` requests `format=full` and then:

1. **Recursively walks the MIME tree** (`findPart`), preferring `text/html` over `text/plain` — counterintuitive, but the plaintext alternative in marketing email is usually a stub that says "view this in your browser."
2. **Base64url-decodes** each part body, restoring `-`/`_` to `+`/`/` and re-padding to a multiple of four.
3. **Strips markup in layered passes**: `<script>`/`<style>` bodies removed wholesale, block-level tags converted to newlines so paragraph structure survives, remaining tags dropped.
4. **Decodes entities** — a named table plus a regex pass over numeric entities in both decimal and hex form, applied in reverse match order so earlier replacements don't invalidate later ranges.
5. **Normalizes whitespace** — collapse runs of spaces/tabs, cap consecutive newlines at two, trim per-line padding.

Sender display names get a second cleaning pass before they're spoken: URLs stripped, then a Unicode-scalar allowlist filter (letters, digits, whitespace, a small punctuation set) that removes the emoji and decorative glyphs newsletters put in their `From` headers — TTS reads those aloud by name.

Server-side filtering keeps the client's work small. Gmail's search syntax does the selection before any message is fetched:

```swift
"from:(a@x.com OR b@y.com) after:1754870400 -in:sent -in:drafts -in:trash -in:spam"
```

The ID listing pages through `nextPageToken` at 500/page with no cap; only the resulting IDs are hydrated in parallel.

---

### Authentication

**Google — incremental authorization.** Reverb never asks for Gmail and Calendar access at sign-in. Each integration requests its own scope at the moment the user taps *Connect*, which keeps the initial consent screen to basic profile only. `AuthenticationViewModel.connect(scope:)` picks the right mechanism based on current state:

```swift
if isSignedIn {
  authenticator.addScopes([scope], completion: completion)   // incremental grant
} else {
  authenticator.signIn(additionalScopes: [scope], ...)       // bundled — one consent screen, not two
}
```

Granted scopes are read back from `GIDGoogleUser.grantedScopes` rather than tracked in local state, so the UI reflects the actual server-side grant even after a revocation elsewhere. Views never touch `GIDSignIn` directly for authorization — everything routes through the view model.

**Nonce validation.** Sign-in generates a nonce, passes it into the authorization request, then decodes the returned ID token's payload segment and compares — per OpenID Connect Core §3.1.3.7 rule 11. Mismatches fail the sign-in. The same JWT segment decoder is reused to read `auth_time` for the "last authenticated" display.

No client secret ships in the binary; the iOS OAuth client ID in `Info.plist` is a public identifier and the code exchange runs through AppAuth's PKCE flow.

**Canvas — validate before persisting.** `CanvasAuthenticator` normalizes the host (scheme prefix and trailing slash stripped), probes `GET /api/v1/users/self` with the supplied token, and maps status codes to typed errors (`401/403` → invalid credentials, `404` → wrong host) with user-actionable messages. Credentials only reach the Keychain after a 200. Storage is a generic-password item under `kSecAttrAccessibleAfterFirstUnlock` — available to background refresh, but not before first device unlock.

---

### Current state

Wired end-to-end: Google sign-in with incremental scopes, Calendar and Gmail ingestion, summarization, parallel TTS with client-side splicing, briefing persistence and library playback, sender allowlist management, onboarding, and settings.

Scaffolded but not yet consumed by the briefing pipeline: `CanvasAuthenticator` / `CanvasAuthViewModel` / `KeychainHelper` complete the full connect-validate-store-disconnect loop and the Integrations UI reflects connection state, but no Canvas data feeds the brief yet. Study Mode, Record, and Streaks are routed and render placeholder screens.

---

Built on top of Google's `DaysUntilBirthday` GoogleSignIn sample; 
