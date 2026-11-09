# HALLYU BOT — FULL IMPLEMENTATION PROMPT

## Role

Act as a senior Node.js/TypeScript engineer, WhatsApp automation engineer, API integrator, and practical product architect. Build a complete, working project named **Hallyu Bot**. Do not stop at a concept, architecture diagram, pseudocode, or attractive demo. Inspect the workspace, plan the implementation, create the actual files, implement the integrations, run the project, test the important flows, and document exactly what works and what still needs credentials or manual setup.

The product must remain deliberately lightweight. Do not turn it into a SaaS platform, streaming website, mobile app, or large backend system.

---

# 1. PRODUCT DEFINITION

**Hallyu Bot is a standalone WhatsApp drama assistant and downloader.** A person saves the bot's dedicated WhatsApp number and messages it like a normal contact. The bot replies conversationally, can talk about dramas, helps users discover titles, and can retrieve video and subtitle downloads through the configured provider.

The intended experience is simple:

1. A user sends `Hi` to the Hallyu Bot WhatsApp number.
2. The bot replies naturally and explains what it can do.
3. The user can chat about dramas or request a title.
4. The bot searches the real provider catalogue and shows actual results.
5. The user chooses a title, season, episode, quality, and subtitle language where available.
6. The bot sends the video as a WhatsApp document when technically supported, and also provides the provider's direct browser download link when that link is genuinely usable.
7. Subtitles are delivered **separately** as subtitle files. Never merge, burn, or embed subtitles into the video.

Hallyu Bot is independent of the Hallyu social app and independent of Hallyu TV. It may promote the Hallyu social app in a later phase, but advertising is not part of the initial release.

## Dedicated WhatsApp number

The owner-provided number is:

- Local format: `09138306698`
- Nigeria international format: `+2349138306698`
- Baileys pairing/config format when digits only are required: `2349138306698`

Treat this as the dedicated bot account, not a user account. Confirm during setup that the WhatsApp account actually connected through Baileys is this number. Do not silently assume that the connected account matches the configured number.

Users do not pair their own accounts with the bot. Only the owner links the dedicated bot number once as the bot's WhatsApp Web companion session.

---

# 2. NON-NEGOTIABLE SCOPE

## Build these things

- WhatsApp messaging through Baileys.
- Natural conversation and drama discussion using a configurable AI provider.
- Real drama search, catalogue browsing, drama details, seasons, episodes, qualities, video sources, and captions through Zarhosting Moviebox.
- Separate video and subtitle delivery.
- WhatsApp document attachment delivery when supported by the current connection and actual file size.
- Direct provider download links for a user's browser when they are verified to work without hidden server-only headers or cookies.
- A simple WhatsApp admin menu.
- Minimal local statistics and configuration stored in small JSON files.
- Clear setup instructions and a local development workflow.

## Explicitly do NOT build these things

- No Supabase.
- No Firebase.
- No PostgreSQL, MongoDB, MySQL, Redis, Prisma, Drizzle, or other database system.
- No cloud database or cloud media storage.
- No permanent video or subtitle storage.
- No separate website, web dashboard, mobile app, or Telegram bot.
- No user registration, passwords, user profiles, or account system.
- No user watch history or persistent conversation-history product.
- No FFmpeg pipeline, subtitle muxing, subtitle burning, or video re-encoding.
- No OpenSubtitles integration. Moviebox/Zarhosting captions are the only planned subtitle source.
- No fake catalogue, mock dramas, placeholder download URLs, fabricated resolutions, or fake success messages.
- No mandatory VPS purchase or deployment setup during the initial development phase.

The bot application itself is a Node.js process that connects to WhatsApp and calls external APIs. Do not interpret “no backend” as “no code process at all.” The requirement is **no separate hosted backend service or database platform**. Develop and test locally first. Public 24/7 hosting can be considered later as a separate deployment decision.

---

# 3. TECHNICAL STACK

Use a simple, maintainable JavaScript/TypeScript stack:

- Node.js LTS.
- TypeScript preferred; use strict TypeScript settings.
- Baileys for the WhatsApp Web companion connection. Check the current maintained package/version and its current API before coding. Do not copy obsolete examples blindly.
- Native `fetch` or a small HTTP client for provider requests.
- An AI SDK/adapter appropriate to the selected provider.
- Pino or another lightweight structured logger.
- JSON files for local configuration and statistics only.
- Vitest or Node's test runner for tests.

Do not add Fastify, Express, or another HTTP server unless a genuinely necessary feature is approved later. The initial application does not need a public web server.

Keep dependencies few. Explain any dependency that is not obvious and avoid adding packages that duplicate Node.js built-in functionality.

---

# 4. WHATSAPP CONNECTION

Use Baileys as the initial WhatsApp transport because the owner wants to avoid building around the paid Meta Cloud API during the prototype stage.

Implementation requirements:

1. Pair only the dedicated Hallyu Bot number above.
2. Support QR pairing and, if the installed Baileys version supports it reliably, pairing-code setup.
3. Persist the Baileys authentication/session credentials locally in a private folder such as `auth/` so the owner does not need to pair on every restart.
4. Never commit the auth folder, QR/session secrets, or credentials to Git.
5. Add reconnection handling with sensible backoff. Do not create an aggressive reconnect loop.
6. Handle WhatsApp connection updates and log whether the connection is connecting, open, closed, or needs re-pairing.
7. Avoid replying to the bot's own messages, status messages, unsupported message types, or groups unless group support is explicitly enabled later.
8. Deduplicate incoming message IDs so reconnects or repeated events do not trigger duplicate downloads.
9. Do not send repeated messages for one request.
10. Do not mark a download successful merely because a provider URL was returned. Track the actual delivery result.

Important operational warning: Baileys is an unofficial WhatsApp Web integration, not the official Meta WhatsApp Cloud API. It can become unstable when WhatsApp changes its protocol and may expose the dedicated number to restrictions or bans. The owner must knowingly accept this risk and should use a number they control. Keep the WhatsApp transport isolated behind an adapter so an official provider could be integrated later without rewriting the drama and conversation logic. Do not claim this approach is officially supported by WhatsApp.

---

# 5. ZARHOSTING / MOVIEBOX PROVIDER INTEGRATION

This is the most important external integration. Implement it carefully and verify it with real responses. Do not fabricate response shapes.

## Known provider base URL

```text
https://movieapi.zarhosting.dpdns.org/api/public/v1
```

Prior Hallyu TV integration/testing established that this public Moviebox API was used without an API key for the tested catalogue and stream calls. Recheck that the API is reachable and currently behaves this way before relying on it. Do not invent or insert an API key.

## Previously used route map

These routes and parameters are the known starting points from the earlier Hallyu TV work. Treat them as integration notes to verify against the existing Hallyu TV source code and live provider responses, not as permission to guess undocumented fields.

| Purpose | Known route pattern |
|---|---|
| Home/catalogue feed | `GET /home` |
| Search | `GET /search?q={encoded_query}&page={page}` |
| Drama/movie details | `GET /detail/{detail_path}` |
| Episode stream/source lookup | `GET /stream/{subject_id}` with the required title/episode query values |
| Episode captions | `GET /stream/{subject_id}/captions` with the required episode values; `stream_id` may be required depending on the provider response |

The stream and captions requests are known to need episode context. Previously tested parameters included:

