Title: From AIza to AQ.: Inside Google's 2026 Gemini API Key Overhaul
Date: 2026-09-24
Category: Article
Tags: Gemini API, Google AI Studio, API Keys, Auth Keys, AIza, API Security, Google Cloud, Developer Tools, Migration Guide
Slug: from-aiza-to-aq-gemini-api-key-overhaul-2026

For years, every Gemini API key looked the same: a long string starting with AIzaSy, the familiar prefix shared across Google Cloud's API ecosystem — Maps, Firebase, and now Gemini. In 2026, that changed. Developers logging into Google AI Studio to grab a fresh key started seeing something unfamiliar: a string beginning with AQ.Ab instead. This wasn't a bug. It was the start of a deliberate, staged migration away from the old key format entirely — and by September 2026, the old format stops working altogether.

Here's what actually changed, why Google made the switch, and what it means if you're still holding an AIza key.

# **The Two Key Types**

Google now distinguishes between two categories of Gemini API credentials:

**Standard API keys** — the legacy format, prefixed AIza. These keys simply associate a request with a Google Cloud project for billing and quota tracking. They don't identify who's calling, which limits how finely Google can control access or respond to abuse.

**Authorization (auth) keys** — the new format, prefixed AQ.. These are bound directly to a Google Cloud service account. Requests made with an auth key run under that service account's identity, which enables much more granular access control. Auth keys are also restricted to the Generative Language API by default, and Google can shut down a leaked auth key almost immediately once it's detected — something standard keys never supported well.

**Standard key (AIza…)**

1. Identity: tied to a Cloud project only, doesn't identify the caller

2. Default scope: can be broad — sometimes valid across other Google APIs too

3. Leak response: slow, manual revocation

4. Cloud usage metrics: shows up in standard service account metrics

5. Status as of late 2026: being phased out

**Auth key (AQ.…)**

1. Identity: bound directly to a service account

2. Default scope: restricted to the Gemini API only, by default

3. Leak response: fast-acting, automated enforcement once a leak is detected

4. Cloud usage metrics: not recorded there — tracked on the AI Studio side instead

5. Status as of late 2026: the default for every new key

# **Why Google Made the Switch**

The trigger was a security disclosure. Security researchers found that simply enabling the Generative Language API on a Google Cloud project silently extended Gemini access to every existing key already tied to that project — including old keys originally created for entirely different services. The result: thousands of live keys sitting exposed in public repositories and client-side bundles, discovered scanning the open web. In one documented case, a single leaked key generated tens of thousands of dollars in usage within 48 hours on an account that normally cost under $200 a month.

Google initially pushed back on the severity of the finding, then acknowledged it — and auth keys, with their service-account binding and automated leak response, are the direct result.

# **The Migration Timeline**

Google rolled the change out in stages rather than flipping a switch overnight:

1. **May 7, 2026** — Unrestricted standard keys that had gone unused for an extended period started getting a Blocked tag in AI Studio.

2. **May 28, 2026** — Every new API key created in Google AI Studio began defaulting to the auth key format automatically. Developers could no longer request a fresh AIza key even if they wanted one.

3. **June 19, 2026** — The Gemini API began rejecting requests from unrestricted standard keys. Standard keys with an explicit restriction applied (for example, locked to the Gemini API only) kept working past this date — this was a grace period, not the full cutoff.

4. **September 2026** — The full cutoff. All standard keys, restricted or not, are rejected. Only auth keys continue to work.

If you're still authenticating with an AIza key today and it's working, that's the grace period — not a sign you're safe long-term.

# **Migrating to an Auth Key**

The move itself is straightforward:

1. Open the **API Keys** page in Google AI Studio and check the **Key Type** column to find any keys still listed as Standard.

2. Click **Create API key**. Any new key generated now is automatically issued as an auth key — there's no toggle to choose the old format.

3. Copy the new key and update your application code, environment variables, and deployment configs. The variable names haven't changed — GEMINI_API_KEY or GOOGLE_API_KEY still work the same way.

4. **Test the exact integration path you actually use**, not just a raw call to the native endpoint.

5. Once the new key is verified in production, revoke the old standard key rather than deleting it immediately, to avoid downtime if something was missed.

If you need more runway before migrating fully, you can restrict an existing standard key instead: in AI Studio, find keys marked Unrestricted, choose Add restrictions, then Restrict to Gemini API only. This buys time past the June cutoff, but not past September — restricted standard keys are cut off too.

# **The Catch: Auth Keys Break Some Integrations**

This is the part tripping up developers right now. Auth keys work fine against Gemini's native endpoints, but a number of third-party tools, SDKs, and OpenAI-compatible wrappers were written with the assumption that a Gemini key always starts with AIza. Point one of those integrations at a new AQ. key and it can fail outright — commonly surfacing as a 401 Unauthorized, or, on OpenAI-compatible routes specifically, a 400: Multiple authentication credentials received.

The key itself usually isn't the problem in these cases. The failure tends to sit in whatever middleware, gateway, or client library is validating the key format before the request ever reaches Google. If you maintain a tool that talks to Gemini, it's worth explicitly testing an AQ. key end-to-end rather than assuming a drop-in replacement will behave identically to the old format.

# **The Bottom Line**

The AIza prefix that's been synonymous with Google API keys for over a decade is being retired for Gemini specifically, replaced by service-account-bound AQ. auth keys with tighter default scoping and faster leak response. The change is security-motivated, not cosmetic, and it's already partially enforced. If any part of your stack — your own code, a dependency, or a third-party tool — still assumes the old format, September 2026 is the hard deadline to have it sorted out.