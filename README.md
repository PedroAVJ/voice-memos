# voice-memos

Read locally materialized Apple Voice Memos by stable UUID, maintain a private
on-demand transcript cache, and run independent source-owned semantic skills.
Before each store read, the plugin launches Voice Memos hidden in the background
with `open -gj` to prompt iCloud and Apple Watch synchronization. It never brings
the app to the foreground or mutates recordings.

## On-demand transcription

The worker prefers a usable transcript embedded by Apple and otherwise invokes
the installed ElevenLabs CLI with `scribe_v2` text output. It keys state by
`ZUNIQUEID`, writes transcript artifacts atomically with SHA-256 provenance,
uses per-item locks and bounded retry state, and never stores secrets in its
cache.

```bash
voice-memos transcriptions baseline --json
voice-memos transcriptions reconcile --limit 10 --json
voice-memos transcriptions status --state completed --limit 100 --json
voice-memos transcriptions show UUID --json
voice-memos transcriptions retry UUID --json
```

Version 0.7.13 restores the bounded hidden-app sync nudge on each store read
while keeping the former transcription LaunchAgent retired. Existing private
cache rows and transcript artifacts remain intact; no process watches or polls
Voice Memos continuously. `retry UUID` is the preferred exact-item path; run
bounded `reconcile` only when a wider manual scan is explicitly requested.

## Scheduled skills

Two skills independently select recordings by their capture time:

- `voice-memos:answer-captured-questions`
- `voice-memos:discuss-health-observations`

Their native schedules contain only the exact skill invocation. An omitted
window defaults to recordings captured during the previous 24 hours. Explicit
windows support replay or wider review, and offset paging remains stateless:

```bash
voice-memos questions scan --json
voice-memos questions scan --since 48h --json

voice-memos health-observations scan --json
voice-memos health-observations scan --since 2026-08-10T12:00:00Z --until 2026-08-12T12:00:00Z --json
```

Selection uses `recorded_at`, never transcript completion time. For a recording
inside the window, a semantic scan reuses a verified cached transcript or
invokes the source-local worker if the transcript is not ready. The same window
may be revisited without a processed cursor.

## Direct reads

```bash
voice-memos list --json
voice-memos path UUID
voice-memos transcript UUID --json
```

Requirements: macOS, Python 3, Full Disk Access for the invoking host, and the
installed ElevenLabs CLI with its API credential in macOS Keychain when Apple
has no usable embedded transcript. Private health-record publication also
follows the resolved repository's own instructions and privacy policy.
