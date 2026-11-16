# Auto-Post Daily YouTube Shorts from a Folder — Build Spec for Your Hermes Agent

> **Give this file to your Hermes agent.**
>
> Paste it into a Hermes chat, or save it and say: *"read this file and build this for me"*.
> It is a complete specification: your agent should end up with a working pipeline, not a plan.
> Answer its questions when it asks for the inputs listed in §2 — it cannot guess those.

---

## 1. What this builds

One video per day, automatically, from a local folder to YouTube Shorts:

1. You drop finished vertical clips into a folder.
2. Every day at a fixed time, the agent picks one unposted clip, watches it
   (frames + speech-to-text), writes a title and description, publishes it
   **public**, confirms that a logged-out stranger can actually play it, then
   files the clip away so it is never posted twice.

No editing, no manual work. Runs unattended on your own machine. Total running
cost: **zero** — no paid APIs anywhere.

## 2. Inputs your agent needs from you

| Input | Example | Why |
|---|---|---|
| Video folder | `D:\clips\to_post` | where new clips land |
| Archive subfolder name | `Uploaded` | where posted clips get moved |
| Project folder | `C:\shorts-poster` | where scripts live |
| Channel name | exact channel title | to *verify* uploads went to the right place |
| Google account | the one owning that channel | for browser login + OAuth hint |
| Post time | `18:00` local | daily slot |
| Content language | e.g. Hindi/English mix | drives transcription + title style |
| Posting order | random / oldest-first | how clips get chosen |

## 3. The hard constraint — read this before building anything

Google's own documentation for `videos.insert`:

> *"All videos uploaded via the `videos.insert` endpoint from unverified API
> projects created after 28 July 2020 will be restricted to private viewing
> mode. To lift this restriction, each API project must undergo an audit to
> verify compliance with the Terms of Service."*

Translated: **a new Google Cloud project can upload via the API, but every
video is locked private and cannot be made public — not via the API, not by
hand in YouTube Studio — until YouTube approves the project through its
Compliance Audit** (expect weeks; approval is not guaranteed).

This single fact determines the architecture. Do not let your agent discover
it after building the wrong half.

**The audit is not optional and nothing bypasses it.** Not n8n, not Zapier, not
Make: they are all just clients of the same API against the same project. If a
guide promises otherwise, it is wrong.

### Submit the audit on day one

`https://support.google.com/youtube/contact/yt_api_form`

Suggested wording:

> Personal channel automation. I upload my own short-form videos (created by
> me) to my own channel, one per day, scheduled by a local script. Sole user is
> the channel owner. No user data is collected or shared; the OAuth token is
> stored locally.

### Therefore: two routes

| Route | Public immediately? | Depends on | Fragility |
|---|---|---|---|
| **Browser** — drive YouTube Studio's web UI | **Yes** | a signed-in Chrome + CDP | medium (UI changes); needs guardrails (§7) |
| **API** — `videos.insert` | No, until the audit passes | GCP project + OAuth | none, fully unattended |

**Build both.** Default to the browser route so posting works today, and switch
a config flag to the API route once the audit clears.

## 4. Architecture

```
<project>/
  config.json          settings: folder, order, STT model, channel, route
  state.json           ledger: one entry per posted video (the anti-dupe source of truth)
  poster.py            pick next / status / list / mark-posted (also moves the file)
  transcribe.py        speech -> text, LOCAL whisper (no API, no cost)
  frame.py             pull 3 frames -> one 3-panel strip image
  preflight.py         which files are actually Shorts-eligible
  upload.py            API route: resumable upload + read-back verification
  auth.py              one-time OAuth -> token.json
  upload_studio.py     browser route: publish via Studio UI over CDP
  cdp.py               minimal Chrome DevTools Protocol driver + CLI helpers
  verify_video.py      is it REALLY public? owner view + logged-out view
  make_chrome_profile.py  clone a Chrome profile for safe automation
  frames/  shots/      scratch: frame strips, step screenshots
  RUNBOOK.md           the human-readable ops doc
```

**Rule: never name a Python file after a stdlib module.** A file called
`inspect.py` or `queue.py` in the project directory silently shadows the stdlib
for every import (`from queue import Queue` resolves to *your* file) and breaks
third-party libraries in baffling ways. Hence `preflight.py` / `poster.py`.

## 5. Component spec

### 5.1 The ledger (`state.json`) — build this first

- A dict keyed by filename: `{video_id, url, privacy, method, title, posted_at}`.
- `next` picks a random (or oldest) file from the folder that is **not** in the ledger.
- `mark-posted` writes the entry **and moves the file** into the archive subfolder.
- The archive subfolder is excluded from scanning, so a lost ledger still can't
  cause a re-post. Belt and braces.

### 5.2 Metadata from the actual content

Two local, free signals — use both:

