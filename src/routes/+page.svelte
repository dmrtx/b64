<script lang="ts">
    import Prism from "prismjs";
    import "prismjs/components/prism-json";
    import { onMount } from "svelte";

    let textInput = $state("");
    let b64Input = $state("");
    let textError = $state("");
    let b64Error = $state("");
    let textCopied = $state(false);
    let b64Copied = $state(false);
    let isJson = $state(false);

    let textEditor: HTMLTextAreaElement;
    let textHighlight: HTMLElement;
    let b64Editor: HTMLTextAreaElement;

    function detectJson(text: string) {
        if (!text.trim()) return false;
        try {
            JSON.parse(text);
            return true;
        } catch (e) {
            return false;
        }
    }

    function highlight(text: string, isJsonContent: boolean) {
        if (!text) return "<br>";
        if (isJsonContent) {
            return Prism.highlight(text, Prism.languages.json, "json") + "<br>";
        }
        // Basic escaping for plain text to keep font metrics consistent with overlay
        return (
            text.replace(/[&<>"']/g, function (m) {
                return (
                    {
                        "&": "&amp;",
                        "<": "&lt;",
                        ">": "&gt;",
                        '"': "&quot;",
                        "'": "&#039;",
                    }[m] || m
                );
            }) + "<br>"
        );
    }

    function encodeToBase64() {
        textError = "";
        isJson = detectJson(textInput);

        if (!textInput) {
            b64Input = "";
            return;
        }
        try {
            b64Input = btoa(unescape(encodeURIComponent(textInput)));
        } catch (e) {
            textError = "Failed to encode text";
        }
    }

    function decodeFromBase64() {
        b64Error = "";
        if (!b64Input) {
            textInput = "";
            isJson = false;
            return;
        }
        try {
            textInput = decodeURIComponent(escape(atob(b64Input)));
            isJson = detectJson(textInput);
        } catch (e) {
            b64Error = "Invalid Base64 string";
        }
    }

    async function copyToClipboard(text: string, type: "text" | "b64") {
        try {
            await navigator.clipboard.writeText(text);
            if (type === "text") {
                textCopied = true;
                setTimeout(() => (textCopied = false), 2000);
            } else {
                b64Copied = true;
                setTimeout(() => (b64Copied = false), 2000);
            }
        } catch (e) {
            console.error("Failed to copy:", e);
        }
    }

    function clearAll() {
        textInput = "";
        b64Input = "";
        textError = "";
        b64Error = "";
        isJson = false;
    }

    function handleTextInput(e: Event) {
        textInput = (e.target as HTMLTextAreaElement).value;
        encodeToBase64();
    }

    function handleB64Input(e: Event) {
        b64Input = (e.target as HTMLTextAreaElement).value;
        decodeFromBase64();
    }

    function syncScroll(source: HTMLTextAreaElement, target: HTMLElement) {
        if (target) {
            target.scrollTop = source.scrollTop;
            target.scrollLeft = source.scrollLeft;
        }
    }
</script>

<svelte:head>
    <title>B64 Converter - Professional Tool</title>
    <meta
        name="description"
        content="Professional Base64 converter. Split-view interface, real-time conversion, JSON syntax highlighting."
    />
    <link
        href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&family=Fira+Code&display=swap"
        rel="stylesheet"
    />
</svelte:head>

