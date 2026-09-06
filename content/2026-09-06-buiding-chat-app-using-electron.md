Title: Desktop AI, Your Way: Building a Claude/ChatGPT-Style Chat App with Electron
Date: 2026-09-06
Category: Article
Tags: electron, ai-chatbot, desktop-apps, javascript, llm-integration, nodejs, react, tutorial
Slug: building-a-desktop-ai-chat-app-with-electron


Chat interfaces like Claude and ChatGPT feel deceptively simple: a message box, a scrolling conversation, a streaming response. But under the hood, there's a real desktop application architecture at work — one you can absolutely build yourself. This article walks through creating a native-feeling AI chat application using Electron, covering the core architecture, streaming responses, conversation persistence, and packaging for distribution.

# **Why Electron for an AI Chat App?**

Electron lets you ship a single JavaScript/HTML/CSS codebase as a native app for Windows, macOS, and Linux. For an AI chat client specifically, it's a strong fit because:

1. You get a real OS-level window, system tray, native menus, and notifications — things a browser tab can't offer.

2. You can securely store API keys and conversation history on disk instead of relying on browser storage.

3. You can make direct HTTPS requests to LLM providers (Anthropic, OpenAI, etc.) from a Node.js backend process without CORS restrictions.

4. The UI layer is just web tech, so any frontend framework (React, Vue, or plain JS) works.

# **Core Architecture**

A well-structured Electron AI chat app splits cleanly into three layers:

1. **Main process (Node.js)** — owns the app lifecycle, creates windows, handles filesystem access, and makes API calls to the LLM provider.

2. **Renderer process (Chromium)** — the actual chat UI: message list, input box, sidebar of conversations.

3. **Preload script** — a secure bridge between the two, exposing only specific, whitelisted functions to the renderer via contextBridge.

This separation matters for security. You never want your renderer (which loads like a webpage) to have direct access to Node.js APIs or your API key — that's how you end up with remote-code-execution vulnerabilities if any content injection ever occurs.

**Project Setup**

bash

    mkdir ai-chat-desktop && cd ai-chat-desktop
    npm init -y
    npm install electron electron-store --save-dev
    npm install node-fetch

Your folder structure will look roughly like:

    ai-chat-desktop/
    ├── package.json
    ├── main.js          # Main process
    ├── preload.js        # Secure bridge
    ├── renderer/
    │   ├── index.html
    │   ├── renderer.js
    │   └── styles.css

# **Step 1: The Main Process**

The main process creates the browser window and handles all "backend" work: talking to the LLM API and reading/writing conversation history to disk.

javascript

    // main.js
    const { app, BrowserWindow, ipcMain } = require('electron');
    const path = require('path');
    const Store = require('electron-store');

    const store = new Store(); // persists data to disk as JSON

    function createWindow() {
    const win = new BrowserWindow({
        width: 1000,
        height: 700,
        webPreferences: {
        preload: path.join(__dirname, 'preload.js'),
        contextIsolation: true,
        nodeIntegration: false, // critical: never expose Node.js to the renderer directly
        },
    });

    win.loadFile('renderer/index.html');
    }

    app.whenReady().then(createWindow);

    app.on('window-all-closed', () => {
    if (process.platform !== 'darwin') app.quit();
    });

# **Step 2: The Preload Script — Your Security Boundary**

The preload script exposes a minimal, controlled API to the renderer. This is the single most important security decision in the whole app.

javascript

    // preload.js
    const { contextBridge, ipcRenderer } = require('electron');

    contextBridge.exposeInMainWorld('chatAPI', {
    sendMessage: (conversationId, messages) =>
        ipcRenderer.invoke('send-message', conversationId, messages),
    onStreamChunk: (callback) =>
        ipcRenderer.on('stream-chunk', (_event, chunk) => callback(chunk)),
    loadConversations: () => ipcRenderer.invoke('load-conversations'),
    saveConversation: (conv) => ipcRenderer.invoke('save-conversation', conv),
    });

The renderer can now call window.chatAPI.sendMessage(...) — but it has zero direct access to fetch, the filesystem, or your API key.

# **Step 3: Calling the LLM API with Streaming**

Streaming is what makes a chat app feel alive — tokens appearing progressively rather than the whole reply landing at once. Here's a main-process handler using the Anthropic API as an example (the same pattern applies to OpenAI or other providers, swapping endpoint and payload shape):