1. **Speech-to-text.** Install `faster-whisper`; run it on CPU. It is local, offline, free.
   - Force the language and pass an `initial_prompt` in the target script. Whisper
     renders Hindi audio in *Perso-Arabic* script unless nudged with a
     Devanagari seed.
   - Enable VAD. Music-only reels return nothing — that is a signal, not a
     failure: fall back to frames.
   - Model size matters: `base` produced unusable garbage in testing; `small`
     was partially usable; test on your own content and pick the smallest model
     that is actually legible. **If the transcript is garbled, do not build a
     title from it** — say so and use the frames.
2. **Frames.** Extract 3 frames (15% / 50% / 85%) and `hstack` them into **one**
   image so one vision call covers the whole clip. Read any on-screen text
   verbatim. Also report `vertical` and `duration` to confirm Shorts eligibility
   (Shorts = vertical or square, ≤ 3 minutes).

Then ask the vision model, and write `meta.json`:
`{"title": str ≤100 chars, "description": str, "tags": [8-15 lowercase]}`.

**Match the channel's existing style.** Have the agent look at the channel's
already-published videos and imitate their title conventions (emoji hooks?
Hinglish? plain?). Title: no hashtags, no invented claims. Description: 1-2
lines, blank line, then 5-8 hashtags including `#Shorts`.

### 5.3 The browser route — the part that goes wrong

Two prerequisites:

1. **Never drive the user's everyday Chrome.** It usually has several Google
   accounts signed in, and the upload will go to whichever one is active. Clone
   the target profile into its own user-data-dir and drive *that*:
   `chrome.exe --user-data-dir=<clone> --profile-directory=... --remote-debugging-port=9222`.
   Cookie databases are locked while Chrome runs — copy the rest of the profile
   normally, and snapshot the cookie DB through SQLite's backup API if you want
   the session to carry over; otherwise sign in once in the new window.
2. **Launch it detached.** On Windows, a Chrome started as a child of a shell
   dies with that shell. Use `Start-Process` (or the platform equivalent).
   Symptom of getting this wrong: the debugging port answers, then later
   "connection refused" and a Chrome "Restore pages?" prompt.

Drive it over **CDP** (Chrome DevTools Protocol) — not screenshot-and-click. Key mechanics:

- Nest the DOM walk. Studio is Polymer with **nested shadow roots**, so
  `document.querySelector` misses most controls. Write one recursive
  `deepQuery(sel)` helper that descends every `shadowRoot`, and reuse it.
- Re-inject your JS helpers **after every navigation** — page globals do not
  survive a page load. Make injection self-healing (check, then inject).
- Attach the video file with `DOM.setFileInputFiles` (walk the *pierced* DOM
  tree to find the `<input type=file>`; it lives inside shadow roots).
- Type into contenteditable title/description with **real input events**
  (`Input.insertText` after focusing + selecting), never by assigning
  `.value`/`.textContent` — Polymer ignores synthetic property changes.
  Verify the text actually stuck by reading the field back.
- Selectors that worked: `#title-textarea #textbox`,
  `#description-textarea #textbox`, `#next-button`, `#done-button`,
  `tp-yt-paper-radio-button[name="PUBLIC"]`,
  `tp-yt-paper-radio-button[name="VIDEO_MADE_FOR_KIDS_NOT_MFK"]`.

### 5.4 The three traps in the Studio publish flow

Each one silently produces a "successful" run that published nothing. Handle all three.

1. **Publish is DISABLED while the file is still uploading.** Clicking it does
   nothing and raises no error. Poll until the button is enabled *and* the
   status text no longer says `Uploading NN%`.
2. **"We're still checking your content"** appears *seconds after* the Publish
   click (not instantly), offering **Publish anyway** / **Go back**. **Nothing
   publishes until it is answered.** So: after clicking Publish, poll for up to
   ~60s for that text, then click "Publish anyway".
   *Note:* YouTube recommends waiting for checks to finish to avoid a strike
   risk. Publishing immediately is the standard automated path, but the honest
   tradeoff is yours to make — for your own original content the risk is low.
   Make it a config flag so the user chooses.
3. **A `youtu.be` link is visible while the video is still a private draft.**
   The link is NOT proof of publishing. This is the trap that produces false
   success.

### 5.5 The verification gate (do not skip)

Never mark a video as posted without independent confirmation:

- **Logged-out check (the strongest).** Open the video URL in a **fresh
  incognito browser context** (`Target.createBrowserContext` — no cookies) and
  confirm it plays.
  Gotcha: an *unavailable* video page has `document.title === "YouTube"`, while
  a real watch page ends with the actual title + `" - YouTube"`. An earlier
  build "verified" a private draft because it only checked that a title existed.
  Require the `" - YouTube"` suffix **and** absence of
  /isn't available|Video unavailable|This video is private/.
- **Owner view.** Load the video's Studio edit page and confirm the visibility
  reads `Public`.
- **Screenshots.** Have the script save a PNG at every step
  (`Page.captureScreenshot`), and have the agent **look at them** (vision) and
  confirm they show "Video published" before believing any return value.
- **Only then** write to the ledger and move the file.

Agent instruction worth repeating verbatim to it: *"A success response is not
evidence. Screenshot it, look at it, and confirm a logged-out viewer can play
it. If the screenshots disagree with the script's output, trust the screenshots."*