<div class="app">
    <header>
        <div class="logo">
            <span class="logo-text">B<span class="accent">64</span></span>
            <span class="divider">|</span>
            <span class="subtitle">Converter</span>
        </div>
        <div class="header-actions">
            <a
                href="https://github.com"
                target="_blank"
                rel="noopener"
                class="nav-link">GitHub</a
            >
        </div>
    </header>

    <main>
        <div class="editor-pane">
            <div class="toolbar">
                <div class="toolbar-title">
                    <svg
                        xmlns="http://www.w3.org/2000/svg"
                        width="16"
                        height="16"
                        viewBox="0 0 24 24"
                        fill="none"
                        stroke="currentColor"
                        stroke-width="2"
                        stroke-linecap="round"
                        stroke-linejoin="round"
                    >
                        <path
                            d="M17 3a2.85 2.83 0 1 1 4 4L7.5 20.5 2 22l1.5-5.5Z"
                        />
                    </svg>
                    <span>Plain Text</span>
                    {#if isJson}
                        <span class="badge">JSON</span>
                    {/if}
                </div>
                <div class="toolbar-actions">
                    <span class="stats">{textInput.length} chars</span>
                    <button
                        class="icon-btn"
                        onclick={() => copyToClipboard(textInput, "text")}
                        disabled={!textInput}
                        title="Copy Text"
                    >
                        {#if textCopied}
                            <svg
                                xmlns="http://www.w3.org/2000/svg"
                                width="16"
                                height="16"
                                viewBox="0 0 24 24"
                                fill="none"
                                stroke="currentColor"
                                stroke-width="2"
                                stroke-linecap="round"
                                stroke-linejoin="round"
                                class="success"
                            >
                                <polyline points="20 6 9 17 4 12" />
                            </svg>
                        {:else}
                            <svg
                                xmlns="http://www.w3.org/2000/svg"
                                width="16"
                                height="16"
                                viewBox="0 0 24 24"
                                fill="none"
                                stroke="currentColor"
                                stroke-width="2"
                                stroke-linecap="round"
                                stroke-linejoin="round"
                            >
                                <rect
                                    width="14"
                                    height="14"
                                    x="8"
                                    y="8"
                                    rx="2"
                                    ry="2"
                                />
                                <path
                                    d="M4 16c-1.1 0-2-.9-2-2V4c0-1.1.9-2 2-2h10c1.1 0 2 .9 2 2"
                                />
                            </svg>
                        {/if}
                    </button>
                </div>
            </div>
            <div class="editor-wrapper with-overlay">
                <pre aria-hidden="true" bind:this={textHighlight}><code
                        class="language-json"
                        >{@html highlight(textInput, isJson)}</code
                    ></pre>
                <textarea
                    bind:this={textEditor}
                    value={textInput}
                    oninput={handleTextInput}
                    onscroll={() => syncScroll(textEditor, textHighlight)}
                    placeholder="Type or paste text/JSON here..."
                    class:has-error={textError}
                    spellcheck="false"
                ></textarea>
                {#if textError}
                    <div class="error-toast">{textError}</div>
                {/if}
            </div>
        </div>

        <div class="center-controls">
            <div class="control-group">
                <button
                    class="control-btn primary"
                    onclick={clearAll}
                    title="Clear All"
                >
                    <svg
                        xmlns="http://www.w3.org/2000/svg"
                        width="20"
                        height="20"
                        viewBox="0 0 24 24"
                        fill="none"
                        stroke="currentColor"
                        stroke-width="2"
                        stroke-linecap="round"
                        stroke-linejoin="round"
                    >
                        <path d="M3 6h18" />
                        <path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6" />
                        <path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2" />
                    </svg>
                    <span>Clear</span>
                </button>
            </div>

            <div class="direction-indicator">
                <svg
                    xmlns="http://www.w3.org/2000/svg"
                    width="24"
                    height="24"
                    viewBox="0 0 24 24"
                    fill="none"
                    stroke="currentColor"
                    stroke-width="2"
                    stroke-linecap="round"
                    stroke-linejoin="round"
                >
                    <path d="M7 10h14l-4-4" />
                    <path d="M17 14H3l4 4" />
                </svg>
            </div>
        </div>

        <div class="editor-pane">
            <div class="toolbar">
                <div class="toolbar-title">
                    <svg
                        xmlns="http://www.w3.org/2000/svg"
                        width="16"
                        height="16"
                        viewBox="0 0 24 24"
                        fill="none"
                        stroke="currentColor"
                        stroke-width="2"
                        stroke-linecap="round"
                        stroke-linejoin="round"
                    >
                        <polyline points="16 18 22 12 16 6" />
                        <polyline points="8 6 2 12 8 18" />
                    </svg>
                    <span>Base64</span>
                </div>
                <div class="toolbar-actions">
                    <span class="stats">{b64Input.length} chars</span>
                    <button
                        class="icon-btn"
                        onclick={() => copyToClipboard(b64Input, "b64")}
                        disabled={!b64Input}
                        title="Copy Base64"
                    >
                        {#if b64Copied}
                            <svg
                                xmlns="http://www.w3.org/2000/svg"
                                width="16"
                                height="16"
                                viewBox="0 0 24 24"
                                fill="none"
                                stroke="currentColor"
                                stroke-width="2"
                                stroke-linecap="round"
                                stroke-linejoin="round"
                                class="success"
                            >
                                <polyline points="20 6 9 17 4 12" />
                            </svg>
                        {:else}
                            <svg
                                xmlns="http://www.w3.org/2000/svg"
                                width="16"
                                height="16"
                                viewBox="0 0 24 24"
                                fill="none"
                                stroke="currentColor"
                                stroke-width="2"
                                stroke-linecap="round"
                                stroke-linejoin="round"
                            >
                                <rect
                                    width="14"
                                    height="14"
                                    x="8"
                                    y="8"
                                    rx="2"
                                    ry="2"
                                />
                                <path
                                    d="M4 16c-1.1 0-2-.9-2-2V4c0-1.1.9-2 2-2h10c1.1 0 2 .9 2 2"
                                />
                            </svg>
                        {/if}
                    </button>
                </div>
            </div>
            <div class="editor-wrapper">
                <textarea
                    bind:this={b64Editor}
                    value={b64Input}
                    oninput={handleB64Input}
                    placeholder="Base64 output..."
                    class:has-error={b64Error}
                    spellcheck="false"
                ></textarea>
                {#if b64Error}
                    <div class="error-toast">{b64Error}</div>
                {/if}
            </div>
        </div>
    </main>
</div>

<style>
    :global(*) {
        box-sizing: border-box;
        margin: 0;
        padding: 0;
    }

    :global(body) {
        font-family:
            "Inter",
            -apple-system,
            BlinkMacSystemFont,
            "Segoe UI",
            Roboto,
            sans-serif;
        background: #0f172a;
        height: 100vh;
        color: #e2e8f0;
        overflow: hidden; /* Prevent body scroll, let app handle it */
    }

    .app {
        display: flex;
        flex-direction: column;
        height: 100vh;
    }

    header {
        height: 48px;
        background: #1e293b;
        border-bottom: 1px solid #334155;
        display: flex;
        align-items: center;
        padding: 0 1.5rem;
        justify-content: space-between;
        flex-shrink: 0;
    }

    .logo {
        display: flex;
        align-items: center;
        gap: 0.5rem;
        font-weight: 600;
    }

    .logo-text {
        font-size: 1.25rem;
        color: #fff;
    }

    .accent {
        color: #818cf8;
    }

    .divider {
        color: #475569;
    }

    .subtitle {
        color: #94a3b8;
        font-size: 0.875rem;
    }

    .nav-link {
        color: #94a3b8;
        text-decoration: none;
        font-size: 0.875rem;
        transition: color 0.2s;
    }

    .nav-link:hover {
        color: #fff;
    }

    main {
        flex: 1;
        display: flex;
        overflow: hidden;
        background: #0f172a;
    }

    .editor-pane {
        flex: 1;
        display: flex;
        flex-direction: column;
        min-width: 0; /* Allow shrinking */
        background: #0f172a;
    }

    .toolbar {
        height: 40px;
        background: #1e293b;
        border-bottom: 1px solid #334155;
        display: flex;
        align-items: center;
        justify-content: space-between;
        padding: 0 1rem;
        flex-shrink: 0;
    }

    .toolbar-title {
        display: flex;
        align-items: center;
        gap: 0.5rem;
        color: #cbd5e1;
        font-size: 0.875rem;
        font-weight: 500;
    }

    .toolbar-title svg {
        color: #818cf8;
    }

    .badge {
        background: rgba(129, 140, 248, 0.2);
        color: #818cf8;
        font-size: 0.65rem;
        padding: 2px 6px;
        border-radius: 4px;
        font-weight: 600;
        letter-spacing: 0.05em;
    }

    .toolbar-actions {
        display: flex;
        align-items: center;
        gap: 1rem;
    }

    .stats {
        font-size: 0.75rem;
        color: #64748b;
        font-family: "Fira Code", monospace;
    }

    .icon-btn {
        background: none;
        border: none;
        color: #94a3b8;
        cursor: pointer;
        padding: 4px;
        border-radius: 4px;
        transition: all 0.2s;
        display: flex;
        align-items: center;
        justify-content: center;
    }

    .icon-btn:hover:not(:disabled) {
        color: #fff;
        background: rgba(255, 255, 255, 0.1);
    }

    .icon-btn:disabled {
        opacity: 0.3;
        cursor: not-allowed;
    }

    .success {
        color: #4ade80;
    }

    .editor-wrapper {
        flex: 1;
        position: relative;
        overflow: hidden;
    }

    /* Common styles for editor and highlight overlay */
    textarea,
    pre {
        width: 100%;
        height: 100%;
        margin: 0;
        padding: 1rem;
        border: none;
        box-sizing: border-box;
        font-family: "Fira Code", "Monaco", monospace;
        font-size: 14px;
        line-height: 1.6;
        tab-size: 2;
        white-space: pre-wrap;
        word-break: break-all;
        overflow: auto;
    }

    .editor-wrapper.with-overlay textarea {
        color: transparent;
        caret-color: #fff;
        z-index: 2;
    }

    .editor-wrapper.with-overlay pre {
        display: block;
        position: absolute;
        top: 0;
        left: 0;
        background: #0f172a;
        color: #e2e8f0;
        pointer-events: none;
        z-index: 1;
    }

    /* Default textarea state (no overlay) */
    textarea {
        position: absolute;
        top: 0;
        left: 0;
        background: transparent;
        color: #e2e8f0; /* Visible by default */
        caret-color: #fff;
        resize: none;
        outline: none;
        z-index: 2;
    }

    /* Default pre state (hidden when no overlay) */
    pre {
        display: none;
    }

    /* Scrollbar styling */
    textarea::-webkit-scrollbar,
    pre::-webkit-scrollbar {
        width: 10px;
        height: 10px;
    }

    textarea::-webkit-scrollbar-track,
    pre::-webkit-scrollbar-track {
        background: #0f172a;
    }

    textarea::-webkit-scrollbar-thumb,
    pre::-webkit-scrollbar-thumb {
        background: #334155;
        border-radius: 5px;
        border: 2px solid #0f172a;
    }

    textarea::-webkit-scrollbar-thumb:hover {
        background: #475569;
    }

    .error-toast {
        position: absolute;
        bottom: 1rem;
        right: 1rem;
        background: rgba(239, 68, 68, 0.9);
        color: white;
        padding: 0.5rem 1rem;
        border-radius: 6px;
        font-size: 0.875rem;
        backdrop-filter: blur(4px);
        animation: slideUp 0.3s ease;
        z-index: 10;
    }

    @keyframes slideUp {
        from {
            transform: translateY(100%);
            opacity: 0;
        }
        to {
            transform: translateY(0);
            opacity: 1;
        }
    }

    .center-controls {
        width: 60px;
        background: #1e293b;
        border-left: 1px solid #334155;
        border-right: 1px solid #334155;
        display: flex;
        flex-direction: column;
        align-items: center;
        padding: 1rem 0;
        gap: 1rem;
        flex-shrink: 0;
        z-index: 10;
    }

    .control-btn {
        display: flex;
        flex-direction: column;
        align-items: center;
        gap: 4px;
        background: none;
        border: none;
        color: #94a3b8;
        cursor: pointer;
        font-size: 0.75rem;
        padding: 0.5rem 0.25rem;
        width: 100%;
        transition: all 0.2s;
    }

    .control-btn:hover {
        color: #fff;
        background: rgba(255, 255, 255, 0.05);
    }

    .control-btn.primary {
        color: #818cf8;
    }

    .control-btn.primary:hover {
        color: #a5b4fc;
        background: rgba(129, 140, 248, 0.1);
    }

    .direction-indicator {
        margin-top: auto;
        margin-bottom: auto;
        color: #475569;
    }

    @media (max-width: 768px) {
        main {
            flex-direction: column;
        }

        .center-controls {
            width: 100%;
            height: auto;
            flex-direction: row;
            justify-content: center;
            border-left: none;
            border-right: none;
            border-top: 1px solid #334155;
            border-bottom: 1px solid #334155;
            padding: 0.5rem;
        }

        .direction-indicator {
            transform: rotate(90deg);
            margin: 0;
        }

        .editor-pane {
            height: 50%;
        }
    }

    /* PrismJS Theme Overrides */
    :global(code[class*="language-"]),
    :global(pre[class*="language-"]) {
        text-shadow: none !important;
        font-family: "Fira Code", "Monaco", monospace !important;
        font-size: 14px !important;
        line-height: 1.6 !important;
    }

    :global(.token.comment),
    :global(.token.prolog),
    :global(.token.doctype),
    :global(.token.cdata) {
        color: #64748b;
    }

    :global(.token.punctuation) {
        color: #94a3b8;
    }

    :global(.token.property) {
        color: #7dd3fc;
    }

    :global(.token.string) {
        color: #86efac;
    }

    :global(.token.number),
    :global(.token.boolean) {
        color: #fca5a5;
    }

    :global(.token.operator) {
        color: #cbd5e1;
    }
</style>
