Title: Two Doors, One House: Connecting GitHub and Google Sheets Separately in an Electron App
Date: 2026-09-05
Category: Article
Tags: electron, github-api, google-sheets-api, oauth,desktop-apps, javascript, nodejs, integrations
Slug: connecting-github-google-sheets-electron

Desktop tools built with Electron often need to talk to more than one external service, and it's tempting to treat those integrations as one big "connect everything" feature. In practice, GitHub and Google Sheets are better thought of as two separate doors into the same house: each has its own authentication flow, its own SDK, its own rate limits, and its own failure modes. Wiring them up independently — rather than forcing them through a shared abstraction too early — keeps your code easier to debug and your users' permissions easier to reason about.

This article walks through connecting both services to an Electron app as standalone integrations: separate credentials, separate auth windows, separate stored tokens, and separate API clients that happen to live in the same app.

# **Why keep them separate**

A few reasons this separation pays off:

1. **Different auth models**. GitHub supports OAuth Apps, GitHub Apps, and personal access tokens. Google Sheets requires an OAuth 2.0 client through Google Cloud Console with specific scopes. Trying to abstract these into one "connector" interface too early usually leaks details anyway.

2. **Independent failure and revocation**. A user might disconnect Google Sheets but keep GitHub connected, or vice versa. If your storage and refresh logic are separate, revoking one doesn't risk breaking the other.

3. **Least privilege**. You only request the scopes each feature actually needs, and you can prompt for each connection at the moment it's relevant instead of front-loading a wall of permissions at first launch.

# **Architecture overview**

In Electron, the main process is the right place to handle OAuth redirects and token storage, since it has access to Node APIs and can open a BrowserWindow for the consent screen. The renderer process should never hold raw tokens or client secrets — it should ask the main process to perform actions ("fetch my repos," "append this row") over IPC.

    Renderer (UI)  --IPC-->  Main process  --HTTPS-->  GitHub API / Google Sheets API
                                |
                            secure token storage
                        (keytar / electron-store)

Use keytar (or safeStorage from Electron itself) to keep tokens out of plaintext files. electron-store is fine for non-sensitive settings, but pair it with safeStorage.encryptString / decryptString if you store anything sensitive there.

bash

    npm install keytar octokit googleapis

# **Part 1 — Connecting GitHub**

**Choosing an auth method**

For a desktop app, the two realistic options are:

1. **OAuth App with a loopback redirect** — best user experience, no manual token pasting.

2. **Personal Access Token (PAT)** — simplest to implement, fine for internal tools or power users, but pushes token creation onto the user.

The OAuth flow is worth the extra setup for anything user-facing. Register an OAuth App at github.com/settings/developers, and set the callback URL to a loopback address like http://127.0.0.1:43117/callback.

**Main process: handling the GitHub OAuth flow**

js

    // main/github-auth.js
    const { BrowserWindow } = require('electron');
    const http = require('http');
    const keytar = require('keytar');

    const CLIENT_ID = process.env.GITHUB_CLIENT_ID;
    const CLIENT_SECRET = process.env.GITHUB_CLIENT_SECRET;
    const REDIRECT_PORT = 43117;

    function connectGitHub() {
    return new Promise((resolve, reject) => {
        const server = http.createServer(async (req, res) => {
        const url = new URL(req.url, `http://127.0.0.1:${REDIRECT_PORT}`);
        const code = url.searchParams.get('code');

        if (code) {
            res.end('GitHub connected. You can close this window.');
            server.close();

            const tokenRes = await fetch('https://github.com/login/oauth/access_token', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json', Accept: 'application/json' },
            body: JSON.stringify({
                client_id: CLIENT_ID,
                client_secret: CLIENT_SECRET,
                code,
                redirect_uri: `http://127.0.0.1:${REDIRECT_PORT}/callback`,
            }),
            });
            const { access_token } = await tokenRes.json();

            await keytar.setPassword('myapp', 'github-token', access_token);
            authWindow.close();
            resolve(access_token);
        }
        });

        server.listen(REDIRECT_PORT);

        const authWindow = new BrowserWindow({ width: 500, height: 700 });
        const scope = 'repo read:user';
        authWindow.loadURL(
        `https://github.com/login/oauth/authorize?client_id=${CLIENT_ID}&scope=${scope}&redirect_uri=http://127.0.0.1:${REDIRECT_PORT}/callback`
        );

        authWindow.on('closed', () => {
        server.close();
        reject(new Error('GitHub auth window closed before completing.'));
        });
    });
    }

    module.exports = { connectGitHub };

**Using the token**

Once stored, retrieve it and drive requests through Octokit:

js

    // main/github-client.js
    const { Octokit } = require('octokit');
    const keytar = require('keytar');

    async function getGitHubClient() {
    const token = await keytar.getPassword('myapp', 'github-token');
    if (!token) throw new Error('GitHub is not connected.');
    return new Octokit({ auth: token });
    }

    async function listRepos() {
    const octokit = await getGitHubClient();
    const { data } = await octokit.rest.repos.listForAuthenticatedUser({ per_page: 50 });
    return data.map((r) => ({ name: r.full_name, private: r.private, url: r.html_url }));
    }

    module.exports = { getGitHubClient, listRepos };