Also: **Shorts live under Studio's "Shorts" tab, not "Videos"** — listing the
Videos tab makes a freshly published Short look missing.

### 5.6 The API route (for after the audit)

- Enable *YouTube Data API v3* in a Google Cloud project.
- OAuth consent screen → External → add scopes `youtube.upload` +
  `youtube.readonly` → **publish the app**. Do not skip publishing: while the
  app is in *Testing*, Google expires refresh tokens after 7 days and the daily
  job silently dies every week.
- Create an OAuth client of type **Desktop app**, download the JSON.
- Authorise once. Pass `login_hint=<account>` and `prompt="select_account
  consent"` so the consent screen cannot silently reuse the wrong Google
  account, then **verify**: call `channels.list(mine=True)` and assert the
  returned channel title equals the expected one. Wrong channel → stop.
- Upload with `MediaFileUpload(..., resumable=True)` in chunks; retry on
  500/502/503/504 with exponential backoff; then **read the video back** with
  `videos.list` — a 200 from `videos.insert` is not proof it exists.
- Set `selfDeclaredMadeForKids` explicitly, and decide on
  `notifySubscribers`.

### 5.7 Scheduling with Hermes

- Create a cron job at the daily time. Two things people miss:
  - the **gateway must be running** for jobs to fire at all
    (`hermes gateway status`, then install/start it);
  - with no gateway-connected chat, output is saved locally but not delivered.
- Keep `enabled_toolsets` minimal (e.g. terminal, browser/file, vision) — it is
  the biggest cost lever for recurring agent runs.
- The prompt must be **fully self-contained** (cron runs have no chat context):
  inputs, exact commands, the guardrails from §5.4/§5.5, and what to do when
  the folder is empty (post nothing, report one line).
- Add the launch-if-down step from §5.3 to the job itself, so 18:00 never
  depends on a browser being open.

## 6. Quota and cost (both are non-issues, know why)

- Uploads have their **own daily bucket: 100 `videos.insert` calls/day**,
  separate from the 10,000-unit pool shared by other endpoints.
- The pipeline is free end to end: local whisper, local frame extraction, one
  small vision call per video, no paid service anywhere.

## 7. Pitfalls checklist

- [ ] New GCP project → API uploads locked private until the audit (§3).
- [ ] A `youtu.be` link is not proof — a private draft has one too (§5.4).
- [ ] Publish is disabled mid-upload; clicking it silently no-ops (§5.4).
- [ ] The "still checking" modal arrives *after* the click; poll, don't check once (§5.4).
- [ ] `document.title === "YouTube"` means unavailable, not success (§5.5).
- [ ] Logged-out page ≠ your logged-in view. Test in a real incognito context.
- [ ] Studio uses nested shadow DOM — plain selectors miss controls (§5.3).
- [ ] Page globals die on navigation; re-inject helpers (§5.3).
- [ ] Contenteditable fields need real input events, not property assignment (§5.3).
- [ ] Don't drive the user's main Chrome — multiple accounts = wrong channel
      (§5.3). Verify the active channel before publishing anything.
- [ ] Chrome launched from a shell dies with it; launch detached (§5.3).
- [ ] Chrome restart loses the debug port and shows "Restore pages?" — recover, don't assume it's still attached.
- [ ] Cookies are locked while Chrome runs — snapshot via SQLite, don't `cp`.
- [ ] Never name a project file after a stdlib module (§4).
- [ ] Whisper renders Hindi in Urdu script without a Devanagari `initial_prompt` (§5.2).
- [ ] Music-only clips: STT returns nothing → fall back to frames, don't invent a title.
- [ ] A naive `--dry-run` that still writes the ledger permanently retires a video. Dry runs must change nothing.
- [ ] Shorts are in Studio's Shorts tab, not Videos (§5.5).
- [ ] `hermes gateway` must be running or the cron job never fires (§5.7).
- [ ] On Windows/git-bash: native tools don't translate `/c/...` paths — pass
      `C:/...` to git/python/node. Verify with real output, don't assume.

## 8. Definition of done

- [ ] Dropping a clip in the folder and running the pipeline once publishes it **public**.
- [ ] A **logged-out** browser context plays the published URL.
- [ ] The clip moves to the archive subfolder; the ledger records the video ID.
- [ ] Running again posts a *different* clip, never the same one twice.
- [ ] An empty folder produces a one-line "nothing to post" report, not an error or a retry loop.
- [ ] Screenshots exist for the run and were actually inspected.
- [ ] The cron job is scheduled **and** the gateway is confirmed running.
- [ ] The audit form has been submitted (the clock only starts when you submit).

## 9. Ground rules

- Only upload **your own content** to **your own channel**.
- Never automate a sign-in, an account chooser, or any password/2FA prompt. If
  one appears, the agent must stop and ask — not guess.
- Never fabricate a video ID, link, or success. "I could not confirm it" is a
  valid, useful answer.
- YouTube's terms apply to the user, not the agent: keep the audit honest and
  describe the use case accurately.
