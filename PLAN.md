# Object Monologue — Implementation Plan

Status: planning complete for the product experience; technical gates below remain unresolved. No application implementation is included in this document.

## 1. Product specification

**Purpose:** Give everyday objects a voice through short, funny, first-person monologues.

**Primary experience:** Choose or photograph an object → tap **“What’s it thinking?”** → read its thought → generate another take.

**Delivery target:** A private prototype running on the owner’s iPhone through an Expo development build.

### Confirmed requirements

- Expo Router with native **Create** and **Collection** tabs.
- NativeWind for screen styling.
- Camera capture and photo-library selection.
- Preview the photo before generation.
- OpenAI generates English, broadly appropriate monologues of 2–4 sentences.
- AI selects and names the most prominent inanimate object.
- Photos without a suitable subject receive a friendly request for another photo.
- Generate additional takes and browse previous takes within the current session.
- Explicitly save individual photo-and-take pairs locally.
- Collection survives app restarts, displays newest first, prevents duplicate saves of a take, and supports deletion.
- Share a photo-and-quote image with a small Object Monologue credit.
- No user accounts, server database, or server-side content history.
- AI API key remains exclusively on the server.
- Target AI spending: US$10/month, with a server-enforced request allowance and a friendly exhausted-limit message.
- One generation request at a time, a 30-second client timeout, and manual retries.
- Saved favorites work offline; generation requires connectivity.

### Excluded from the prototype

Public store release, Android/web acceptance testing, cloud synchronization, subscriptions, spoken audio, personality selectors, multilingual output, animal monologues, automatic retries, background jobs, folders, and collection search.

## 2. Architecture and interfaces

### Mobile application

Keep the existing Expo project and Expo Router structure. The inspected manifest declares Expo `~58.0.26`, React Native `0.88.1`, and React `19.3.0`; inspect installed versions and the lockfile again before modifying dependencies.

- Routes and navigator layouts belong in `src/app/`.
- Components, session state, storage, API clients, and sharing utilities belong outside the route directory.
- Use React context/reducers for session state; avoid adding a global state library or query cache without a demonstrated need.
- Use device-local files and a versioned JSON collection manifest, avoiding a database service.
- NativeWind styles application content; configure native tab appearance through the navigator’s supported properties.
- Use Expo modules where suitable. Configure native behavior through app configuration and config plugins; do not hand-edit generated native projects.

