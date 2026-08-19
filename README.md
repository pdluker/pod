# pod.stluker.com -- "Orbit and Ground" daily audio briefing

## What this is

A daily 5-7 minute audio briefing combining `space.stluker.com` and
`earth.stluker.com` into one scripted, narrated episode, published as a standard
podcast RSS feed you subscribe to in Apple Podcasts / Overcast / Pocket Casts.

**It adds no new data sources.** It is a consumer of the two KV blobs the
existing pipelines already write during `stl-dispatcher`'s daily 11:00 UTC run.
Everything downstream of that is one Haiku call, one ElevenLabs call, one R2
write.

```
stl-dispatcher (cron 0 11 * * *)
  Task 4  spaceIngest  -> SPACE_KV
  Task 5  earthIngest  -> EARTH_KV
  Task 7  podcastIngest
            |- reads SPACE_KV + EARTH_KV     (no external news fetches)
            |- 1x Haiku call -> ~900 word ASCII script
            |- budget gate: monthly ElevenLabs credits
            |- 1x ElevenLabs TTS -> mp3
            |- R2 put  pod-audio/episodes/YYYY-MM-DD.mp3
            '- KV put  podcast:episode:*, podcast:manifest, podcast:credits

pod Worker (pod.stluker.com)
  GET /feed.xml           RSS built from podcast:manifest at request time
  GET /audio/:date.mp3    streamed from R2, Range-aware
  GET /transcript/:date   plain text script
  GET /                   landing page + subscribe URL
```

## Cost model

| Line item | Monthly |
|---|---|
| ElevenLabs Creator (already paid) | $22.00 |
| Cloudflare Workers / KV / R2 | $0.00 (free tier) |
| Anthropic Haiku script call, ~30/mo | **~$0.15** |
| **New spend** | **~$0.15/mo** |

Haiku math: ~1,500 input tokens (the two KV digests) + ~1,200 output, 30 days.
The system prompt carries `cache_control: ephemeral` but will rarely hit, since
these calls are 24h apart -- the cache is 5 minutes. It costs nothing to leave in.

### ElevenLabs character budget -- the real constraint

Creator = 100,000 credits/month. A ~900-word script is **~5,400 characters**.

| Model | Credits/char | Per episode | 30 episodes |
|---|---|---|---|
| `eleven_multilingual_v2` | 1.0 | 5,400 | **162,000 -- over budget** |
| `eleven_flash_v2_5` (default here) | 0.5 | 2,700 | **81,000 -- fits** |

So the pipeline ships on Flash. Two consequences worth knowing:

- Flash is lower-latency and cheaper per character; multilingual v2 is a touch
  warmer on long-form narration. For a 6-minute solo read in the car, Flash is
  fine. Swap `TTS_MODEL` + `CREDITS_PER_CHAR` together in `podcast-ingest.js` if
  you disagree -- but then drop to weekdays only (~22 episodes = 119k, still
  over; you would need ~800-word scripts or the $99 Pro tier).
- `MONTHLY_CREDIT_BUDGET` is set to **92,000**, not 100,000. The 8k reserve
  exists so a long script late in the month can't strand you at zero credits
  with other ElevenLabs work pending. ElevenLabs does not auto-charge overage on
  Creator -- generation simply stops -- so running the tank dry means silent
  failures, not a surprise bill.

The ledger in `podcast:credits` tracks actual spend per calendar month and the
task skips itself rather than exceed the budget. Check it via `/health`.

## Deploy runbook

Order matters. Step 1 before step 3, per the `intel` Jul 23 placeholder failure.

```powershell
# 1. Create the resources FIRST, capture the real IDs
wrangler kv namespace create PODCAST_KV
wrangler r2 bucket create pod-audio

# 2. Paste the returned namespace ID into BOTH wrangler.jsonc files
#    (pod/wrangler.jsonc and stl-dispatcher/wrangler.jsonc)
#    -- replace REPLACE_AFTER_CREATE. Do not deploy until this is done.

# 3. Deploy the feed Worker -- FROM THE PROJECT ROOT ONLY
cd C:\Users\pdluk\stl-pod\
wrangler deploy
```

