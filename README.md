# Freiheit Media – Open WebUI Fork

> Forked from [open-webui/open-webui](https://github.com/open-webui/open-webui).
> This fork adds light brand customization for an internal Freiheit Media test
> deployment and a written conceptual answer for the second part of the task.

The original upstream README starts further below.

## 1. Setup

Tested on macOS 14 with Docker Desktop 29.3 and Docker Compose v2.

```bash
# 1. clone the fork
git clone https://github.com/volodadatskiv2223/open-webui.git
cd open-webui

# 2. build & start (Open WebUI + bundled Ollama)
docker compose up -d --build

# 3. open the UI
open http://localhost:3000
```

The first build takes a few minutes because the SvelteKit frontend is compiled
from source (so the UI changes in this fork end up in the container image).

To stop / clean up:

```bash
docker compose down            # stop containers, keep data
docker compose down -v         # stop containers and wipe volumes
```

If you do not want the bundled Ollama service, the simpler one-liner from the
upstream README also works (but will *not* include the brand changes, since it
pulls the official image):

```bash
docker run -d -p 3000:8080 -v open-webui:/app/backend/data \
  --name open-webui ghcr.io/open-webui/open-webui:main
```

## 2. UI changes (Part A)

Two small, code-level modifications. All changes are confined to the
SvelteKit frontend — no upstream behaviour is altered.

### 2.1 Branding change — primary color

The login / sign-up button (the most visible call-to-action on first contact
with the app) now uses the Freiheit Media brand blue **`#1E40AF`** instead of
the muted gray default.

- New CSS custom properties `--brand-color` and `--brand-color-hover` and a
  reusable `.fm-brand-btn` class added in [`src/app.css`](src/app.css).
- The `Sign in` / `Create Account` / `Authenticate` buttons in
  [`src/routes/auth/+page.svelte`](src/routes/auth/+page.svelte) switched to
  the new class.

### 2.2 Custom UI elements

Two small, static elements that make the deployment context obvious to any
user landing in the app:

1. **Top banner** — a slim brand-blue strip rendered on top of the app shell
   reading *"Freiheit Media – Internal LLM"*. Implemented inside the `(app)`
   route layout
   [`src/routes/(app)/+layout.svelte`](src/routes/%28app%29/+layout.svelte)
   and styled via `.fm-brand-banner` in `src/app.css`. The shell layout was
   converted from a single `flex-row` container to a `flex-col` parent with
   the banner as the first child and the existing sidebar/content row as the
   second child, so the banner does not overlap the sidebar.
2. **Sidebar environment badge** — a small pill labelled
   *"Environment: Local Test"* with a green status dot, placed directly above
   the user menu at the bottom of the sidebar. Implemented in
   [`src/lib/components/layout/Sidebar.svelte`](src/lib/components/layout/Sidebar.svelte)
   and styled via `.fm-env-badge` in `src/app.css`.

### 2.3 Files touched

| File | Change |
|------|--------|
| `src/app.css` | Brand CSS variables, `.fm-brand-banner`, `.fm-env-badge`, `.fm-brand-btn` styles |
| `src/routes/auth/+page.svelte` | Auth submit buttons re-skinned with `.fm-brand-btn` |
| `src/routes/(app)/+layout.svelte` | Top banner + flex-column layout wrap |
| `src/lib/components/layout/Sidebar.svelte` | Environment badge above user menu |

## 3. Part B — RAG answer quality & customer safety

Scenario: each customer has its own Knowledge Base (KB) and the model is, at
any given time, bound to exactly **one** customer's KB.

### 3.1 System prompt

```text
You are Freiheit Media's internal assistant for the customer whose Knowledge
Base is currently attached to this conversation. Treat that Knowledge Base as
the only authoritative source.

Hard rules:
1. Use ONLY the retrieved excerpts from the active Knowledge Base. Do not use
   prior knowledge, training data, public web facts, or anything from other
   customers' Knowledge Bases.
2. For every user question, classify the available evidence into exactly one
   of three states and answer accordingly:
   - FULL: every required fact is explicitly present in the retrieved
     excerpts. Answer directly and concisely.
   - PARTIAL: some required facts are present, others are missing. State what
     is supported by the KB, then explicitly list what is missing and stop.
     Do not fill the gap from outside knowledge.
   - NONE: no relevant excerpt was retrieved. Reply that the active Knowledge
     Base does not contain information on this topic and suggest the user
     contact the document owner. Do not guess.
3. Never speculate, infer beyond the text, or extrapolate numbers, dates,
   prices, names, policies or procedures that are not literally written in
   the retrieved excerpts.
4. Reference sources minimally and only when they add value. Use the format
   "Based on <Document Title>" or "(<Document Title>, p.X)". Never invent
   document titles or page numbers.
5. If the user asks about another customer, organisation or unrelated topic,
   refuse and remind them that this assistant is scoped to a single
   Knowledge Base.
6. If retrieved excerpts contradict each other, say so explicitly and quote
   the conflicting passages instead of picking one.
7. Answer in the user's language. Do not invent links, file paths, contacts
   or identifiers.

Output format:
- 1–6 sentences for FULL answers, bullet points only when listing items that
  are themselves bulleted in the source.
- Always end PARTIAL or NONE answers with a single line starting with
  "Missing in KB:" describing what was not found.
```

### 3.2 Three example user questions and expected answers

Assume the active KB belongs to customer *Acme GmbH* and contains, among
others, an *"Onboarding Guide – 2024"* and an *"IT Security Policy v3"*.

**Q1 (FULL answer available)**
> "How many vacation days does a new employee at Acme get in their first
> year?"

Expected answer:
> New Acme employees are entitled to 26 paid vacation days in their first
> calendar year, prorated by start date.
> Based on *Onboarding Guide – 2024*.

**Q2 (PARTIAL answer)**
> "What's the process for requesting a company laptop and how long does
> approval usually take?"

Expected answer:
> The Knowledge Base describes the request flow: the employee opens a ticket
> in the IT portal, their line manager approves it, and IT provisions the
> device. Based on *IT Security Policy v3*.
>
> Missing in KB: typical approval time is not stated in the active Knowledge
> Base. Please confirm with the IT team.

**Q3 (NONE — no information available)**
> "What is Acme's revenue forecast for Q4 2026?"

Expected answer:
> The active Knowledge Base does not contain information about Acme's Q4 2026
> revenue forecast. I cannot answer this from the documents available to me;
> please reach out to the Finance team or the document owner.
>
> Missing in KB: financial forecasts for 2026.

### 3.3 Why this approach improves reliability

- **Hard scoping to the active KB** is the single most effective guardrail
  against the most dangerous failure mode in a multi-tenant setup: leaking or
  mixing one customer's information into another customer's session. Pinning
  "only the retrieved excerpts" closes the easy attack surface where the
  model would otherwise fall back to its pre-training memory.
- **Three explicit evidence states (FULL / PARTIAL / NONE)** force the model
  to be honest about coverage instead of producing a single confident-looking
  answer regardless of how strong the evidence actually is. This is the
  classic defence against *fluent hallucination* — answers that read well but
  invent facts.
- **"Missing in KB:" tail line** gives the user (and any downstream
  automation) a deterministic, easy-to-detect signal that the answer is
  incomplete or unsupported, instead of having to interpret natural language.
- **Minimal, explicit source references** keep the answer scannable and let
  the user verify quickly, while the rule "never invent titles/pages" blocks
  one of the most common RAG hallucinations: fabricated citations.
- **Refusal on cross-customer or off-topic questions** keeps the assistant's
  behaviour predictable and auditable, which matters more in a B2B internal
  tool than raw helpfulness.

### 3.4 Failure modes this prompt prevents

| Failure mode | How it is prevented |
|---|---|
| Cross-customer leakage | Rule 1 forbids any source other than the active KB; rule 5 forces a refusal on cross-customer questions. |
| Confident hallucinated facts | Rules 2–3 force the FULL / PARTIAL / NONE classification and forbid filling gaps from prior knowledge. |
| Fabricated citations | Rule 4 restricts the citation format and forbids invented titles or page numbers. |
| Silent partial answers | Rule 2 requires PARTIAL answers to explicitly list what is missing, plus the mandatory `Missing in KB:` line. |
| Contradictions hidden from the user | Rule 6 requires surfacing conflicting passages instead of silently choosing one. |
| Prompt injection from a document ("ignore previous instructions") | The system prompt's hard rules are stated as non-overridable; can be reinforced operationally by sanitising retrieved chunks and never executing instructions inside them. |

### 3.5 Possible future improvements

- **Structured output** — return a JSON object `{state: FULL|PARTIAL|NONE,
  answer, citations[], missing[]}`. This makes the assistant directly
  consumable from automation, evaluation harnesses and analytics pipelines
  without parsing free text.
- **Retrieval guardrails on top of prompting** — enforce a similarity-score
  threshold so that "no good chunks retrieved" deterministically maps to
  state `NONE` even before the model is invoked, instead of relying on the
  model's self-assessment.
- **Per-customer KB isolation at the storage layer**, not only via prompt —
  separate vector indices (or hard tenant-id filters on every query) so
  leakage is impossible by construction, not only by instruction.
- **Citation verification step** — a lightweight post-processing pass that
  checks every cited document title actually exists in the active KB and
  rejects the answer otherwise.
- **Eval set per customer** — a small, manually curated set of FULL / PARTIAL
  / NONE questions per customer KB, run on every model or prompt change to
  catch regressions early.

---



![GitHub stars](https://img.shields.io/github/stars/open-webui/open-webui?style=social)
![GitHub forks](https://img.shields.io/github/forks/open-webui/open-webui?style=social)
![GitHub watchers](https://img.shields.io/github/watchers/open-webui/open-webui?style=social)
![GitHub repo size](https://img.shields.io/github/repo-size/open-webui/open-webui)
![GitHub language count](https://img.shields.io/github/languages/count/open-webui/open-webui)
![GitHub top language](https://img.shields.io/github/languages/top/open-webui/open-webui)
![GitHub last commit](https://img.shields.io/github/last-commit/open-webui/open-webui?color=red)
[![Discord](https://img.shields.io/badge/Discord-Open_WebUI-blue?logo=discord&logoColor=white)](https://discord.gg/5rJgQTnV4s)
[![](https://img.shields.io/static/v1?label=Sponsor&message=%E2%9D%A4&logo=GitHub&color=%23fe8e86)](https://github.com/sponsors/tjbck)

![Open WebUI Banner](./banner.png)

**Open WebUI is an [extensible](https://docs.openwebui.com/features/extensibility/plugin), feature-rich, and user-friendly self-hosted AI platform designed to operate entirely offline.** It supports various LLM runners like **Ollama** and **OpenAI-compatible APIs**, with **built-in inference engine** for RAG, making it a **powerful AI deployment solution**.

Passionate about open-source AI? [Join our team →](https://careers.openwebui.com/)

![Open WebUI Demo](./demo.png)

> [!TIP]  
> **Looking for an [Enterprise Plan](https://docs.openwebui.com/enterprise)?** – **[Speak with Our Sales Team Today!](https://docs.openwebui.com/enterprise)**
>
> Get **enhanced capabilities**, including **custom theming and branding**, **Service Level Agreement (SLA) support**, **Long-Term Support (LTS) versions**, and **more!**

For more information, be sure to check out our [Open WebUI Documentation](https://docs.openwebui.com/).

## Key Features of Open WebUI ⭐

- 🚀 **Effortless Setup**: Install seamlessly using Docker or Kubernetes (kubectl, kustomize or helm) for a hassle-free experience with support for both `:ollama` and `:cuda` tagged images.

- 🤝 **Ollama/OpenAI API Integration**: Effortlessly integrate OpenAI-compatible APIs for versatile conversations alongside Ollama models. Customize the OpenAI API URL to link with **LMStudio, GroqCloud, Mistral, OpenRouter, and more**.

- 🛡️ **Granular Permissions and User Groups**: By allowing administrators to create detailed user roles and permissions, we ensure a secure user environment. This granularity not only enhances security but also allows for customized user experiences, fostering a sense of ownership and responsibility amongst users.

- 📱 **Responsive Design**: Enjoy a seamless experience across Desktop PC, Laptop, and Mobile devices.

- 📱 **Progressive Web App (PWA) for Mobile**: Enjoy a native app-like experience on your mobile device with our PWA, providing offline access on localhost and a seamless user interface.

- ✒️🔢 **Full Markdown and LaTeX Support**: Elevate your LLM experience with comprehensive Markdown and LaTeX capabilities for enriched interaction.

- 🎤📹 **Hands-Free Voice/Video Call**: Experience seamless communication with integrated hands-free voice and video call features using multiple Speech-to-Text providers (Local Whisper, OpenAI, Deepgram, Azure) and Text-to-Speech engines (Azure, ElevenLabs, OpenAI, Transformers, WebAPI), allowing for dynamic and interactive chat environments.

- 🛠️ **Model Builder**: Easily create Ollama models via the Web UI. Create and add custom characters/agents, customize chat elements, and import models effortlessly through [Open WebUI Community](https://openwebui.com/) integration.

- 🐍 **Native Python Function Calling Tool**: Enhance your LLMs with built-in code editor support in the tools workspace. Bring Your Own Function (BYOF) by simply adding your pure Python functions, enabling seamless integration with LLMs.

- 💾 **Persistent Artifact Storage**: Built-in key-value storage API for artifacts, enabling features like journals, trackers, leaderboards, and collaborative tools with both personal and shared data scopes across sessions.

- 📚 **Local RAG Integration**: Dive into the future of chat interactions with groundbreaking Retrieval Augmented Generation (RAG) support using your choice of 9 vector databases and multiple content extraction engines (Tika, Docling, Document Intelligence, Mistral OCR, PaddleOCR-vl, External loaders). Load documents directly into chat or add files to your document library, effortlessly accessing them using the `#` command before a query.

- 🔍 **Web Search for RAG**: Perform web searches using 15+ providers including `SearXNG`, `Google PSE`, `Brave Search`, `Kagi`, `Mojeek`, `Tavily`, `Perplexity`, `serpstack`, `serper`, `Serply`, `DuckDuckGo`, `SearchApi`, `SerpApi`, `Bing`, `Jina`, `Exa`, `Sougou`, `Azure AI Search`, and `Ollama Cloud`, injecting results directly into your chat experience.

- 🌐 **Web Browsing Capability**: Seamlessly integrate websites into your chat experience using the `#` command followed by a URL. This feature allows you to incorporate web content directly into your conversations, enhancing the richness and depth of your interactions.

- 🎨 **Image Generation & Editing Integration**: Create and edit images using multiple engines including OpenAI's DALL-E, Gemini, ComfyUI (local), and AUTOMATIC1111 (local), with support for both generation and prompt-based editing workflows.

- ⚙️ **Many Models Conversations**: Effortlessly engage with various models simultaneously, harnessing their unique strengths for optimal responses. Enhance your experience by leveraging a diverse set of models in parallel.

- 🔐 **Role-Based Access Control (RBAC)**: Ensure secure access with restricted permissions; only authorized individuals can access your Ollama, and exclusive model creation/pulling rights are reserved for administrators.

- 🗄️ **Flexible Database & Storage Options**: Choose from SQLite (with optional encryption), PostgreSQL, or configure cloud storage backends (S3, Google Cloud Storage, Azure Blob Storage) for scalable deployments.

- 🔍 **Advanced Vector Database Support**: Select from 9 vector database options including ChromaDB, PGVector, Qdrant, Milvus, Elasticsearch, OpenSearch, Pinecone, S3Vector, and Oracle 23ai for optimal RAG performance.

- 🔐 **Enterprise Authentication**: Full support for LDAP/Active Directory integration, SCIM 2.0 automated provisioning, and SSO via trusted headers alongside OAuth providers. Enterprise-grade user and group provisioning through SCIM 2.0 protocol, enabling seamless integration with identity providers like Okta, Azure AD, and Google Workspace for automated user lifecycle management.

- ☁️ **Cloud-Native Integration**: Native support for Google Drive and OneDrive/SharePoint file picking, enabling seamless document import from enterprise cloud storage.

- 📊 **Production Observability**: Built-in OpenTelemetry support for traces, metrics, and logs, enabling comprehensive monitoring with your existing observability stack.

- ⚖️ **Horizontal Scalability**: Redis-backed session management and WebSocket support for multi-worker and multi-node deployments behind load balancers.

- 🌐🌍 **Multilingual Support**: Experience Open WebUI in your preferred language with our internationalization (i18n) support. Join us in expanding our supported languages! We're actively seeking contributors!

- 🧩 **Pipelines, Open WebUI Plugin Support**: Seamlessly integrate custom logic and Python libraries into Open WebUI using [Pipelines Plugin Framework](https://github.com/open-webui/pipelines). Launch your Pipelines instance, set the OpenAI URL to the Pipelines URL, and explore endless possibilities. [Examples](https://github.com/open-webui/pipelines/tree/main/examples) include **Function Calling**, User **Rate Limiting** to control access, **Usage Monitoring** with tools like Langfuse, **Live Translation with LibreTranslate** for multilingual support, **Toxic Message Filtering** and much more.

- 🌟 **Continuous Updates**: We are committed to improving Open WebUI with regular updates, fixes, and new features.

Want to learn more about Open WebUI's features? Check out our [Open WebUI documentation](https://docs.openwebui.com/features) for a comprehensive overview!

---

We are incredibly grateful for the generous support of our sponsors. Their contributions help us to maintain and improve our project, ensuring we can continue to deliver quality work to our community. Thank you!

## How to Install 🚀

### Installation via Python pip 🐍

Open WebUI can be installed using pip, the Python package installer. Before proceeding, ensure you're using **Python 3.11** to avoid compatibility issues.

1. **Install Open WebUI**:
   Open your terminal and run the following command to install Open WebUI:

   ```bash
   pip install open-webui
   ```

2. **Running Open WebUI**:
   After installation, you can start Open WebUI by executing:

   ```bash
   open-webui serve
   ```

This will start the Open WebUI server, which you can access at [http://localhost:8080](http://localhost:8080)

### Quick Start with Docker 🐳

> [!NOTE]  
> Please note that for certain Docker environments, additional configurations might be needed. If you encounter any connection issues, our detailed guide on [Open WebUI Documentation](https://docs.openwebui.com/) is ready to assist you.

> [!WARNING]
> When using Docker to install Open WebUI, make sure to include the `-v open-webui:/app/backend/data` in your Docker command. This step is crucial as it ensures your database is properly mounted and prevents any loss of data.

> [!TIP]  
> If you wish to utilize Open WebUI with Ollama included or CUDA acceleration, we recommend utilizing our official images tagged with either `:cuda` or `:ollama`. To enable CUDA, you must install the [Nvidia CUDA container toolkit](https://docs.nvidia.com/dgx/nvidia-container-runtime-upgrade/) on your Linux/WSL system.

### Installation with Default Configuration

- **If Ollama is on your computer**, use this command:

  ```bash
  docker run -d -p 3000:8080 --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:main
  ```

- **If Ollama is on a Different Server**, use this command:

  To connect to Ollama on another server, change the `OLLAMA_BASE_URL` to the server's URL:

  ```bash
  docker run -d -p 3000:8080 -e OLLAMA_BASE_URL=https://example.com -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:main
  ```

- **To run Open WebUI with Nvidia GPU support**, use this command:

  ```bash
  docker run -d -p 3000:8080 --gpus all --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:cuda
  ```

### Installation for OpenAI API Usage Only

- **If you're only using OpenAI API**, use this command:

  ```bash
  docker run -d -p 3000:8080 -e OPENAI_API_KEY=your_secret_key -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:main
  ```

### Installing Open WebUI with Bundled Ollama Support

This installation method uses a single container image that bundles Open WebUI with Ollama, allowing for a streamlined setup via a single command. Choose the appropriate command based on your hardware setup:

- **With GPU Support**:
  Utilize GPU resources by running the following command:

  ```bash
  docker run -d -p 3000:8080 --gpus=all -v ollama:/root/.ollama -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:ollama
  ```

- **For CPU Only**:
  If you're not using a GPU, use this command instead:

  ```bash
  docker run -d -p 3000:8080 -v ollama:/root/.ollama -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:ollama
  ```

Both commands facilitate a built-in, hassle-free installation of both Open WebUI and Ollama, ensuring that you can get everything up and running swiftly.

After installation, you can access Open WebUI at [http://localhost:3000](http://localhost:3000). Enjoy! 😄

### Other Installation Methods

We offer various installation alternatives, including non-Docker native installation methods, Docker Compose, Kustomize, and Helm. Visit our [Open WebUI Documentation](https://docs.openwebui.com/getting-started/) or join our [Discord community](https://discord.gg/5rJgQTnV4s) for comprehensive guidance.

### Troubleshooting

Encountering connection issues? Our [Open WebUI Documentation](https://docs.openwebui.com/troubleshooting/) has got you covered. For further assistance and to join our vibrant community, visit the [Open WebUI Discord](https://discord.gg/5rJgQTnV4s).

#### Open WebUI: Server Connection Error

If you're experiencing connection issues, it’s often due to the WebUI docker container not being able to reach the Ollama server at 127.0.0.1:11434 (host.docker.internal:11434) inside the container . Use the `--network=host` flag in your docker command to resolve this. Note that the port changes from 3000 to 8080, resulting in the link: `http://localhost:8080`.

**Example Docker Command**:

```bash
docker run -d --network=host -v open-webui:/app/backend/data -e OLLAMA_BASE_URL=http://127.0.0.1:11434 --name open-webui --restart always ghcr.io/open-webui/open-webui:main
```

### Keeping Your Docker Installation Up-to-Date

Check our Updating Guide available in our [Open WebUI Documentation](https://docs.openwebui.com/getting-started/updating).

### Using the Dev Branch 🌙

> [!WARNING]
> The `:dev` branch contains the latest unstable features and changes. Use it at your own risk as it may have bugs or incomplete features.

If you want to try out the latest bleeding-edge features and are okay with occasional instability, you can use the `:dev` tag like this:

```bash
docker run -d -p 3000:8080 -v open-webui:/app/backend/data --name open-webui --add-host=host.docker.internal:host-gateway --restart always ghcr.io/open-webui/open-webui:dev
```

### Offline Mode

If you are running Open WebUI in an offline environment, you can set the `HF_HUB_OFFLINE` environment variable to `1` to prevent attempts to download models from the internet.

```bash
export HF_HUB_OFFLINE=1
```

## What's Next? 🌟

Discover upcoming features on our roadmap in the [Open WebUI Documentation](https://docs.openwebui.com/roadmap/).

## License 📜

This project contains code under multiple licenses. The current codebase includes components licensed under the Open WebUI License with an additional requirement to preserve the "Open WebUI" branding, as well as prior contributions under their respective original licenses. For a detailed record of license changes and the applicable terms for each section of the code, please refer to [LICENSE_HISTORY](./LICENSE_HISTORY). For complete and updated licensing details, please see the [LICENSE](./LICENSE) and [LICENSE_HISTORY](./LICENSE_HISTORY) files.

## Support 💬

If you have any questions, suggestions, or need assistance, please open an issue or join our
[Open WebUI Discord community](https://discord.gg/5rJgQTnV4s) to connect with us! 🤝

## Star History

<a href="https://star-history.com/#open-webui/open-webui&Date">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/svg?repos=open-webui/open-webui&type=Date&theme=dark" />
    <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/svg?repos=open-webui/open-webui&type=Date" />
    <img alt="Star History Chart" src="https://api.star-history.com/svg?repos=open-webui/open-webui&type=Date" />
  </picture>
</a>

---

Created by [Timothy Jaeryang Baek](https://github.com/tjbck) - Let's make Open WebUI even more amazing together! 💪