SDK 58 documents native tabs under `expo-router/native-tabs`. Follow those versioned APIs rather than examples for older SDKs: [native tabs reference](https://docs.expo.dev/versions/v58.0.0/sdk/router/native-tabs.md).

### AI endpoint

Provide one endpoint: **POST `/monologue`**.

Request information:

- Unique request identifier.
- One resized JPEG image.
- Up to three previous monologues for this session, to discourage repetition.

Successful response:

- Request identifier.
- Identified object name.
- Monologue text.

Expected error categories:

- Invalid or oversized image.
- No suitable object.
- Temporary rate limit.
- Prototype allowance exhausted.
- Provider refusal or invalid output.
- Provider unavailable or timeout.

The server owns prompts, input validation, OpenAI credentials, output validation, and spending controls. The mobile client never accepts model names or system prompts from users. Finalize the transport encoding and exact response schema with the endpoint implementation, keeping this contract shared between client and server.

Treat text appearing inside photographs as image content, not instructions.

### Core data types

| Entity | Required information | Lifetime |
| --- | --- | --- |
| Photo session | ID, prepared photo URI, takes, selected take ID, request state | Current app session |
| Take | ID, subject, monologue, creation time | Session unless saved |
| Favorite | ID, source take ID, local image URI, subject, monologue, saved time | Until deleted |
| Collection manifest | Schema version, favorites | Persistent local file |

Store relative image paths and resolve them against the current application document directory. Do not persist temporary picker URIs as favorites.

### Privacy boundary

- Re-encode and resize images before sending; omit EXIF/location metadata.
- Do not persist photos, prompts, or monologues on the app server.
- Do not place image payloads or generated text in logs.
- Explain cloud photo processing near the first generation action.
- Verify OpenAI retention behavior and applicable request settings before finalizing privacy copy; app-server transience does not establish provider-level zero retention.

Reference: [OpenAI data controls](https://developers.openai.com/api/docs/guides/your-data).

## 3. Implementation sequence

Complete and validate each milestone before proceeding. Check off steps only when their acceptance criteria pass. Resolve each technical gate before implementing the affected integration.

### Step 1 — Establish a compatible baseline

- [ ] Inspect installed dependencies, lockfile, current routes, and repository instructions.
- [ ] Read Expo SDK 58 documentation and the current Expo documentation index before using Expo or React Native APIs.
- [ ] Run baseline lint, typecheck, and Expo Doctor; distinguish existing issues from new ones.
- [ ] Verify the exact NativeWind release against Expo 58 and React Native 0.88 before installation.
- [ ] Preserve the existing SDK unless an explicit compatibility blocker requires discussion.
- [ ] Install dependencies through `npx expo install`; this workspace currently uses npm and has `package-lock.json`.
- [ ] Configure NativeWind’s documented Metro/Babel/CSS integration and TypeScript support.
- [ ] Validate a styled screen and native tabs on the iPhone development build.

**Compatibility gate:** The checked stable NativeWind installation guide describes v4.2.7 and Expo 57 support; Expo 58 compatibility is not established by that statement. Do not silently switch styling libraries or downgrade Expo.

References: [Expo SDK 58](https://docs.expo.dev/versions/v58.0.0/), [Expo documentation index](https://docs.expo.dev/llms.txt), [NativeWind installation](https://www.nativewind.dev/docs/getting-started/installation).

### Step 2 — Build navigation and the visual foundation

- [ ] Replace the starter experience with Create and Collection native tabs.
- [ ] Use a warm cream surface, dark readable text, and one bright accent.
- [ ] Use system typography initially, generous touch targets, safe-area-aware layouts, and accessible labels.
- [ ] Provide Create empty, photo-preview, loading, result, and recoverable-error states.
- [ ] Provide Collection empty, populated, detail, and missing-file states.
- [ ] Open directly into Create.

**Acceptance:** Navigation works, content remains readable with larger text, and no controls overlap native tabs or device insets.

### Step 3 — Implement photo acquisition

- [ ] Use the system camera and library flows through `expo-image-picker`.
- [ ] Request permissions when the related action is invoked.
- [ ] Handle cancellation without showing an error.
- [ ] Explain denied permissions and offer a Settings action where appropriate.
- [ ] Normalize orientation and prepare a JPEG with a maximum long edge of 1,024 pixels.
- [ ] Enforce a default upload limit of 2 MiB; recompress or report a recoverable preparation error.
- [ ] Preserve the original library asset; work with an app-owned prepared copy.
- [ ] Starting a new photo resets the unsaved session and invalidates pending responses.

**Acceptance:** Camera and library photos display correctly, including rotated and large images.

Reference: [Expo ImagePicker](https://docs.expo.dev/versions/v58.0.0/sdk/imagepicker.md).

### Step 4 — Implement and connect generation

- [ ] Resolve the OpenAI model, endpoint runtime, access, and spending-control gates below.
- [ ] Use an economical OpenAI model supporting image input and structured output; pin the exact model after current pricing and quality validation.
- [ ] Generate object identification and the monologue in one provider request.
- [ ] Keep output short, first-person, playful, and specific to visible object details.
- [ ] Validate response structure and length before returning it to the app.
- [ ] Disable automatic SDK retries to match the manual-retry product decision.
- [ ] Show loading immediately and disable duplicate submissions.
- [ ] Enforce the 30-second client timeout; configure the server/provider timeout below that limit.
- [ ] Retain the previous displayed take while another is pending.
- [ ] Ignore responses belonging to an obsolete session.
- [ ] Do not automatically retry or start background generation after backgrounding. On return, recover to a usable state without allowing a stale request to overwrite a newer session.

**Acceptance:** Relevant takes appear, repeated taps cannot create parallel requests, and failures preserve the current photo and previous results.

### Step 5 — Add take browsing and collection persistence

- [ ] Display the latest successful take and provide previous/next navigation.
- [ ] Save the selected take independently of other takes.
- [ ] Copy its prepared image into persistent application storage before committing the manifest.
- [ ] Commit manifest updates atomically and serialize storage mutations.
- [ ] Mark already-saved takes visibly.
- [ ] Delete the manifest entry and clean up its associated image safely; preserve any image still referenced by another favorite.
- [ ] Handle storage failures without claiming a save succeeded.
- [ ] Handle a corrupt manifest without silently overwriting recoverable data.
- [ ] Clean up abandoned temporary photos and unreferenced collection files without deleting active or saved images.

**Acceptance:** Favorites reopen after app termination, remain readable offline, and disappear correctly after deletion.

### Step 6 — Add quote-card sharing

- [ ] Render a reusable card containing the photo, complete monologue, and small app credit.
- [ ] Use a default width of 1,080 pixels with content-driven height to avoid text clipping.
- [ ] Capture it to a temporary image using an SDK-compatible view-capture dependency.
- [ ] Share through `expo-sharing`.
- [ ] Do not request photo-library write permission just to share.
- [ ] Treat share-sheet cancellation as normal.
- [ ] Clean up generated temporary cards after they are no longer needed by the share operation.

**Acceptance:** A complete, readable card reaches an actual share destination on the iPhone.

Reference: [Expo Sharing](https://docs.expo.dev/versions/v58.0.0/sdk/sharing.md).

### Step 7 — Add endpoint controls and minimal diagnostics

- [ ] Reject oversized requests before provider invocation.
- [ ] Bound previous-take text and output tokens.
- [ ] Keep secrets in server environment configuration and out of app bundles.
- [ ] Log only request ID, duration, outcome category, and usage totals.
- [ ] Count attempted provider requests conservatively, including uncertain timeout outcomes.
- [ ] Return clear limit-exhaustion and temporary-failure messages.
- [ ] Keep public endpoint deployment blocked until durable, concurrency-safe spending enforcement and access controls are established.
- [ ] Ensure automatic infrastructure retries cannot create uncontrolled paid requests.

**Acceptance:** Invalid requests do not reach OpenAI; exhausted allowance prevents further provider calls; logs contain no user content.

### Step 8 — Validate and deliver

- [ ] Configure app identity, permission descriptions, and the EAS development build profile.
- [ ] Confirm Apple signing and device provisioning prerequisites.
- [ ] Build through `npx eas-cli@latest build --profile development --platform ios`.
- [ ] Verify on the owner’s physical iPhone.
- [ ] Run `npx expo lint`, `npx tsc --noEmit`, and `npx expo-doctor`.
- [ ] Document environment setup, endpoint configuration, build commands, known limitations, and budget reset behavior.
- [ ] Record completed milestones and remaining blockers in this file.

## 4. Test and acceptance plan

### Meaningful automated checks

- Endpoint validation, output parsing, unsuitable-subject responses, and provider failures.
- Allowance exhaustion and concurrent-request accounting.
- Session changes prevent stale responses from replacing the active photo.
- Duplicate saves, collection reload, failed writes, and deletion consistency.

Mock provider responses in automated tests; do not consume paid requests in routine checks.

### Physical-device scenarios

- Capture a mug and select shoes from the library.
- Generate, regenerate, browse previous takes, and save a specific take.
- Relaunch and reopen the saved favorite offline.
- Share the quote card and delete the favorite.
- Cancel camera, picker, and share sheet.
- Deny camera permission.
- Submit a person, animal, blank image, and multi-object scene.
- Simulate offline operation, slow requests, timeout, provider failure, and exhausted allowance.
- Switch photos while generation is pending.
- Background and reopen the app during generation.
- Save multiple takes from one photo and delete one without breaking the others.
- Check large text, VoiceOver labels, long monologues, and image orientation.

**Definition of done:** The agreed core loop works reliably on the iPhone, errors are recoverable, secrets remain server-side, and required checks pass.

## 5. Assumptions and unresolved gates

### Assumptions

These are defaults for implementation, not additional confirmed product requirements:

- No onboarding tour, analytics service, push notifications, or monetization.
- Unsaved sessions need not survive app termination or replacement by another photo.
- Local collection data may disappear on uninstall; cross-device restoration is not promised.
- No fixed deadline.
- No public hosting purchase or deployment has been authorized.
- Basic accessibility is part of the prototype quality bar.
- React context/reducers and local files plus a JSON manifest are sufficient for prototype state and persistence.
- System typography is sufficient initially.
- Initial image preparation defaults to a 1,024-pixel long edge and a 2 MiB upload ceiling.
- Quote-card export defaults to a 1,080-pixel width and content-driven height.

### Decisions required before the affected implementation step

- **NativeWind:** Exact version and demonstrated Expo 58 compatibility.
- **OpenAI:** Exact model, current pricing, response limits, and retention configuration.
- **Hosting:** Provider and deployment topology. Local development can validate the contract before deployment; the physical phone must be able to reach the endpoint.
- **API contract:** Transport encoding and exact shared request/response schema.
- **Spending enforcement:** Durable allowance mechanism compatible with the no-database constraint. Request counts alone cannot guarantee a dollar ceiling; account for payload limits, maximum output, concurrency, retries, and provider pricing.
- **Endpoint exposure:** Private access or abuse controls without user accounts. A token embedded in a distributed app is not a durable secret.
- **Apple delivery:** Available signing account and device provisioning.
- **Backup policy:** Whether device-local favorites should participate in iOS backups.
- **Supporting modules:** Verify SDK-compatible image manipulation, file storage, and view-capture APIs before using them.

These are explicit implementation gates, not permission to silently add accounts, database services, alternative styling frameworks, or paid infrastructure.