> **Run `wrangler deploy` from `stl-pod\`, never from `stl-pod\public\`.**
> This is the third project where a deploy from the assets subfolder would leak
> `.wrangler/cache/wrangler-account.json` publicly (`stl-status` Jul 14,
> `stl-intel` Jul 23). An `.assetsignore` containing `.wrangler/` belongs inside
> `public\` -- confirm it saved with its leading dot via `Get-ChildItem -Force`
> before the first deploy, not after.

```powershell
# 4. Dispatcher side
cd C:\Users\pdluk\stl-dispatcher\
# copy podcast-ingest.js into this folder
wrangler secret put ELEVENLABS_API_KEY
wrangler secret put ELEVENLABS_VOICE_ID
# apply DISPATCHER-SNIPPET.md, then:
wrangler deploy
```

Then in the Cloudflare dashboard:
- Add `pod.stluker.com` as a custom domain on the `pod` Worker.
- Add the DNS record (Worker route, proxied), same as every other subdomain.
- Drop a 3000x3000 JPEG at `public/cover.jpg` (Apple requires 1400-3000px
  square artwork; the feed will validate without it but won't be listable).

## First run and verification

```powershell
# Force one episode now, bypassing the same-day check
curl.exe -X POST "https://stl-dispatcher.pdluker.workers.dev/trigger?includePodcast=true&forcePodcast=true" -H "Authorization: Bearer <DISPATCH_SECRET>"
```

Read the `diagnostics` array in the response before trusting anything -- it
reports source counts and ages for both KV blobs, token usage, TTS byte count,
and the credit ledger. Then:

```powershell
# Cache-bypass on the first read -- edge cache lag has masked a healthy Worker
# three times in this account now (status Jul 14, music Jul 21, earth Jul 23)
curl.exe -H "Cache-Control: no-cache" https://pod.stluker.com/feed.xml
curl.exe -I https://pod.stluker.com/audio/2026-07-24.mp3   # expect 200 + accept-ranges: bytes
curl.exe -H "Range: bytes=0-1023" -I https://pod.stluker.com/audio/2026-07-24.mp3   # expect 206
```

Validate the feed at `podba.se/validate` or `castfeedvalidator.com` before
submitting anywhere. Then subscribe by URL in your podcast app of choice --
Overcast and Pocket Casts both take a raw feed URL; Apple Podcasts takes one via
Library > Add a Show by URL. Submitting to the Apple/Spotify directories is
optional and not required for it to appear on your phone each morning.

## Known dependency -- read this before deploying

**`earthIngest` is not built yet** (per `stluker-project-index.md`'s Watch
Items: `EARTH_KV` currently holds only a placeholder). Until it is, this
pipeline degrades honestly rather than failing: it reports
`earth: { ok: false }` in diagnostics and, if the combined story count is under
3, skips the day entirely rather than spending credits narrating one source.

Practical options:
1. Build `earthIngest` first -- it's the blocking dependency for the "combined"
   part of this request.
2. Or ship now as a space-only briefing by lowering the `totalStories < 3` gate,
   and let earth stories start appearing automatically once `earthIngest` lands.
   No code change needed on this side when it does.

## Editorial guardrails worth knowing about

The "inspirational and positive" brief is implemented as *framing*, not as a
tone filter over everything. The system prompt explicitly forbids spinning
disasters -- an earthquake or wildfire that harmed people is reported plainly,
and the prompt bans silver-lining framing on those stories outright. The
positivity comes from story selection and from closing on something specific and
forward-looking (a launch window, an instrument coming online, a recovery
underway) rather than a generic uplifting sentence.

It also forbids inventing any fact not present in the source material. That is
the failure mode that matters most here: unlike the `space` briefing panel,
nobody proofreads this before it plays. There are no citations in audio, so
there is no way to catch a hallucinated number at listen time -- hence the
transcript route at `/transcript/YYYY-MM-DD.txt` for spot-checking.

## Doc updates once this is live

Per Standing Operating Instruction #3, edit in place:
- `stluker-project-index.md`: add `pod.stluker.com` to the Subdomain Map; add
  Watch Items for "confirm first unattended daily run produces an episode" and
  "check `podcast:credits` ledger against ElevenLabs' own usage page at month
  end -- confirm the 0.5 credits/char Flash assumption is real, not assumed."
- `stluker-infrastructure.md`: add to Local Deploy Reference, Cloudflare Workers
  list, R2 bucket table, and the dispatcher task inventory (Task 7).
#   p o d  
 