Expose this to the renderer through a narrow IPC surface:

js

    // main/index.js
    ipcMain.handle('github:connect', () => connectGitHub());
    ipcMain.handle('github:list-repos', () => listRepos());

# **Part 2 — Connecting Google Sheets**

Google's flow is conceptually similar but has its own quirks: you need a project in Google Cloud Console, an OAuth 2.0 Client ID of type "Desktop app," and the Sheets API enabled.

**Setting up the client**

js

    // main/sheets-auth.js
    const { google } = require('googleapis');
    const { BrowserWindow } = require('electron');
    const http = require('http');
    const keytar = require('keytar');

    const oAuth2Client = new google.auth.OAuth2(
    process.env.GOOGLE_CLIENT_ID,
    process.env.GOOGLE_CLIENT_SECRET,
    'http://127.0.0.1:43118/callback'
    );

    function connectGoogleSheets() {
    return new Promise((resolve, reject) => {
        const authUrl = oAuth2Client.generateAuthUrl({
        access_type: 'offline',
        scope: ['https://www.googleapis.com/auth/spreadsheets'],
        prompt: 'consent',
        });

        const server = http.createServer(async (req, res) => {
        const url = new URL(req.url, 'http://127.0.0.1:43118');
        const code = url.searchParams.get('code');

        if (code) {
            res.end('Google Sheets connected. You can close this window.');
            server.close();

            const { tokens } = await oAuth2Client.getToken(code);
            await keytar.setPassword('myapp', 'google-tokens', JSON.stringify(tokens));
            authWindow.close();
            resolve(tokens);
        }
        });

        server.listen(43118);

        const authWindow = new BrowserWindow({ width: 500, height: 700 });
        authWindow.loadURL(authUrl);

        authWindow.on('closed', () => {
        server.close();
        reject(new Error('Google auth window closed before completing.'));
        });
    });
    }

    module.exports = { connectGoogleSheets, oAuth2Client };

Note the access_type: 'offline' and prompt: 'consent' — without these, you won't reliably get a refresh token, and your users will be re-prompted to log in every hour.

**Reading and writing rows**

js

    // main/sheets-client.js
    const { google } = require('googleapis');
    const keytar = require('keytar');
    const { oAuth2Client } = require('./sheets-auth');

    async function getSheetsClient() {
    const raw = await keytar.getPassword('myapp', 'google-tokens');
    if (!raw) throw new Error('Google Sheets is not connected.');
    oAuth2Client.setCredentials(JSON.parse(raw));
    return google.sheets({ version: 'v4', auth: oAuth2Client });
    }

    async function appendRow(spreadsheetId, range, values) {
    const sheets = await getSheetsClient();
    return sheets.spreadsheets.values.append({
        spreadsheetId,
        range,
        valueInputOption: 'USER_ENTERED',
        requestBody: { values: [values] },
    });
    }

    module.exports = { getSheetsClient, appendRow };

js

    // main/index.js
    ipcMain.handle('sheets:connect', () => connectGoogleSheets());
    ipcMain.handle('sheets:append-row', (_, { spreadsheetId, range, values }) =>
    appendRow(spreadsheetId, range, values)
    );

Google's access tokens expire in about an hour; the googleapis client will use the stored refresh token automatically as long as oAuth2Client.setCredentials includes it, so re-save the credentials whenever the library emits a tokens event with a refreshed access_token.

# **Keeping the two integrations independent in the UI**

In the renderer, treat "GitHub connected" and "Sheets connected" as two unrelated booleans, each with its own connect/disconnect button:

js

    // renderer/connections.js
    async function refreshConnectionStatus() {
        const githubConnected = await window.api.invoke('github:status');
        const sheetsConnected = await window.api.invoke('sheets:status');

        document.getElementById('github-status').textContent = githubConnected ? 'Connected' : 'Not connected';
        document.getElementById('sheets-status').textContent = sheetsConnected ? 'Connected' : 'Not connected';
    }

Add a small status handler on each side that just checks whether a token exists in keytar, and a disconnect handler that deletes it. Resist the urge to build a shared "IntegrationManager" class until you actually have a third or fourth service — with only two, a shared abstraction usually costs more clarity than it saves.

# **Security checklist**

1. Never commit CLIENT_SECRET values; load them from environment variables or a build-time config that isn't checked into source control.

2. Store tokens with keytar or Electron's safeStorage, not in plain JSON files.

3. Request the narrowest scopes that work (repo vs. full account access on GitHub; a single spreadsheet scope vs. Drive-wide access on Google where possible).

4. Validate redirect URIs match exactly what's registered with each provider — loopback ports are your friend for avoiding custom protocol handler complexity.

5. Handle disconnect explicitly: revoke server-side where the API supports it (GitHub's DELETE /applications/{client_id}/token, Google's oauth2.revoke) rather than just deleting the local token.

# **Wrapping up**

Connecting GitHub and Google Sheets to an Electron app doesn't require a unified integrations framework on day one. Two loopback OAuth servers, two token stores under different keytar keys, and two thin API clients get you a long way, and they fail independently in ways that are easy to debug. Once you have a third integration on the horizon, that's the right time to look for the shared shape between them — not before.