# B64 Converter

A professional, modern, and high-performance Base64 converter application built with SvelteKit. Designed for developers who need to quickly encode or decode text and binary files with advanced editor features.

## ✨ Features

- **Real-Time Conversion**: Instant bidirectional encoding and decoding as you type.
- **Advanced Editor Interface**:
  - Split-pane layout with independent toolbars.
  - Synchronized scrolling between editor and syntax highlighter.
  - Line numbers for precise navigation.
  - "Clear All" and clipboard copy controls.
- **Smart Syntax Highlighting**:
  - Automatically detects and highlights content.
  - Supports **JSON**, **YAML**, **XML**, **Java**, and **Kotlin**.
  - Powered by [PrismJS](https://prismjs.com/).
- **Drag & Drop Support**:
  - **Text Files**: Drag text-based files (txt, md, json, xml, java, etc.) to the Text pane to load and encode.
  - **Base64 Files**: Drag Base64 text files to the Base64 pane to decode.
  - **Binary Files**: Drag binary files (images, PDFs, executables) to the Text pane to **automatically convert them to Base64** strings without textual corruption.
- **Modern UI/UX**:
  - Clean, dark-themed aesthetics.
  - Responsive design for various screen sizes.
  - Visual feedback for drag-and-drop actions.

## 🛠️ Technology Stack

- **Framework**: [SvelteKit](https://kit.svelte.dev/) (Svelte 5)
- **Language**: TypeScript
- **Styling**: Native CSS (Scoped components)
- **Syntax Highlighting**: PrismJS
- **Formatting Detection**: js-yaml, custom heuristics
- **Deployment**: [Cloudflare Pages](https://pages.cloudflare.com/) (`@sveltejs/adapter-cloudflare`)

## 🚀 Development

1. **Clone the repository**:
   ```bash
   git clone <repository-url>
   cd b64
   ```

2. **Install dependencies**:
   ```bash
   npm install
   ```

3. **Start the development server**:
   ```bash
   npm run dev
   ```
   > **Note**: Ensure you only have **one** instance of the dev server running to avoid conflicts.

4. **Build for production**:
   ```bash
   npm run build
   ```
   The output will be generated in the `.svelte-kit/cloudflare` directory ready for deployment.

## 📦 Deployment

This project is configured to deploy to **Cloudflare Pages**. It uses the `@sveltejs/adapter-cloudflare` adapter.

To deploy manually via Wrangler:
```bash
npx wrangler pages deploy .svelte-kit/cloudflare
```

## 📝 License

MIT
