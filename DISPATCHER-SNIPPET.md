# stl-dispatcher wiring -- Task 7: podcastIngest

Paste into `C:\Users\pdluk\stl-dispatcher\dispatcher.js`. Mirrors the Task 6
(`runIntelRefresh`) pattern, with one difference: this task is **ordered last**
and depends on Tasks 4 and 5 having already written their KV blobs in the same
run.

## 1. Import (top of `dispatcher.js`)

```js
import { runPodcastIngest } from './podcast-ingest.js';
```

## 2. Inside `scheduled()` / the shared task runner

Place this **after** `spaceIngest` and `earthIngest` complete, not in parallel
with them. It reads what they write.

```js
// --- Task 7: daily podcast (runs after space + earth have written KV)
let podcastResult = { ran: false, skipped: 'not-attempted' };
try {
  podcastResult = await runPodcastIngest(env, { force: false });
} catch (err) {
  // Never let the podcast take down the rest of the run.
  podcastResult = { ran: true, ok: false, error: err.message };
}
report.podcastIngest = podcastResult;
```

No day-of-week gate: this one is daily by design. The **budget gate inside
`runPodcastIngest` is the cost control**, not the schedule -- if the month's
ElevenLabs credits run low it stops on its own and says so in diagnostics.

## 3. `/trigger` opt-in

Matches `includeBucket` / `includeIntel`:

```js
if (url.searchParams.get('includePodcast') === 'true') {
  const force = url.searchParams.get('forcePodcast') === 'true';
  out.podcastIngest = await runPodcastIngest(env, { force });
}
```

`forcePodcast=true` bypasses the same-day idempotency check **only**. It does
not bypass the monthly credit budget -- that check runs on every path.

## 4. `/health`

```js
podcast: {
  lastEpisode: (await env.PODCAST_KV.get('podcast:credits').then(
    (r) => (r ? JSON.parse(r).lastEpisode : null)).catch(() => null)),
  ledger: (await env.PODCAST_KV.get('podcast:credits').then(
    (r) => (r ? JSON.parse(r) : null)).catch(() => null)),
},
```

## 5. New bindings on `stl-dispatcher`'s `wrangler.jsonc`

The dispatcher already binds `STATUS_KV`. Add:

```jsonc
"kv_namespaces": [
  { "binding": "STATUS_KV",  "id": "208b3be26f2741b9988350d83fc77036" },
  { "binding": "SPACE_KV",   "id": "6ceb0fea2ef3468c9709e9335455f4cd" },
  { "binding": "EARTH_KV",   "id": "cc679a87f521416c9b3d985ffe5f4b78" },
  { "binding": "PODCAST_KV", "id": "REPLACE_AFTER_CREATE" }
],
"r2_buckets": [
  { "binding": "POD_BUCKET", "bucket_name": "pod-audio" }
]
```

> **Do not deploy with `REPLACE_AFTER_CREATE` still in place.** That exact
> placeholder-left-behind mistake is what failed the first `intel` deploy on
> Jul 23 (`KV namespace 'REPLACE_WITH_REAL_NAMESPACE_ID' is not valid [10042]`).
> Create the namespace first (step 1 of the deploy runbook), paste the real ID.

Confirm whether `SPACE_KV`/`EARTH_KV` are already bound to the dispatcher --
`spaceIngest`/`earthIngest` run there, so they very likely are. If so, add only
`PODCAST_KV` and `POD_BUCKET` rather than replacing the block.

## 6. New secrets on `stl-dispatcher`

```
wrangler secret put ELEVENLABS_API_KEY
wrangler secret put ELEVENLABS_VOICE_ID
```

`ANTHROPIC_API_KEY` is already set. As with `INTEL_SECRET`, the dispatcher does
**not** inherit secrets from any other Worker -- set them here explicitly.

## 7. Subrequest accounting

Per-invocation subrequest cost added by this task: 2 KV reads (space, earth) +
1 KV read (episode idempotency) + 1 KV read (ledger) + 1 Anthropic + 1
ElevenLabs + 1 R2 put + 3 KV writes = **10**.

Per Lesson #8 (Jul 24): verify this against the actual dispatcher code before
trusting it in a budget decision -- do not take this README's number on faith.