javascript

    // main.js (continued)
    const fetch = require('node-fetch');

    ipcMain.handle('send-message', async (event, conversationId, messages) => {
    const response = await fetch('https://api.anthropic.com/v1/messages', {
        method: 'POST',
        headers: {
        'Content-Type': 'application/json',
        'x-api-key': store.get('apiKey'), // stored securely, never in renderer
        'anthropic-version': '2023-06-01',
        },
        body: JSON.stringify({
        model: 'claude-sonnet-4-6',
        max_tokens: 1024,
        messages,
        stream: true,
        }),
    });

    const reader = response.body;
    reader.on('data', (chunk) => {
        const lines = chunk.toString().split('\n').filter((l) => l.startsWith('data:'));
        for (const line of lines) {
        const payload = line.replace('data:', '').trim();
        if (payload === '[DONE]') continue;
        try {
            const parsed = JSON.parse(payload);
            if (parsed.delta?.text) {
            event.sender.send('stream-chunk', parsed.delta.text);
            }
        } catch {
            // ignore incomplete JSON fragments mid-stream
        }
        }
    });

    return { started: true };
    });

Handle your API key storage carefully — electron-store writes to a JSON file on disk by default, so for anything beyond a personal prototype, encrypt the key at rest or use the OS keychain via a library like keytar.

# **Step 4: The Chat UI**

The renderer doesn't need a heavy framework, but React (or Vue) makes managing message state and re-renders much easier as the conversation grows. A minimal vanilla JS version:

javascript

    // renderer/renderer.js
    const messagesEl = document.getElementById('messages');
    const inputEl = document.getElementById('input');
    const sendBtn = document.getElementById('send');

    let currentMessages = [];
    let assistantBuffer = '';

    window.chatAPI.onStreamChunk((chunk) => {
    assistantBuffer += chunk;
    renderStreamingMessage(assistantBuffer);
    });

    sendBtn.addEventListener('click', async () => {
    const text = inputEl.value.trim();
    if (!text) return;

    currentMessages.push({ role: 'user', content: text });
    appendMessage('user', text);
    inputEl.value = '';
    assistantBuffer = '';

    await window.chatAPI.sendMessage('default', currentMessages);
    });

    function appendMessage(role, content) {
    const div = document.createElement('div');
    div.className = `message ${role}`;
    div.textContent = content;
    messagesEl.appendChild(div);
    messagesEl.scrollTop = messagesEl.scrollHeight;
    }

    function renderStreamingMessage(text) {
    let el = document.getElementById('streaming-msg');
    if (!el) {
        el = document.createElement('div');
        el.id = 'streaming-msg';
        el.className = 'message assistant';
        messagesEl.appendChild(el);
    }
    el.textContent = text;
    messagesEl.scrollTop = messagesEl.scrollHeight;
    }

# **Step 5: Persisting Conversations**

To mimic Claude's or ChatGPT's sidebar of past conversations, store each thread as a record keyed by ID:

javascript

    ipcMain.handle('save-conversation', (event, conversation) => {
    const all = store.get('conversations', []);
    const idx = all.findIndex((c) => c.id === conversation.id);
    if (idx >= 0) all[idx] = conversation;
    else all.push(conversation);
    store.set('conversations', all);
    return true;
    });

    ipcMain.handle('load-conversations', () => {
    return store.get('conversations', []);
    });

For larger histories or full-text search across conversations, swap electron-store for a lightweight embedded database like SQLite (via better-sqlite3), which scales much better than a single JSON blob.

# **Design Details That Make It Feel Native**

A few touches separate a "webpage in a window" from something that feels like a real desktop AI assistant:

1. **Native window controls and menus** via Electron's Menu module (New Chat, Settings, Quit).

2. **Global keyboard shortcuts** (e.g., Cmd/Ctrl+N for a new conversation) using globalShortcut or app-level accelerators.

3. **System tray integration** so the app can run in the background and be summoned instantly.

4. **Markdown and code-block** rendering in responses, typically with marked or react-markdown plus a syntax highlighter like highlight.js or shiki.

5. **Auto-updates** via electron-updater, so users get new versions without manual reinstalls.

# **Packaging and Distribution**

Once the app works locally, package it with electron-builder:

bash

    npm install electron-builder --save-dev

json

    // package.json
    "build": {
    "appId": "com.yourname.aichat",
    "mac": { "target": "dmg" },
    "win": { "target": "nsis" },
    "linux": { "target": "AppImage" }
    },
    "scripts": {
    "dist": "electron-builder"
    }

Running npm run dist produces installers for each target platform.

# **Security and Cost Checklist**

Before shipping, run through this list:

1. contextIsolation: true and nodeIntegration: false are non-negotiable defaults.

2. Never let the renderer hold or transmit the raw API key.

3. Sanitize any rendered Markdown/HTML from model output to avoid injection when displaying it in the DOM.

4. Add request timeouts and error handling around the API call — network failures shouldn't crash the stream.

5. Track token usage per conversation if you're billing users or capping usage, since LLM API costs scale with tokens, not messages.

# **Wrapping Up**

An Electron-based AI chat app boils down to three well-separated concerns: a secure main process that talks to the model API, a preload bridge that limits what the UI can touch, and a renderer focused purely on presenting the conversation. Once that skeleton is in place, everything else — themes, plugins, multi-model support, voice input — is additive. The architecture in this article is the same shape used by production desktop AI clients; scaling it up is a matter of refinement, not redesign.