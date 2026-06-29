Title: How to Publish Your Chrome Extension on the Chrome Web Store
Date: 2026-06-29
Category: Article
Tags: google, web-store, steps, chrome-extension, extensions
Slug: publish-your-chrome-extension-on-the-chrome-web-store


You built the thing. Now let's get it in front of people.

So you've built a Chrome extension. It works on your machine, your friends think it's cool, and you're ready to share it with the world. The Chrome Web Store is where you do that — it's Google's official marketplace for extensions, and getting listed there gives you access to millions of Chrome users.

The process is straightforward once you know the steps. Here's everything you need to go from a local folder of files to a live, published extension.


# **Before You Start: What You'll Need**

Before touching the Chrome Web Store, make sure you have three things ready:

A **Google Account** — preferably one dedicated to development. You'll receive policy updates, review notifications, and potential takedown warnings here, so using a separate account keeps things clean.

A **working extension** — not just "it kind of works," but thoroughly tested. Once you submit for review, Google's team will poke around your extension. Bugs that slip through reflect poorly on your listing and can get you rejected.

**Required assets** — at minimum, a 128×128 pixel icon. You'll also want screenshots (1280×800 px) for your store listing. These are often the first thing a user sees, so they matter more than developers tend to think.


**Step 1: Register as a Developer**

Head to the Chrome Web Store Developer Dashboard and sign in with your Google account. The first time you visit, you'll be asked to agree to the Developer Agreement and pay a one-time registration fee of **$5 USD.**

That's it — five dollars, once, for your entire developer account. It covers every extension you'll ever publish under that account, with no recurring fees. The fee exists mainly to reduce spam and low-quality submissions.

Once registered, complete your account profile: set your publisher name (this appears under every extension you publish), verify your contact email, and fill in any required business details.


**Step 2: Test Your Extension Locally**

This step sounds obvious, but it's worth being deliberate about. Open Chrome, navigate to chrome://extensions/, and flip on **Developer mode** in the top-right corner. Click **Load unpacked** and select your extension folder.

Now test everything. Click every button. Trigger every feature. Open DevTools (F12) and watch the console for errors. If your extension interacts with specific websites, test it on those sites. If it has settings, change them and confirm they persist.

One critical check: open your manifest.json and confirm it says "manifest_version": 3. The Chrome Web Store no longer accepts Manifest V2 extensions. If yours is still on V2, you'll need to migrate before proceeding — Google's migration guide covers the key differences.

Also audit your permissions. Look at every permission listed in your manifest and ask honestly: does the extension actually use this? Unnecessary permissions are a red flag for reviewers and a trust signal for users. Only request what you genuinely need.


**Step 3: Package Your Extension as a ZIP**

The Chrome Web Store only accepts ZIP file uploads — not raw folders, not CRX files. Packaging correctly is simple but easy to get wrong.

First, clean up your extension folder. Remove anything that doesn't belong in production: .git directories, node_modules, test files, .DS_Store files, and any temporary scripts. The folder should contain only what the extension needs to run.

Then, make sure your manifest.json is at the root level of the folder — not nested inside a subfolder. If Chrome opens your ZIP and can't find manifest.json immediately, the upload will fail.

Once the folder is clean, compress it into a ZIP. On Mac, right-click → Compress. On Windows, right-click → Send to → Compressed folder. On Linux, zip -r my-extension.zip my-extension/.

One more thing: set your version number in manifest.json to something low, like 0.0.1. Every future update you push must have a higher version number than the previous one, so give yourself room to grow.


**Step 4: Upload to the Developer Dashboard**

Go to the Chrome Developer Dashboard, sign in, and click **Add new item**. You'll be prompted to upload your ZIP file.

If your manifest is valid and the ZIP is structured correctly, you'll be taken to the item edit page. If not, the dashboard will tell you what went wrong — usually a malformed manifest or a missing required field.

At this point, your extension exists in the dashboard as a draft. You can upload new ZIPs as many times as you want before submitting for review, so don't treat this upload as the point of no return.


**Step 5: Fill Out Your Store Listing**

This is the part most developers underestimate. Your store listing is your marketing page, and a weak one means fewer installs — even if your extension is excellent.

Work through each tab in the dashboard:

**Store Listing** — Write a clear, honest title and description. The description should explain what your extension does, why someone would want it, and how it works. Use plain language, not jargon. Upload your icon and at least one screenshot. Show the extension in action, not just a blank UI.

**Privacy** — Declare your extension's single purpose (Google requires this) and explain how your extension handles user data. If your extension runs entirely locally and collects nothing, say so clearly. If it does collect data, be specific and accurate — reviewers cross-reference these declarations with your code.

**Distribution** — Choose your visibility: Public (anyone can find and install it), Unlisted (only people with the direct link), or Private (restricted to your domain or trusted testers). You can also control which countries your extension is available in.

If your extension collects any user data, you must provide a working Privacy Policy URL. Host it on your own site or link to a relevant file in your GitHub repo — it just needs to be publicly accessible.


**Step 6: Submit for Review**

When all tabs are filled out and the Submit for Review button becomes active, you're ready. Click it.

A confirmation dialog will appear asking whether you want to publish automatically once the review passes, or publish manually at a time of your choosing. If you want to coordinate a launch — an announcement post, a tweet, a Product Hunt submission — choose manual publishing so you control the timing.

After submission, your extension enters Google's review queue. Simple extensions with minimal permissions can be approved within a few hours. Extensions requesting sensitive permissions (like access to all URLs, or use of remote code) may take days and could require additional justification.

You'll receive an email when your extension is approved, or if there's an issue that needs to be addressed. Once approved, you have **30 days** to publish it — after that, the submission reverts to a draft and you'll need to resubmit.


# **After You're Live**

Publishing is the beginning, not the end.

**Read your reviews**. Users will tell you exactly what's broken, what's confusing, and what they wish your extension did. The Developer Dashboard lets you respond publicly, which signals to prospective users that you're engaged and maintaining the project.

**Push updates regularly**. When you upload a new version, increment the version number in manifest.json, re-zip, and upload through the same dashboard. Updates go through the same review process, though they're typically faster than initial submissions.

**Watch for policy changes**. Google updates the Chrome Web Store Developer Program Policies periodically. An extension that's compliant today might need adjustments tomorrow. Your developer account email is where you'll hear about this.

**Use the analytics**. The dashboard gives you install counts, active user metrics, and ratings over time. These numbers tell you whether your extension is retaining users or losing them — and retention is the real signal of whether you've built something people genuinely value.


# **Common Reasons for Rejection**

It helps to know what gets extensions rejected so you can avoid those pitfalls:

1. **Requesting unnecessary permissions** — only ask for what you actually use
2. **Vague or misleading descriptions** — be specific about what your extension does
3. **Missing or inaccessible privacy policy** — required if you handle any user data
4. **Using Manifest V2** — Manifest V3 is mandatory
5. **Remote code execution** — loading and running code from external URLs is not allowed
6. **Poor user experience** — extensions that crash, produce errors, or behave unexpectedly


# **The Bottom Line**

Publishing a Chrome extension takes maybe an hour the first time — most of that is filling out the store listing carefully. The $5 registration fee is the only cost, and from there, you can publish as many extensions as you like (up to 20 live at once per account).

The review process exists to protect users, and it's worth treating seriously. A well-prepared submission — clean code, honest permissions, a clear listing — sails through. A rushed one gets rejected and costs you time.

Build something useful, ship it properly, and then let the Chrome Web Store do what it's designed to do: put your work in front of the people who need it.