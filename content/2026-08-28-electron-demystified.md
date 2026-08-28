Title: Electron Demystified: How to Build Desktop Apps with Web Technology
Date: 2026-08-28
Category: Article
Tags: Electron, JavaScript, Desktop Development, Node.js, Cross-Platform Development, Web Development, Software Engineering, Programming, Chromium, App Development
Slug: how-to-build-desktop-apps-with-web-tech

If you've ever used Visual Studio Code, Slack, Discord, or Figma's desktop app, you've already used Electron — even if you didn't know it. Electron lets developers build cross-platform desktop applications using nothing but HTML, CSS, and JavaScript. No native Swift, no C++, no platform-specific SDKs required.

This post walks through what Electron actually is, how it works under the hood, why teams choose it, and where it falls short.

# **What Is Electron?**

Electron is an open-source framework, originally created by GitHub (for building Atom, the code editor), that combines two things:

1. **Chromium** — the open-source browser engine behind Google Chrome, which renders your UI.

2. **Node.js** — a JavaScript runtime that gives your app access to the file system, operating system, and native modules.

By fusing these two together, Electron lets you write a desktop app the same way you'd write a web page, but with full access to things a browser normally blocks for security reasons — like reading files, spawning system processes, or accessing hardware.

# **The Core Architecture: Main Process vs. Renderer Process**

The most important concept to understand in Electron is the split between two types of processes.

**The Main Process**

Every Electron app has exactly one main process. This is essentially a Node.js script that:

1. Creates and manages application windows (BrowserWindow instances)
2. Controls the app lifecycle (launch, quit, minimize, etc.)
3. Has full access to Node.js APIs and native OS features
4. Acts as the "backend" of your desktop app

**The Renderer Process**

Each window you open in Electron runs in its own **renderer process**. This is where your actual UI lives — it's essentially a Chromium browser tab rendering your HTML/CSS/JS. By default, renderer processes are sandboxed and don't have direct access to Node.js or OS-level APIs, for security reasons.

**Inter-Process Communication (IPC)**

Since the main process and renderer processes are isolated from each other, they need a way to talk. This is done through **IPC (Inter-Process Communication)**, typically using:

1. ipcMain (in the main process)
2. ipcRenderer (in the renderer process)
3. A preload script, which acts as a secure bridge, exposing only specific, whitelisted functionality to the renderer via contextBridge

A typical flow looks like this:

    Renderer (UI) → ipcRenderer.invoke('save-file', data)
            ↓
    Preload script (contextBridge exposes safe API)
            ↓
    Main process → ipcMain.handle('save-file', ...) → writes to disk
            ↓
    Result sent back to renderer

This separation exists so that a compromised or buggy webpage can't directly reach into your file system — it has to go through a controlled gate.

# **A Minimal Electron App**

Here's what a bare-bones Electron app looks like:

**main.js** (Main process)

javascript

    const { app, BrowserWindow } = require('electron');

    function createWindow() {
         const win = new BrowserWindow({
            width: 800,
            height: 600,
            webPreferences: {
                preload: `${__dirname}/preload.js`
            }
        });

        win.loadFile('index.html');
    }

    app.whenReady().then(createWindow);

**preload.js** (Bridge)

javascript

    const { contextBridge, ipcRenderer } = require('electron');

    contextBridge.exposeInMainWorld('api', {
        getVersion: () => ipcRenderer.invoke('get-app-version')
    });

**index.html** (Renderer)

html

    <!DOCTYPE html>
    <html>
        <body>
            <h1>Hello Electron!</h1>
            <script>
                window.api.getVersion().then(v => console.log(v));
            </script>
        </body>
    </html>

That's genuinely enough to launch a real, cross-platform desktop window.

# **Why Developers Choose Electron**

1. Write once, ship everywhere — the same codebase runs on Windows, macOS, and Linux.

2. Reuse web skills — if your team knows React, Vue, or plain JS/CSS, they can build desktop apps immediately.

3. Rich ecosystem — full access to npm packages, plus Node's native modules for OS-level tasks.

4. Fast prototyping — going from idea to a working desktop app is much faster than with native toolkits.

# **The Trade-offs**

Electron isn't free of downsides, and they're worth knowing before you commit:

1. **Bundle size** — every Electron app ships its own copy of Chromium and Node.js, so even simple apps can be 100+ MB.

2. **Memory usage** — each window is essentially a Chrome tab, so RAM usage adds up quickly compared to native apps.

3. **Performance ceiling** — CPU-intensive or highly graphical tasks may run better in a native framework.

4. **Security responsibility** — features like nodeIntegration, context isolation, and IPC channels must be configured carefully, or you open the door to remote code execution vulnerabilities.

# **Where Electron Shines**

Electron tends to be the right choice when:

1. You need a desktop presence for a product that's primarily web-based

2. Developer velocity and cross-platform reach matter more than binary size

3. Your app is UI/workflow heavy rather than performance-critical (e.g., productivity tools, chat apps, code editors, internal tools)

If you're building something like a game engine or a real-time video editor, a native framework (or something like Tauri, which uses the OS's built-in webview instead of bundling Chromium) might serve you better.

# **Wrapping Up**

Electron's genius is architectural: it takes the browser's rendering engine and Node's system access, and stitches them together with a secure messaging layer. Once you understand the main process/renderer process split and how IPC bridges them, the rest of Electron development feels a lot like ordinary web development — just with superpowers.

