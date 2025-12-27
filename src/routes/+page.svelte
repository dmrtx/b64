<script lang="ts">
    import Prism from "prismjs";
    import "prismjs/components/prism-json";
    import "prismjs/components/prism-yaml";
    import "prismjs/components/prism-java";
    import "prismjs/components/prism-kotlin";
    import "prismjs/components/prism-markup"; // Handles XML/HTML
    import yaml from "js-yaml";

    let textInput = $state("");
    let b64Input = $state("");
    let textError = $state("");
    let b64Error = $state("");
    let textCopied = $state(false);
    let b64Copied = $state(false);
    let detectedType = $state<
        "json" | "yaml" | "java" | "kotlin" | "xml" | null
    >(null);
    let isDragging = $state(false);
    let dragTarget = $state<"text" | "b64" | null>(null);

    let textEditor: HTMLTextAreaElement;
    let textHighlight: HTMLElement;
    let textGutter: HTMLElement;

    let b64Editor: HTMLTextAreaElement;
    let b64Gutter: HTMLElement;

    function detectType(
        text: string,
    ): "json" | "yaml" | "java" | "kotlin" | "xml" | null {
        const trimmed = text.trim();
        if (!trimmed) return null;

        try {
            // Ensure it looks like JSON object/array before parsing to avoid false positives with numbers/strings
            if (
                (trimmed.startsWith("{") && trimmed.endsWith("}")) ||
                (trimmed.startsWith("[") && trimmed.endsWith("]"))
            ) {
                JSON.parse(text);
                return "json";
            }
        } catch (e) {}

        // XML Detection
        if (
            trimmed.startsWith("<?xml") ||
            (trimmed.startsWith("<") &&
                trimmed.endsWith(">") &&
                /<\/[a-zA-Z0-9_\-:]+>/.test(text))
        ) {
            return "xml";
        }

        try {
            const parsed = yaml.load(text);
            if (typeof parsed === "object" && parsed !== null) {
                return "yaml";
            }
        } catch (e) {}

        if (
            /\b(public|private|protected)\s+(class|interface|enum)\b/.test(
                text,
            ) ||
            /\bpublic\s+static\s+void\s+main\b/.test(text) ||
            /\bSystem\.out\.print/.test(text) ||
            (/\bclass\b/.test(text) &&
                /[;{}]/.test(text) &&
                !/\bfun\b/.test(text))
        ) {
            return "java";
        }

        if (
            /\bfun\s+main\b/.test(text) ||
            /\b(val|var)\s+\w+/.test(text) ||
            /\bdata\s+class\b/.test(text) ||
            /\bcompanion\s+object\b/.test(text) ||
            /\bsuspend\s+fun\b/.test(text) ||
            (text.includes("fun ") && !text.includes("function "))
        ) {
            return "kotlin";
        }

        return null;
    }

    function highlight(
        text: string,
        type: "json" | "yaml" | "java" | "kotlin" | "xml" | null,
    ) {
        if (!text) return "<br>";

        if (type === "json") {
            return Prism.highlight(text, Prism.languages.json, "json") + "<br>";
        } else if (type === "yaml") {
            return Prism.highlight(text, Prism.languages.yaml, "yaml") + "<br>";
        } else if (type === "java") {
            return Prism.highlight(text, Prism.languages.java, "java") + "<br>";
        } else if (type === "kotlin") {
            return (
                Prism.highlight(text, Prism.languages.kotlin, "kotlin") + "<br>"
            );
        } else if (type === "xml") {
            return (
                Prism.highlight(text, Prism.languages.markup, "markup") + "<br>"
            );
        }

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

    function getLineNumbers(text: string) {
        const lines = text.split("\n").length;
        return Array.from({ length: Math.max(1, lines) }, (_, i) => i + 1).join(
            "\n",
        );
    }

    function encodeToBase64() {
        textError = "";
        detectedType = detectType(textInput);

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
            detectedType = null;
            return;
        }
        try {
            textInput = decodeURIComponent(escape(atob(b64Input)));
            detectedType = detectType(textInput);
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
        detectedType = null;
    }

    function handleTextInput(e: Event) {
        textInput = (e.target as HTMLTextAreaElement).value;
        encodeToBase64();
    }

    function handleB64Input(e: Event) {
        b64Input = (e.target as HTMLTextAreaElement).value;
        decodeFromBase64();
    }

    function syncScroll(
        source: HTMLTextAreaElement,
        highlightTarget: HTMLElement | null,
        gutterTarget: HTMLElement | null,
    ) {
        if (highlightTarget) {
            highlightTarget.scrollTop = source.scrollTop;
            highlightTarget.scrollLeft = source.scrollLeft;
        }
        if (gutterTarget) {
            gutterTarget.scrollTop = source.scrollTop;
        }
    }

    // Drag and Drop Handlers
    let dragCounter = $state(0);

    function handleDragEnter(e: DragEvent, type: "text" | "b64") {
        e.preventDefault();
        e.stopPropagation();
        dragTarget = type;
        dragCounter++;
        isDragging = true;
    }

    function handleDragLeave(e: DragEvent) {
        e.preventDefault();
        e.stopPropagation();
        dragCounter--;
        if (dragCounter <= 0) {
            isDragging = false;
            dragCounter = 0;
            dragTarget = null;
        }
    }

    function handleDragOver(e: DragEvent) {
        e.preventDefault(); // Necessary to allow dropping
        e.stopPropagation();
    }

    function handleDrop(e: DragEvent, type: "text" | "b64") {
        e.preventDefault();
        e.stopPropagation();
        isDragging = false;
        dragCounter = 0;
        dragTarget = null;

        const files = e.dataTransfer?.files;
        if (!files || files.length === 0) return;

        const file = files[0];
        const reader = new FileReader();

        reader.onload = (event) => {
            const content = event.target?.result as string;
            if (type === "text") {
                textInput = content;
                encodeToBase64();
            } else {
                b64Input = content;
                decodeFromBase64();
            }
        };

        reader.readAsText(file);
    }
</script>

<svelte:head>
    <title>B64 Converter - Professional Tool</title>
    <meta
        name="description"
        content="Professional Base64 converter with line numbers. Split-view interface, real-time conversion, JSON/YAML/Java/Kotlin syntax highlighting."
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
        <!-- Text Input Pane -->
        <div
            class="editor-pane {isDragging && dragTarget === 'text'
                ? 'dragging'
                : ''}"
            ondragenter={(e) => handleDragEnter(e, "text")}
            ondragover={handleDragOver}
            ondragleave={handleDragLeave}
            ondrop={(e) => handleDrop(e, "text")}
            role="region"
            aria-label="Text Input Pane"
        >
            {#if isDragging && dragTarget === "text"}
                <div class="drag-overlay">
                    <svg
                        xmlns="http://www.w3.org/2000/svg"
                        width="48"
                        height="48"
                        viewBox="0 0 24 24"
                        fill="none"
                        stroke="currentColor"
                        stroke-width="2"
                        stroke-linecap="round"
                        stroke-linejoin="round"
                    >
                        <path
                            d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"
                        />
                        <polyline points="14 2 14 8 20 8" />
                        <line x1="12" y1="18" x2="12" y2="12" />
                        <line x1="9" y1="15" x2="15" y2="15" />
                    </svg>
                    <span>Drop Plain Text File</span>
                </div>
            {/if}
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
                    {#if detectedType === "json"}
                        <span class="badge">JSON</span>
                    {:else if detectedType === "yaml"}
                        <span class="badge yaml">YAML</span>
                    {:else if detectedType === "xml"}
                        <span class="badge xml">XML</span>
                    {:else if detectedType === "java"}
                        <span class="badge java">Java</span>
                    {:else if detectedType === "kotlin"}
                        <span class="badge kotlin">Kotlin</span>
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

            <div class="editor-container">
                <div class="gutter" bind:this={textGutter}>
                    {getLineNumbers(textInput)}
                </div>
                <div class="editor-wrapper with-overlay">
                    <pre aria-hidden="true" bind:this={textHighlight}><code
                            class="language-{detectedType === 'xml'
                                ? 'markup'
                                : detectedType || 'none'}"
                            >{@html highlight(textInput, detectedType)}</code
                        ></pre>
                    <textarea
                        bind:this={textEditor}
                        value={textInput}
                        oninput={handleTextInput}
                        onscroll={() =>
                            syncScroll(textEditor, textHighlight, textGutter)}
                        placeholder="Type or paste code here, or drop a file..."
                        class:has-error={textError}
                        spellcheck="false"
                    ></textarea>
                </div>
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

        <!-- Base64 Input Pane -->
        <div
            class="editor-pane {isDragging && dragTarget === 'b64'
                ? 'dragging'
                : ''}"
            ondragenter={(e) => handleDragEnter(e, "b64")}
            ondragover={handleDragOver}
            ondragleave={handleDragLeave}
            ondrop={(e) => handleDrop(e, "b64")}
            role="region"
            aria-label="Base64 Input Pane"
        >
            {#if isDragging && dragTarget === "b64"}
                <div class="drag-overlay">
                    <svg
                        xmlns="http://www.w3.org/2000/svg"
                        width="48"
                        height="48"
                        viewBox="0 0 24 24"
                        fill="none"
                        stroke="currentColor"
                        stroke-width="2"
                        stroke-linecap="round"
                        stroke-linejoin="round"
                    >
                        <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4" />
                        <polyline points="7 10 12 15 17 10" />
                        <line x1="12" y1="15" x2="12" y2="3" />
                    </svg>
                    <span>Drop Base64 File</span>
                </div>
            {/if}
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

            <div class="editor-container">
                <div class="gutter" bind:this={b64Gutter}>
                    {getLineNumbers(b64Input)}
                </div>
                <div class="editor-wrapper">
                    <textarea
                        bind:this={b64Editor}
                        value={b64Input}
                        oninput={handleB64Input}
                        onscroll={() => syncScroll(b64Editor, null, b64Gutter)}
                        placeholder="Base64 output or drop file..."
                        class:has-error={b64Error}
                        spellcheck="false"
                    ></textarea>
                </div>
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
        overflow: hidden;
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
        min-width: 0;
        background: #0f172a;
        position: relative;
        transition: background 0.2s ease;
    }

    .editor-pane.dragging {
        background: rgba(129, 140, 248, 0.05);
    }

    .drag-overlay {
        position: absolute;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        gap: 1rem;
        background: rgba(15, 23, 42, 0.85);
        backdrop-filter: blur(4px);
        z-index: 50;
        border: 2px dashed #818cf8;
        margin: 1rem;
        border-radius: 12px;
        color: #818cf8;
        font-weight: 600;
        pointer-events: none;
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

    .badge.yaml {
        background: rgba(234, 179, 8, 0.2);
        color: #facc15;
    }

    .badge.java {
        background: rgba(249, 115, 22, 0.2);
        color: #fb923c;
    }

    .badge.kotlin {
        background: rgba(167, 139, 250, 0.2);
        color: #c4b5fd;
    }

    .badge.xml {
        background: rgba(236, 72, 153, 0.2);
        color: #f472b6;
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

    .editor-container {
        flex: 1;
        display: flex;
        position: relative;
        overflow: hidden;
        background: #0f172a;
    }

    .gutter {
        width: 48px;
        background: #0f172a;
        border-right: 1px solid #334155;
        color: #475569;
        font-family: "Fira Code", "Monaco", monospace;
        font-size: 14px;
        line-height: 1.6;
        padding: 1rem 0;
        text-align: right;
        padding-right: 0.75rem;
        overflow: hidden;
        user-select: none;
        white-space: pre;
    }

    .editor-wrapper {
        flex: 1;
        position: relative;
        overflow: hidden;
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

    textarea {
        position: absolute;
        top: 0;
        left: 0;
        background: transparent;
        color: #e2e8f0;
        caret-color: #fff;
        resize: none;
        outline: none;
        z-index: 2;
    }

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

    /* Common styles for editor/pre for text matching */
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

    /* Common Token Colors */
    :global(.token.comment),
    :global(.token.prolog),
    :global(.token.doctype),
    :global(.token.cdata) {
        color: #64748b;
    }

    :global(.token.punctuation) {
        color: #94a3b8;
    }

    :global(.token.property),
    :global(.token.tag),
    :global(.token.constant),
    :global(.token.symbol),
    :global(.token.deleted) {
        color: #7dd3fc;
    }

    :global(.token.string),
    :global(.token.char),
    :global(.token.attr-value),
    :global(.token.builtin),
    :global(.token.inserted) {
        color: #86efac;
    }

    :global(.token.number),
    :global(.token.boolean) {
        color: #fca5a5;
    }

    :global(.token.operator),
    :global(.token.entity),
    :global(.token.url) {
        color: #cbd5e1;
    }

    :global(.token.key) {
        color: #7dd3fc;
    }

    :global(.token.atrule),
    :global(.token.attr-value),
    :global(.token.keyword) {
        color: #c084fc;
    }

    /* Java/Kotlin Specifics */
    :global(.token.class-name) {
        color: #fbbf24;
    }

    :global(.token.function) {
        color: #60a5fa;
    }

    :global(.token.annotation) {
        color: #fce7f3;
    }
</style>