- `subject_id`
- `detail_path` (the provider's title slug, not a display title)
- `se` (season number)
- `ep` (episode number)
- `stream_id` where returned by the source and required by the captions route

A previously tested example from Hallyu TV was *Breaking Bad*:

```text
subject_id = 6207982430134357800
detail_path = breaking-bad-ej6Bp0MCAo7
se = 1
ep = 1
```

Use this only as a diagnostic reference if the provider still recognizes it. Do not hard-code this example into the user-facing catalogue or assume it will always remain available.

## Existing Hallyu TV code audit is mandatory when available

Before implementing the provider, search the current workspace for the existing Hallyu TV project/source files. Reuse the proven Zarhosting request construction, response parsing, headers, and episode selection logic where it is present. Do not copy the Hallyu TV UI or unrelated app features. Only reuse provider-integration knowledge.

If the Hallyu TV source is not present in the workspace, state that clearly in the final report and verify the public provider endpoints directly using small diagnostic requests. Do not claim to have inspected Hallyu TV if it was not available.

## Required provider headers

Earlier tests used Moviebox origin/referrer headers for CDN source access:

```text
Referer: https://www.moviebox.ph/
Origin: https://www.moviebox.ph
User-Agent: a normal current browser user-agent
```

Verify which requests actually need these headers. Keep them in one provider configuration module. Do not scatter hard-coded headers across bot handlers. Do not spoof credentials, bypass DRM, bypass paywalls, or circumvent access controls.

## Provider implementation rules

1. Build a dedicated `ZarhostingMovieboxProvider` module/class. Do not call provider URLs directly from WhatsApp message handlers.
2. Normalize inconsistent provider response shapes into stable internal types, for example:
   - `DramaSummary`
   - `DramaDetails`
   - `SeasonInfo`
   - `EpisodeInfo`
   - `VideoSource`
   - `CaptionSource`
3. Preserve provider IDs and slugs exactly. Never substitute a display title for `detail_path` or invent a `subject_id`.
4. Preserve the provider's episode numbering. Do not assume every series starts at season 1 or that every episode list is contiguous.
5. Show only qualities and formats actually returned by the provider. Never advertise 1080p if only 720p is available.
6. Support pagination for search/catalogue responses when the provider offers it.
7. Parse and sanitize result data before displaying it in WhatsApp.
8. Use request timeouts, limited retries for transient failures, and clear error messages.
9. Handle rate limits, unavailable titles, missing sources, malformed JSON, empty captions, expired CDN links, and provider downtime.
10. Cache only small catalogue metadata briefly in memory if useful. Do not create a persistent media cache.
11. Avoid repeatedly fetching the same detail/source request during a single active user flow.
12. Log provider status, endpoint name, duration, and safe error details. Never log API keys, session credentials, or private user message contents unnecessarily.
13. Build sanitized test fixtures from actual responses after the first successful live test. Tests must not require the provider to be online.

## Provider acceptance checks

Before declaring the integration complete, verify at least:

- Home feed returns parseable real titles.
- Search returns real matching results and handles no results.
- A selected result resolves to its correct detail page.
- Series seasons/episodes are represented accurately.
- A selected episode resolves to at least one real source if the provider has one.
- Captions are fetched for that exact selected episode.
- The bot does not confuse captions for another season/episode.
- Video and subtitle URLs are tested for actual accessibility, not just string presence.

If any route or response differs from these notes, follow the actual working Hallyu TV implementation or current live API contract and document the difference. Do not make up a route to force the table above to work.

---

# 6. VIDEO DOWNLOAD AND DELIVERY

The user wants both delivery methods:

1. A real WhatsApp document attachment whenever the current WhatsApp connection and file size support it.
2. A direct download link that opens in the user's browser, whenever the provider supplies a genuinely browser-accessible link.

When both are supported, send both. Do not force the owner to choose one permanently.

## No permanent media storage

- Do not upload videos to Firebase, Supabase, S3, R2, or another cloud storage service.
- Do not keep a video library on the bot machine.
- Do not create a permanent download cache.
- Prefer streaming provider data into the WhatsApp media sender if the current Baileys implementation supports a safe stream input.
- If a WhatsApp document upload requires buffering, use a bounded in-memory buffer only under a configurable size threshold. Check file size before loading it into memory.
- Do not write full video files to persistent disk. If a library absolutely requires a temporary file, do not silently introduce it: document the limitation and make the smallest OS temporary-file fallback optional, tightly size-limited, and deleted in a `finally` block after success or failure. The default should remain no disk persistence.
- If a file is too large or cannot safely be relayed as a document, do not attempt to buffer several gigabytes into RAM. Send the verified direct provider link instead.

## Direct link validation

A URL is not a usable direct download link merely because the provider returned a string.

Before presenting a direct link, determine whether it:

- Is reachable from an ordinary browser without custom headers, cookies, or app-only authentication.
- Has not already expired.
- Points to the selected episode and selected quality.
- Returns a usable video response and a sensible content type.

If the provider requires `Referer`, `Origin`, cookies, or headers that a user's browser cannot provide, do not label that URL as a working direct browser link. Do not create a custom proxy server in this project to hide the problem. Report that a browser-ready link is unavailable and use WhatsApp attachment delivery if supported.

## Attachment size and WhatsApp limits

Do not assume WhatsApp accepts 5 GB documents. Do not hard-code an unverified maximum. Check current Baileys/WhatsApp behavior and use a conservative configurable setting such as `MAX_WHATSAPP_ATTACHMENT_BYTES`. Attempt document delivery only when the real file size is below the configured threshold and the connection/library supports it. If file size is unknown, use a safe bounded strategy rather than unbounded buffering.

Send MP4 as a document (not a compressed WhatsApp video message) when possible, so the original file remains a downloadable file with a filename. Use a clear filename such as:

```text
The.Last.Immortal.S01E12.720p.mp4
```

Do not claim a document was delivered until Baileys confirms that the send operation completed without throwing. If an attachment upload fails, report that honestly and still provide a verified direct link if one exists.

## Delivery message example

```text
🎬 The Last Immortal
Season 1 · Episode 12 · 720p

WhatsApp document: [attached when supported]
Direct download: [verified provider link when available]

📝 English subtitles are available separately.
```

Do not send fake placeholder links. Do not put a link in a message unless it resolves to the requested file.

---

# 7. SUBTITLE SYSTEM — SEPARATE FILES ONLY

Moviebox/Zarhosting captions are the only subtitle provider for this version. Do not integrate OpenSubtitles.

Subtitles must be delivered independently from the video:

- The video remains an unmodified MP4/source file.
- The subtitle is a separate file, usually `.srt`, `.vtt`, or another format actually returned by the provider.
- Never mux subtitles into the video.
- Never burn subtitles into the picture.
- Never re-encode video.
- Do not add FFmpeg for this feature.

Requirements:

1. Retrieve captions for the exact selected title, season, and episode.
2. Display only languages and subtitle files the provider actually returns.
3. Let the user choose a language when more than one is available.
4. If the provider gives a direct browser-accessible subtitle URL, send it.
5. If the provider returns subtitle text/content instead of a public URL, create a small in-memory file buffer and send it as a WhatsApp document with the correct extension and MIME type, if supported.
6. Do not create a permanent subtitle file on disk.
7. Use meaningful filenames, for example `The.Last.Immortal.S01E12.en.srt`.
8. If no subtitle exists, say so. Never manufacture subtitle files or silently use subtitles for a different episode.
9. Preserve subtitle format and language metadata when available.
10. If caption results are ambiguous, show the available choices and ask the user to choose instead of guessing.

The bot must clearly explain to users that they load the downloaded subtitle separately in a compatible player such as VLC or MX Player. Do not imply that the subtitle is already inside the video.

---

# 8. AI DRAMA ASSISTANT

The bot must do more than match commands. Users should be able to talk to it naturally about dramas and entertainment.

## AI provider must be configurable

Do not hard-code Gemini as the only possible AI service. Create a small provider adapter interface so the owner can configure one supported service through environment variables.

Support the following configuration pattern:

```env
AI_PROVIDER=gemini
AI_API_KEY=
AI_MODEL=
AI_BASE_URL=
```

- Implement Gemini as the first straightforward option if the owner supplies a Gemini key.
- Keep an adapter path for OpenAI or another OpenAI-compatible service.
- Allow an NVIDIA NIM or other hosted model endpoint only when the selected NVIDIA service actually supports the configured API protocol. Verify its model ID, endpoint, and authentication format from the owner's chosen provider documentation; do not assume every NVIDIA open-source model is a hosted API.
- A local model endpoint can be an optional later configuration, not a requirement for the first build.
- Never ask the owner to paste API keys into source files or commit them to Git.
- If the AI key is absent or invalid, command-based catalogue search and downloads must still work. Return a helpful fallback message rather than crashing the bot.

## AI behavior

The assistant should be able to:

- Respond to normal greetings and casual conversation.
- Discuss K-dramas, C-dramas, J-dramas, anime, Hollywood films and series, and other entertainment where it has reliable knowledge.
- Recommend dramas based on genre, mood, pacing, themes, and titles the user mentions.
- Compare two dramas.
- Explain a drama's premise, themes, characters, and genre.
- Help users decide what to watch next.
- Answer questions about cast, release details, episode counts, and ratings only when the information is known or verified through provider/catalogue data.
- Explain the bot's search and download process.
- Switch naturally from discussion into a real catalogue search when the user says they want to download or find a title.

## AI accuracy and spoiler rules

- Avoid inventing release dates, episode counts, cast members, ratings, or provider availability.
- Use the Zarhosting catalogue for current availability, episode lists, quality options, and caption availability.
- AI-generated knowledge must never manufacture a download URL.
- Ask before revealing major spoilers. Default to spoiler-light answers.
- If asked about a specific episode's ending or plot and the model is uncertain, say so rather than making up a recap.
- Do not claim that the AI has watched a show or personally experienced it.
- Keep replies natural, concise, friendly, and useful for WhatsApp. Avoid essay-length responses unless the user asks for detail.
- Use tools/function calling or a deterministic intent router to call catalogue search/details when necessary. The AI should not be the source of truth for provider data.

---

# 9. USER EXPERIENCE AND WHATSAPP CONVERSATION FLOWS

Support both natural-language requests and a simple menu.

## Welcome / greeting

When a new person says `Hi`, `Hello`, `Hey`, or similar, reply with a natural welcome that introduces Hallyu Bot and explains that it can discuss dramas, find titles, and provide episode downloads and separate subtitles.

Do not send an oversized wall of text. Provide a compact menu, for example:

```text
Welcome to Hallyu Bot 🎬
Your drama companion and downloader.

1. Search for a drama
2. Trending / popular titles
3. Browse categories
4. Talk about dramas
5. Help

You can also just type what you're looking for.
```

## Natural-language examples that must work

- `I want to download The Last Immortal.`
- `Find historical Chinese dramas.`
- `Show me popular K-dramas.`
- `Do you have episode 12?`
- `Download episode 12 in 720p.`
- `I need English subtitles.`
- `What should I watch after Alchemy of Souls?`
- `Is The Last Immortal worth watching?`
- `Compare The Untamed and Word of Honor.`

## Search flow

1. Extract the title or search phrase.
2. Search the real provider API.
3. Return a small, readable numbered list of matching titles with available metadata.
4. Let the user select by number or title.
5. Support `next`, `previous`, and `cancel` when pagination exists.
6. If there are many matches, ask a useful narrowing question rather than dumping the entire catalogue.
7. Do not show fabricated posters, ratings, years, genres, or descriptions.

## Detail and episode flow

After a title is chosen:

1. Show title, poster if a valid image URL is provided, year/genre/description when available, and any actual metadata.
2. For a series, display the seasons that really exist.
3. Display the episodes available in the selected season.
4. Let the user select one episode or a clearly specified range.
5. Never interpret a request for one episode as permission to download the whole season or series.
6. For a movie, skip season/episode selection.

## Quality selection

Show only the actual qualities returned by the provider, such as 480p or 720p when those options are present. Do not invent 1080p/4K choices.

If the user already specifies a quality and it is available, do not ask them again. If unavailable, show the real alternatives.

## Subtitle selection

- Ask whether they want a subtitle only when it is not already clear from their request and captions are available.
- If a language was specified, use that language if available.
- If several versions of the same language exist, show distinguishing provider metadata such as format or release label when available.
- Subtitles remain separate downloads.

## Conversation state

Use a small in-memory conversation state machine per chat for pending selections. This is not a conversation-history database.

- Store only what is needed for the current selection flow (search results, selected title, season, episode, quality, language, and expiry time).
- Expire inactive state after a configurable short period, for example 15–30 minutes.
- Clear state on completion or cancellation.
- Do not store full message transcripts.
- On restart, users can simply start the request again. Do not introduce a database just to persist temporary menus.

## WhatsApp UI compatibility

Prefer plain numbered text choices because they work across more WhatsApp clients and Baileys versions. Interactive list/button messages may be used only if the installed library/version supports them reliably. Always provide a plain-text fallback. Do not make the bot depend on unsupported interactive message features.

---

# 10. ADMIN MENU

There is no separate admin website. Administration happens inside WhatsApp.

The owner must configure administrator phone numbers separately from the bot's own number. Do not assume the bot number is also the administrator's personal number.

Use environment configuration such as:

```env
ADMIN_NUMBERS=
ADMIN_PASSCODE=
```

Do not put the admin passcode in source code or in a committed JSON file. Use an allowlisted admin number and a passcode for sensitive actions. Rate-limit incorrect attempts. Keep an authenticated admin session short-lived and in memory. Do not expose admin commands to ordinary users.

Admin entry can use `/admin` or `admin`, but only allowlisted numbers may enter the flow.

Initial admin features:

- View total unique users seen by the bot (do not call them registered users; there is no registration system).
- View total video download requests.
- View total subtitle download requests.
- View successful and failed deliveries.
- View popular requested dramas.
- View simple daily totals.
- View provider health / last provider error.
- View WhatsApp connection status.
- View current bot settings that are safe to display.
- Block or unblock a user number using a local blocklist.
- Send a manually approved announcement to opted-in users only, with conservative rate limits and a clear opt-out mechanism.

Do not build advanced analytics, a dashboard, roles/permissions management, or a multi-admin organization system in version one.

---

# 11. LOCAL JSON DATA — NOT A DATABASE

Use a few small local JSON files only for settings and basic counts. Do not install a database engine.

Suggested layout:

```text
data/
  stats.json
  seen-users.json
  blocked-users.json
  settings.json
```

Possible aggregate statistics:

```json
{
  "uniqueUsersSeen": 0,
  "videoRequests": 0,
  "subtitleRequests": 0,
  "successfulAttachments": 0,
  "directLinksSent": 0,
  "failedDeliveries": 0,
  "popularTitles": {},
  "dailyTotals": {},
  "lastUpdatedAt": null
}
```

Requirements:

- Count a user once when their WhatsApp sender ID is first seen. This is not user registration.
- Store as little user information as possible. Do not store message history.
- Keep any local identifiers solely for unique-user counting or admin blocking.
- Update counters only when the corresponding event actually happens. For example, do not count a video as successfully delivered when the provider lookup succeeds but the WhatsApp upload fails.
- Use safe/atomic JSON writes (write a temporary JSON file and rename it, or another safe method) to reduce corruption risk.
- Serialize concurrent writes so two simultaneous downloads do not overwrite each other's counters.
- Validate JSON on startup. If a file is malformed, preserve a backup and recover safely instead of crashing silently.
- Create default files automatically if missing.
- Do not commit live user statistics, local admin data, or private runtime state to Git.
- Provide a simple manual backup/export instruction in README.

The bot does not need a database for the initial requirements. If the project later grows beyond what safe local JSON writes can handle, discuss an upgrade with the owner first. Do not add a database pre-emptively.

---

# 12. CONFIGURATION AND SECRETS

Create `.env.example` with placeholders only. Never put real keys, session credentials, or admin passcodes in source files.

At minimum:

```env
NODE_ENV=development
BOT_PHONE_NUMBER=2349138306698
WHATSAPP_AUTH_DIR=./auth
ZARHOSTING_BASE_URL=https://movieapi.zarhosting.dpdns.org/api/public/v1
ZARHOSTING_TIMEOUT_MS=20000
AI_PROVIDER=gemini
AI_API_KEY=
AI_MODEL=
AI_BASE_URL=
ADMIN_NUMBERS=
ADMIN_PASSCODE=
MAX_WHATSAPP_ATTACHMENT_BYTES=
CONVERSATION_STATE_TTL_MINUTES=20
LOG_LEVEL=info
```

Validate required values at startup. The bot should still start in catalogue-only mode when the AI key is missing, and it should clearly report that AI chat is disabled until configured.

Add `.env`, `auth/`, `data/`, temporary files, logs, and build output to `.gitignore` as appropriate. Provide `.env.example` but never `.env` with real credentials.

---

# 13. SECURITY, RELIABILITY, AND RESOURCE LIMITS

- Validate and normalize incoming sender JIDs and phone numbers.
- Escape or sanitize untrusted provider fields before composing WhatsApp messages.
- Prevent ordinary users from invoking admin operations.
- Rate-limit repeated searches and download requests per sender.
- Prevent duplicate processing of the same incoming WhatsApp message.
- Add timeouts and sensible retry limits to provider and AI requests.
- Do not retry a non-idempotent WhatsApp attachment send blindly, because this may deliver duplicate files.
- Use bounded concurrency for media delivery so several users cannot exhaust memory.
- Check content length where available before buffering media.
- Never allocate a huge Buffer for an unknown-size video.
- Do not expose raw stack traces, internal paths, session files, API keys, or admin configuration to users.
- Do not log full private chats unnecessarily.
- Handle shutdown cleanly and preserve the Baileys auth session.
- Keep secrets and local runtime files outside version control.
- Implement a clear `/help` and `/cancel` behavior.
- Make failure messages useful and truthful.

Only retrieve and distribute media from sources the operator is authorized to access and redistribute. Do not implement DRM circumvention, access-control bypass, or paywall evasion. Provider availability by itself is not proof of redistribution permission.

---

# 14. PROJECT STRUCTURE

Keep the architecture organized but not over-engineered. A suitable starting structure is:

```text
hallyu-bot/
  src/
    index.ts
    config/
      env.ts
    whatsapp/
      connection.ts
      message-handler.ts
      send-message.ts
    bot/
      router.ts
      welcome.ts
      help.ts
      conversation-state.ts
      flows/
        search-flow.ts
        detail-flow.ts
        download-flow.ts
        subtitle-flow.ts
    providers/
      zarhosting/
        client.ts
        provider.ts
        normalizers.ts
        types.ts
    ai/
      ai-provider.ts
      gemini-provider.ts
      openai-compatible-provider.ts
      drama-assistant.ts
    downloads/
      source-resolver.ts
      attachment-sender.ts
      direct-link-validator.ts
    subtitles/
      caption-service.ts
      subtitle-sender.ts
    admin/
      admin-router.ts
      admin-auth.ts
      admin-menu.ts
    stats/
      stats-service.ts
      json-store.ts
    utils/
      logger.ts
      errors.ts
      phone.ts
      rate-limit.ts
  data/
    .gitkeep
  tests/
  auth/                 # ignored by Git; created at runtime
  .env.example
  .gitignore
  package.json
  tsconfig.json
  README.md
```

You may adjust the structure if the implementation clearly benefits, but keep provider, WhatsApp, AI, download, subtitle, admin, and local-statistics responsibilities separate. Do not add empty files or unused abstraction layers just to make the tree look impressive.

---

# 15. TESTING REQUIREMENTS

Write and run tests for:

1. Phone-number normalization for the configured Nigerian number.
2. Incoming-message deduplication.
3. Greeting and help responses.
4. Natural-language search intent routing.
5. Provider search response normalization.
6. Provider detail and episode normalization.
7. Correct season/episode parameters passed to stream lookup.
8. Caption lookup for the exact selected episode.
9. Only real provider qualities and languages are shown.
10. Missing captions are handled gracefully.
11. AI disabled/missing-key fallback still allows catalogue and downloads.
12. Admin authorization rejects non-admin senders.
13. Local JSON statistics are updated safely under concurrent events.
14. Unique users are counted once.
15. Failed WhatsApp attachment delivery is not counted as successful.
16. Direct links are not labelled browser-ready unless validation passes.
17. Oversized videos do not trigger unbounded memory allocation.
18. Conversation state expires and `/cancel` clears it.

Use sanitized provider fixtures for automated tests. Do not make the entire test suite depend on Zarhosting or an AI provider being online. Provide a separate manual live-integration test command.

---

# 16. DEVELOPMENT AND DELIVERY PROCESS

Follow this order:

## Phase A — Inspect and plan

- Inspect the entire current workspace before changing files.
- Check whether any Hallyu Bot code already exists. Do not overwrite useful existing work blindly.
- Search for Hallyu TV provider integration code if the source is present.
- Write a short implementation plan and list any assumptions.
- Confirm the correct Baileys package/version and current APIs.

## Phase B — Build the working foundation

- Create the Node.js/TypeScript project.
- Add configuration validation, logging, and safe local JSON files.
- Implement the WhatsApp connection and reconnect handling.
- Implement basic greeting, help, and cancel flows.
- Confirm the actual connected account is the configured bot number.

## Phase C — Integrate Zarhosting

- Implement the provider client and normalizers.
- Verify home, search, details, episode stream lookup, and captions using real provider responses.
- Reuse the Hallyu TV implementation when it is actually available.
- Add fixture-based tests.

## Phase D — Integrate AI

- Implement a provider adapter.
- Start with the provider configured by the owner.
- Implement drama discussion and recommendation behavior.
- Ensure missing AI credentials do not break core bot functionality.

## Phase E — Implement downloads and subtitles

- Implement exact episode/quality selection.
- Implement verified provider direct links.
- Implement bounded WhatsApp document attachment delivery.
- Implement independent subtitle selection and delivery.
- Confirm no media is permanently stored and no subtitle is embedded.

## Phase F — Admin and statistics

- Add local JSON counters.
- Add allowlisted WhatsApp admin access.
- Add the minimal admin menu and blocklist.
- Test that non-admin users cannot access it.

## Phase G — Verify and document

- Run tests, type checks, linting if configured, and a production build.
- Fix errors rather than merely listing them.
- Run available live provider checks.
- Document manual WhatsApp pairing and API key configuration.
- Document what was tested and what remains dependent on external access.

Do not buy, provision, or configure a VPS as part of this implementation. Local development and testing come first. Explain later hosting requirements in the README only as a future deployment note.

---

# 17. DEFINITION OF DONE

The implementation is not complete until all applicable items below work:

- The project starts locally using documented commands.
- The owner can pair the dedicated number `2349138306698` as the bot's WhatsApp companion session.
- The bot responds to greetings and help requests.
- The bot can discuss dramas through the configured AI provider.
- The bot can search real Zarhosting titles.
- The bot can show real details, seasons, episodes, and available qualities.
- The bot can resolve the selected episode's actual video source.
- The bot can resolve captions for that exact episode from Moviebox/Zarhosting.
- The user can receive a WhatsApp document attachment when supported.
- The user can receive a verified browser download link when available.
- Subtitles are separate downloadable files.
- There is no OpenSubtitles integration.
- There is no subtitle embedding or FFmpeg processing.
- There is no Supabase, Firebase, database engine, cloud storage, or permanent media archive.
- Basic statistics persist in local JSON files.
- Admin features are protected by an allowlist and passcode.
- Tests and type checking pass.
- README setup instructions are accurate.

If an item cannot be completed because an external provider is unavailable, credentials are missing, the direct URL is not browser-accessible, or WhatsApp rejects an attachment size, report that exact limitation. Do not fake a successful test.

---

# 18. FINAL REPORT REQUIRED FROM THE CODING AGENT

When implementation is finished, provide a clear report containing:

1. What was built.
2. The final project file structure.
3. Exact commands to install dependencies and run the bot locally.
4. How to pair the dedicated WhatsApp number.
5. Which AI provider was configured and which environment variables the owner must fill in.
6. Which Zarhosting routes were verified and which were not.
7. Whether video document delivery was tested and the actual tested size/limit.
8. Whether browser direct links were verified from a normal browser context.
9. Whether subtitle attachment/link delivery was tested.
10. Test/type-check/build results.
11. Any remaining manual steps or external limitations.
12. Explicit confirmation that no Supabase, Firebase, database, cloud storage, OpenSubtitles, or permanent media storage was added.

Do not say “everything works” unless the corresponding behavior was actually tested. Do not deploy publicly or spend money without the owner's explicit approval.

**Build Hallyu Bot as a simple, dependable WhatsApp drama companion and downloader. Prioritize correct provider integration, truthful download delivery, reliable WhatsApp behavior, and a clean user experience. Resist unnecessary infrastructure.**
