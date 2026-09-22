# DOCX preview + edit in the browser — technology and licensing research

**Prepared for:** a commercial CLOSED-SOURCE React 19 AI-chat product that needs a right-hand panel to **preview and edit `.docx`**
**Research date:** 2026-09-21 · All versions/statuses below were verified on this date unless explicitly flagged otherwise.
**Candidate stack under review:** `mammoth` (docx→HTML) + Tiptap editor + `docx` (docx.js) for export.

> Note on method: the `pwsh` shell in this environment has no working outbound network, so every fact below came from `web_fetch`/`web_search` against npm registry JSON, GitHub API, raw README/LICENSE files and vendor pages. Anything I could not reach or confirm is called out in **§6 What I could not verify**.

---

## 1. Tiptap 3.x — current status, licensing, React 19

### 1.1 Version and core license

| Item | Value | Source |
|---|---|---|
| Latest `@tiptap/core` | **3.31.3** | [registry.npmjs.org/@tiptap/core/latest](https://registry.npmjs.org/@tiptap/core/latest) |
| Latest `@tiptap/react` | **3.31.3** | [registry.npmjs.org/@tiptap/react/latest](https://registry.npmjs.org/@tiptap/react/latest) |
| Core license | **MIT** (`"license":"MIT"` in package metadata) | npm registry JSON above |
| Repo LICENSE | **MIT** — "Copyright (c) 2025, Tiptap GmbH" | [github.com/ueberdosis/tiptap/blob/main/LICENSE.md](https://github.com/ueberdosis/tiptap/blob/main/LICENSE.md) |
| Docs version | Tiptap Docs **3.x** | [tiptap.dev/docs](https://tiptap.dev/docs) |
| Repo | [github.com/ueberdosis/tiptap](https://github.com/ueberdosis/tiptap) (~32k stars per Tiptap's own site) | [tiptap.dev/open-source-to-platform](https://tiptap.dev/open-source-to-platform) |

**Conclusion:** the Tiptap **editor core is genuinely MIT** and free for commercial closed-source use. The things you pay for are not the editor — they are the **Platform bundles** (private `@tiptap-pro/*` registry packages + Tiptap Cloud services). Tiptap states this itself: *"The Tiptap Editor is open source (MIT) and free; only platform features and Cloud documents are priced."* ([feature comparison](https://tiptap.dev/feature-comparison))

### 1.2 What is commercial / proprietary

Paid things are delivered from a **private npm registry** (`@tiptap-pro:registry=https://registry.tiptap.dev/`) and require a Tiptap account + token ([Pro Extensions guide](https://tiptap.dev/docs/guides/pro-extensions)). The docs sidebar labels each paid item with its plan, which is the cleanest authoritative list ([Extensions overview](https://tiptap.dev/docs/editor/extensions/overview)):

| Capability (docs label) | Status |
|---|---|
| **Comments** | **Start** plan (and requires Cloud-hosted documents) |
| **Export** (DOCX/PDF/ODT/EPUB/MD), **Import** (DOCX/MD) | **BETA + Start** |
| **Version / Snapshot** | **Start** |
| **Snapshot Compare** | **Team** |
| **Pages** (paginated page layout, headers/footers, page breaks) | **Team** |
| **Paste Handler** | **Team** |
| **Basic AI Generation** | **Start** |
| **AI Toolkit** | **BETA Add-on** |
| **Tracked Changes** | **Add-on, +$249/month** |
| **Collaboration** (OSS extension MIT, but **Cloud-hosted documents required**) | Platform |
| **UI Components for paid features** | Not open source; "A Tiptap Cloud subscription or trial is required" ([UI Components docs](https://tiptap.dev/docs/ui-components/getting-started/overview)) |
| Templates: **Notion Editor** (Start), **DOCX Editor** (Team); **Simple Editor = MIT** | [UI Components docs](https://tiptap.dev/docs/ui-components/getting-started/overview) |

**Explicitly free now (formerly Tiptap Pro), open-sourced under MIT in June 2025** — [Tiptap release notes](https://tiptap.dev/blog/release-notes/were-open-sourcing-more-of-tiptap), [HN thread](https://hn.nuxt.dev/item/44202103):

- Details / DetailsContent / DetailsSummary
- **Emoji**
- **DragHandle** (React and Vue)
- **FileHandler**
- InvisibleCharacters
- **Mathematics**
- **TableOfContents**
- UniqueID

Tiptap's own words: *"We originally monetized by selling individual 'Pro' extensions… we've sunset the free tier of Tiptap Cloud (where those extensions were included) and switched to a time-limited free trial."*

Collaboration specifics: `@tiptap/extension-collaboration` **3.31.3, MIT** ([npm](https://registry.npmjs.org/@tiptap/extension-collaboration/latest)) and **Hocuspocus** (self-hostable collab server) `@hocuspocus/server` **4.7.0, MIT** ([npm](https://registry.npmjs.org/@hocuspocus/server/latest)). So **self-hosted collaboration is free/MIT**; the paid part is Tiptap's managed Cloud (documents, storage, webhooks, comments/history hosted documents).

### 1.3 Tiptap pricing tiers

Source: **[https://tiptap.dev/pricing](https://tiptap.dev/pricing)** (monthly figures; page offers "Yearly (-20%)", and the annual totals are JS-rendered as `$Value` placeholders, so exact annual numbers are not quoted here).

| Plan | Price | Notable inclusions |
|---|---|---|
| **Open source** | **$0** (MIT) | Tiptap Editor + Hocuspocus. No cloud documents. |
| **Start** | **$49/mo** | 500 cloud documents, 2 environments, 2 dev licenses, "In-line AI extension", **"Simple DOCX import & export"**, community support, 30-day free trial |
| **Team** | **$149/mo** | 5,000 cloud docs, **"Pages for page-based layouts"**, webhooks + API access, email support |
| **Business** | **$999/mo** | 50,000 cloud docs, 5 environments, 10 dev licenses, betas |
| **Enterprise** | Custom | On-premises, own auth/storage/AI models, SLA, HIPAA "coming soon" |
| Add-ons | **Tracked Changes +$249/mo**; AI Toolkit "Talk to sales"; Developer licenses **from $39/dev/mo** | |

**Is there a free tier for small companies?** **No free Platform tier.** The MIT editor is free forever, but the cloud/platform features went from a free hobby tier to a **30-day trial only** ([release notes, June 6 2025](https://tiptap.dev/blog/release-notes/were-open-sourcing-more-of-tiptap)). Also note the paid Pro license is **B2B only**: the [Pro license](https://tiptap.dev/pro-license) applies to *"a user … acting as entrepreneurs within the meaning of section 14 German Civil Code (BGB)"* and *"Consumers … are not offered the use of the Software."* It grants perpetual, non-exclusive, non-sublicensable rights, but **you may not distribute the Pro software as a standalone product**, and you must ship a copy of the license with distributions — relevant if your app is redistributed/white-labelled.

### 1.4 DOCX import/export in Tiptap (the direct competitor to mammoth)

Yes — **Tiptap now sells DOCX import/export**, as part of the **Conversion** product:

- Product page: [tiptap.dev/product/conversion](https://tiptap.dev/product/conversion) — tagged **Beta**, **Platform / On-premises**, "Available with the **Start** Plan", installs as **`@tiptap-pro/extension-import`** (and export counterpart).
- Docs: [Conversion overview](https://tiptap.dev/docs/conversion/getting-started/overview) — *"Available in Start plan. Beta. Conversion is a Pro package included with all Tiptap subscriptions."*
- Current packages are `@tiptap-pro/extension-import` / `-export` plus a new `ConvertKit` (`@tiptap-pro/extension-convert-kit`); the **legacy** import/export packages "are being deprecated and will be sunset in 2026".
- Supported: DOCX/Markdown **import**; **export** to DOCX, PDF, ODT, EPUB, DOC, Markdown.
- Page-aware rendering (headers, footers, page breaks, page numbers, margins) additionally requires the **Pages** extension (Team tier).

Its own published **feature support matrix** ([link](https://tiptap.dev/docs/conversion/getting-started/feature-support-matrix)) is the most useful fidelity document in this whole report — see §5.3.

### 1.5 React 19 compatibility

- `@tiptap/react@3.31.3` declares `peerDependencies.react: "^17.0.0 || ^18.0.0 || ^19.0.0"` and its own devDependencies are `react@^19.0.0` — so **React 19 is an officially supported, actively tested target for the editor bindings** ([npm](https://registry.npmjs.org/@tiptap/react/latest)).
- ⚠️ **But Tiptap's UI Components layer is behind:** *"We're currently working on upgrading support for React 19 and newer framework versions. Some components may not yet be fully compatible. For now, the UI Components work best with **React 18** (and corresponding framework versions like Next.js 15)."* — [UI Components overview](https://tiptap.dev/docs/ui-components/getting-started/overview). If you were planning to use Tiptap's prebuilt React UI components/templates, that is a real blocker on React 19 today.
- ⚠️ **Open React 19 bug:** ["[Bug]: flushSync error when remounting useEditor with existing ID and ReactNodeViewRenderer in React 19"](https://github.com/ueberdosis/tiptap/issues/7543) — filed 2026-02-26 against 3.14.0/3.20.0, still **open** as of 2026-07-16, labelled `impact: high`, `complexity: hard`. Triggers when React-19 list reordering remounts components using `ReactNodeViewRenderer`. Relevant if your right-hand panel mounts/unmounts editors or renders custom node views.

---

## 2. mammoth.js — version, license, maintenance, fidelity limits

| Item | Value |
|---|---|
| Latest version | **1.12.3** ([npm registry](https://registry.npmjs.org/mammoth/latest)) |
| License | **BSD-2-Clause** (npm metadata `"license":"BSD-2-Clause"`; GitHub API confirms `bsd-2-clause`) |
| Repo | [github.com/mwilliamson/mammoth.js](https://github.com/mwilliamson/mammoth.js) |
| Maintenance | **Active.** Repo `pushed_at` **2026-09-19**, `updated_at` **2026-09-20**; not archived; 6,306 stars; 60 open issues; 670 forks ([GitHub API](https://api.github.com/repos/mwilliamson/mammoth.js)) |
| Release cadence | Steady (1.12.3 is current; npm publish timestamp is 2026) |
| Note | **Pull requests are not accepted** — the maintainer directs contributors to issues only (see [PR #459](https://github.com/mwilliamson/mammoth.js/pull/459), auto-closed with "In general, pull requests are not currently accepted") |

### 2.1 The official "what mammoth is and isn't" philosophy (quoted from the README)

Source: [raw README](https://raw.githubusercontent.com/mwilliamson/mammoth.js/master/README.md)

> "Mammoth is designed to convert .docx documents, such as those created by Microsoft Word, Google Docs and LibreOffice, and convert them to HTML. **Mammoth aims to produce simple and clean HTML by using semantic information in the document, and ignoring other details.** For instance, Mammoth converts any paragraph with the style `Heading 1` to `h1` elements, **rather than attempting to exactly copy the styling (font, text size, colour, etc.) of the heading.**"
>
> "**There's a large mismatch between the structure used by .docx and the structure of HTML, meaning that the conversion is unlikely to be perfect for more complicated documents. Mammoth works best if you only use styles to semantically mark up your document.**"

Its documented supported-feature list, verbatim highlights (these are the *supported* items, and the caveats are inside them):

> * "Tables. **The formatting of the table itself, such as borders, is currently ignored**, but the formatting of the text is treated the same as in the rest of the document."
> * "Images."
> * "**Text boxes. The contents of the text box are treated as a separate paragraph that appears after the paragraph containing the text box.**"
> * "Footnotes and endnotes." / "Comments." (comments are *"appended to the end of the document"*, and only if you map `comment-reference`)
> * Underline: *"By default, the underlining of any text is ignored"* (must opt in via a style map).
> * "Markdown support is **deprecated**."

Also relevant for an AI chat product that ingests **untrusted** `.docx`:

> "**Mammoth performs no sanitisation of the source document, and should therefore be used extremely carefully with untrusted user input.**" — README §Security, which lists `javascript:` link injection, external-file references (disabled by default via `externalFileAccess`), and "The conversion may exhibit pathological performance on certain documents" (DoS).

### 2.2 What mammoth does **not** do (fidelity limitations)

Confirmed by the README and its issue tracker:

| Not supported | Evidence |
|---|---|
| **Headers and footers** — not converted at all | ["Support header/footer" issue #9](https://github.com/mwilliamson/mammoth.js/issues/9) — **open since 2013-12-16**, last touched 2023-08-27; two community PRs (#373, #459) were never merged |
| **Exact page layout / pagination / page breaks / sections / columns** | README: mammoth ignores detail; output is an HTML *fragment* with no page model. (docx-preview's README makes the same point explicitly for renderers — see §4.2) |
| **Fonts, sizes, colours, highlight, alignment, spacing, indentation, paragraph borders** | README: *"ignoring other details … rather than attempting to exactly copy the styling"* |
| **Table borders, widths, cell shading, vertical merge fidelity** | README: table formatting *"is currently ignored"* |
| **Floating / anchored images, text wrapping, z-order** | Images are emitted by the image converter as plain `<img>` elements with a `src` (and alt text) — there is no positioning/wrap model in the API. The README's "Images" entry documents only `convertImage` / `src` handling. (A GitHub issue search for floating images returned 0 results — i.e. no support was ever landed.) |
| **Text boxes as positioned objects** | README: contents are re-flowed into a following paragraph — the box and its position are lost |
| **Shapes / WordArt / SmartArt / charts** | No mention anywhere in the README's supported list; nothing in the document model |
| **Table of contents (field-based)** | Not in the supported list (TOC is a field; mammoth's supported list has no field support) |
| **Equations (OMML)** | Not in the supported list |
| **Document metadata** | Not in the supported list |
| **Tracked changes / revision marks** | Not in the supported list — **flagged as not fully verified in this session** (see §6) |
| **Comments as margin anchors** | Comments are flattened to the end of the document with reference links |

---

## 3. `docx` (docx.js) — version, license, maintenance, and can it round-trip?

| Item | Value |
|---|---|
| Latest version | **9.7.1** ([npm registry](https://registry.npmjs.org/docx/latest)) |
| License | **MIT** (`"license":"MIT"`; GitHub API `spdx_id: MIT`) |
| Repo | [github.com/dolanmiu/docx](https://github.com/dolanmiu/docx) · docs [docx.js.org](https://docx.js.org/) |
| Maintenance | **Active.** `pushed_at` **2026-08-07**, `updated_at` **2026-09-18**; not archived; 5,909 stars; 163 open issues ([GitHub API](https://api.github.com/repos/dolanmiu/docx)) |
| Positioning (README, verbatim) | *"Easily generate and modify .docx files with JS/TS with a nice declarative API. Works for Node and on the Browser."* |

### 3.1 Does it read / parse an existing `.docx`? **Effectively no.**

This is the key finding, and it is confirmed from the repo's own docs:

1. The **README frames the library as a generator** — every linked example/demo is about *creating* (`demo/` = paragraphs, tables, images, headers/footers, TOC…), plus a "Docx.js Editor" playground for *writing code and previewing output*.
2. The **entire `docs/usage/` API index contains no "read", "parse", "extract" or "import" page.** It is a catalogue of document *authoring* features: bookmarks, bullet-points, change-tracking, checkboxes, columns, comments, convenience-functions, document, endnotes, fields, fonts, footnotes, headers-and-footers, hyperlinks, images, line-numbers, math, numbering, packers, page-layout, page-numbers, paragraph, **patcher**, sections, styling-with-js/xml, symbols, table-of-contents, tables, tabs, templates, text-box, text-frames, text, wps-text-box. ([docs/usage listing](https://api.github.com/repos/dolanmiu/docx/contents/docs/usage))
3. The **only** mechanism that touches an existing file is **`patchDocument`**, documented at [docx.js.org/usage/patcher](https://docx.js.org/usage/patcher.md). Its workflow *requires you to pre-edit the document in Word and insert mustache tags*:
   > "1. Open your existing word document in your favorite Word Processor
   > 2. Write tags in the document where you want to patch in a mustache style notation. For example, `{{my_patch}}`…
   > 3. Run the patcher with the patches as a key value pair."
   Patches are `PatchType.DOCUMENT` (Paragraph/Table children) or `PatchType.PARAGRAPH` (inline runs), with a `keepOriginalStyles` flag.
4. `templates.md` / `patcher.md` therefore give you **placeholder substitution**, not **round-tripping**.

**Conclusion:** `docx` 9.x **cannot parse an arbitrary existing `.docx` into a document model**. It can (a) generate a brand-new document from scratch, or (b) do surgical mustache-placeholder replacement inside a template you control. So in the `mammoth + Tiptap + docx` chain, the export step is a **re-authoring** step: you must write your own Tiptap-JSON → docx.js serializer, and the output document is a *new* document (new `styles.xml`, numbering, theme, section properties) — not the original file with your edits applied.

---

## 4. Other docx-in-browser options

### 4.1 `@eigenpal/docx-js-editor` — it exists, but it has been **renamed twice** and the old names are `deprecated` on npm

This was the most surprising finding. The trail:

| Package | Version | License | Status on npm |
|---|---|---|---|
| `@eigenpal/docx-js-editor` | 0.5.3 | MIT | **`"deprecated": "deprecated"`** — description: *"DEPRECATED: renamed to `@eigenpal/docx-editor-react`"* ([npm](https://registry.npmjs.org/@eigenpal/docx-js-editor/latest)) |
| `@eigenpal/docx-editor-react` | 1.9.0 | Apache-2.0 | **also flagged deprecated** ([npm](https://registry.npmjs.org/@eigenpal/docx-editor-react/latest)) |
| `@eigenpal/docx-editor-core` | 1.9.0 | Apache-2.0 | **also flagged deprecated** ([npm](https://registry.npmjs.org/@eigenpal/docx-editor-core/latest)) |
| **`@docx-editor.dev/core`** | **2.21.0** | **Apache-2.0** | Current ([npm](https://registry.npmjs.org/@docx-editor.dev/core/latest)) |
| **`@docx-editor.dev/react`** | **2.21.0** | **Apache-2.0** | Current; peers `react ^18 \|\| ^19` ([npm](https://registry.npmjs.org/@docx-editor.dev/react/latest)) |

So: **the project is very much alive, just rebranded** — it is *not* unmaintained. Evidence of active development:

- Repo [github.com/eigenpal/docx-editor](https://github.com/eigenpal/docx-editor) created **2026-07-20**, `pushed_at` **2026-09-20**, **388 stars**, 80 forks, 65 open issues, not archived ([GitHub API](https://api.github.com/repos/eigenpal/docx-editor)). Site: [docx-editor.dev](https://www.docx-editor.dev/).
- Site stats: **npm v2.21.0**, **135.3k downloads/month** for `@docx-editor.dev/core`.
- Claims (vendor-stated, not independently verified): "Lossless Roundtrip — The editor rewrites what it understands and preserves the rest byte-for-byte"; "Pixel-perfect OOXML rendering … inline and floating images with positioning and text wrap"; tracked changes, threaded comments, real-time collab, i18n, Markdown converter, an Office.js-compatible automation API, plus an MCP server (the core package ships a `docx-editor-mcp` binary).
- **Licensing is open-core**, per [LICENSE scope notice](https://raw.githubusercontent.com/eigenpal/docx-editor/main/LICENSE) and [pricing](https://www.docx-editor.dev/pricing):
  - `packages/core` + React/Vue adapters = **Apache-2.0**, free **including commercial products**.
  - `packages/pro/` and `packages/editor-api/` = **EigenPal Pro Evaluation License 1.0** — **commercial**, priced at **$500 USD/month**, covering tracked changes, comments, custom nodes, real-time collaboration, and the document automation API. One subscription covers one product.
- ⚠️ Pre-2.x `@eigenpal/*` versions are what a naive `npm install` search will surface, and they are explicitly deprecated — do **not** build on `@eigenpal/docx-js-editor`. Note also that the *old* `@eigenpal/docx-editor-core@1.9.0` depended on `docxtemplater` + `pizzip`; the current `@docx-editor.dev/core@2.21.0` no longer does (its deps are `fflate`, `fast-xml-parser`, `harfbuzzjs`, `utif2`, `emf-converter`, `bidi-js`, ProseMirror packages).

### 4.2 `docx-preview` / `docxjs` — **RENDER ONLY**, confirmed

| Item | Value |
|---|---|
| `docx-preview` | **0.4.0**, **Apache-2.0** ([npm](https://registry.npmjs.org/docx-preview/latest)) |
| `docxjs` (npm) | **does not exist — 404** ([registry](https://registry.npmjs.org/docxjs/latest)). "docxjs" is the *GitHub repo name* of `docx-preview`: [github.com/VolodymyrBaydalka/docxjs](https://github.com/VolodymyrBaydalka/docxjs) |
| Edit / round-trip? | **No.** README: *"Docx rendering library"*, *"Goal of this project is to render/convert DOCX document into HTML document with keeping HTML semantic as much as possible."* Public API is `renderAsync`, plus experimental `parseAsync` / `renderDocument`. There is no save/serialize-back path. |

Its README's own limitations section is a good reality check on browser pagination:

> *"Realtime page breaking is not implemented because it's requires re-calculation of sizes on each insertion and that could affect performance a lot."*
> *"Table of contents is built using the TOC fields and there is no efficient way to get table of contents at this point, since fields is not supported yet."*
> *"Thumbnails … Library renders DOCX into HTML, so it can't be efficiently used for thumbnails."*
> *"Status and stability: So far I can't come up with final approach of parsing documents and final structure of API. Only `renderAsync` function is stable."*

So `docx-preview` is a legitimate **view-only** option for the "preview" half of your panel (it does render headers/footers/footnotes/endnotes and can emulate page breaks), but it cannot edit.

### 4.3 SuperDoc — a very strong DOCX-native engine, but **AGPL-3.0**

| Item | Value |
|---|---|
| `superdoc` | **2.16.0**, **AGPL-3.0** ([npm](https://registry.npmjs.org/superdoc/latest)) |
| Repo | [github.com/superdoc/docx-editor](https://github.com/superdoc/docx-editor) — 909 stars per [docs.superdoc.dev](https://docs.superdoc.dev/resources/license/) |
| License model | **Dual-licensed**: *"SuperDoc's open-source code is available under the GNU Affero General Public License v3.0… Proprietary and commercial deployments are available under the SuperDoc Commercial License."* ([Licensing page](https://docs.superdoc.dev/resources/license/), [README](https://raw.githubusercontent.com/superdoc/docx-editor/main/README.md)) |
| Edit + round-trip? | **Yes.** README: *"**DOCX-native.** Pagination, sections, headers, footers, and tables stay document structures. **Edits write back to the XML without an HTML conversion step.**"* V2 uses an OOXML-backed document model (V1 used ProseMirror as its authoritative model). Ships React adapter (`superdoc/ui/react`), Yjs collaboration, tracked changes, comments, suggest mode. |
| **`SuperEditor` standalone package** | **Does not exist on npm.** `@superdoc/super-editor` → **404**; `@harbour-enterprises/super-editor` → **404**. A registry search for `super-editor` returns only unrelated packages (a 2020 Slate-based `super-editor`, `yanzi-super-editor`, etc.). SuperEditor is now documented as **"the low-level DOCX editing engine that powers SuperDoc"** — an internal layer, not a separately published package ([v1 docs](https://docs-v1.superdoc.dev/advanced/supereditor/overview)). |

**Bottom line for your closed-source product:** SuperDoc's AGPL-3.0 means you would need the **commercial license** (contact via [superdocportal.dev](https://www.superdocportal.dev/get-in-touch)). I could **not** retrieve SuperDoc's commercial pricing — `https://www.superdoc.dev/pricing` returns **404**.

### 4.4 Others I found (2025–2026)

| Option | What it is | Version / license | Edit + round-trip? | Maturity / caveats |
|---|---|---|---|---|
| **Tiptap Conversion** (`@tiptap-pro/*` + ConvertKit + Pages) | DOCX import→Tiptap JSON, export DOCX/PDF/ODT/EPUB/MD, paginated Pages layout | Paid, Start plan minimum (**$49/mo**); Pages needs **Team ($149/mo)**; Tracked changes **+$249/mo**; on-premises available | **Yes (round-trip via Tiptap JSON)**, but see fidelity matrix §5.3 | **BETA**; legacy `extension-import`/`-export` sunset in 2026; requires private registry token; conversion is synchronous; RTL unsupported |
| **EigenPal DOCX Editor** (`@docx-editor.dev/*`) | Browser WYSIWYG DOCX editor for React/Vue, canonical OOXML, Office.js-compatible API, MCP server | **2.21.0**, core **Apache-2.0**; Pro **$500/mo** | **Yes** — "lossless roundtrip", save-and-reopen digest in CI (vendor claim) | 2 months old as a repo (created 2026-07-20) but extremely active; 388 stars; the brand has already changed twice — watch for churn |
| **SuperDoc** (`superdoc`) | DOCX-native editor + Document API + Node/Python SDKs + CLI + MCP | **2.16.0**, **AGPL-3.0** / commercial | **Yes** | Mature; AGPL is the blocker for closed source |
| **docx-preview** | HTML renderer | **0.4.0**, **Apache-2.0** | **No — render only** | Mature and stable for viewing; API explicitly "experimental" beyond `renderAsync` |
| **ranuts/document** | "Edit DOCX/XLSX/PPTX in your browser — client-side, no server, works offline (OnlyOffice + WebAssembly)" | **license not verified** | Yes in principle (embeds OnlyOffice WASM) | Found via search ([github.com/ranuts/document](https://github.com/ranuts/document)); **not verified in this session** |
| **OnlyOffice / Collabora Online** | Full office suite engines embedded in the browser | AGPL-3.0 + commercial (OnlyOffice) | Yes | Heavyweight server/WASM deployment; **specific 2026 DOCX feature/version details not verified in this session.** A client-side OnlyOffice v9 wrapper also exists: [electroluxcode/onlyoffice-web-comp](https://github.com/electroluxcode/onlyoffice-web-comp) |
| **CKEditor 5** | Commercial rich-text editor with Word import/export features | GPL-2.0-or-later **or** commercial ([licensing page](https://ckeditor.com/legal/ckeditor-licensing-options/)) | Partially — Word import/export are premium features, not an OOXML editor | **Not verified in detail in this session** — must be checked before relying on it |
| **docxtemplater** | Templating engine (not an editor) | **3.70.1**, core **MIT**; *"Functionality can be added with the following **paid modules**"* (Image, HTML, XLSX, Chart, Table, Styling, Footnotes, …) | Generation/templating only | Only relevant if you template documents; listed here because the older EigenPal package depended on it |
| **`super-editor`, `@xcrong/docx-editor-core`, `@guillermorecoba/docx-editor-core`** | Lookalikes/forks surfaced by npm search | — | — | **Not vetted**; the EigenPal-named ones are deprecated forks of the old package |

---

## 5. Fidelity assessment

### 5.1 Summary table

| Library | Latest version | License | Edit or render only | Round-trip existing `.docx`? | Maintenance status |
|---|---|---|---|---|---|
| **Tiptap** (`@tiptap/core`, `@tiptap/react`) | **3.31.3** (2026) | **MIT** | Edit (rich text / ProseMirror) | **No** — HTML/JSON only; DOCX needs paid Conversion | **Very active** |
| **Tiptap Pro/Platform** (`@tiptap-pro/*`, Cloud) | Conversion/ConvertKit BETA; Pages etc. | **Proprietary** (Pro License; Cloud subscription) | Edit + import/export DOCX | **Yes**, via Tiptap JSON | Active, but **BETA**; legacy Import/Export sunset 2026 |
| **mammoth** | **1.12.3** | **BSD-2-Clause** | Convert only (docx→HTML) | **No** (one-way) | **Active** (pushed 2026-09-19); no PRs accepted |
| **docx** (docx.js) | **9.7.1** | **MIT** | Generate / patch (write only) | **No** — `patchDocument` = mustache placeholders only; no parse API | **Active** (pushed 2026-08-07) |
| **@eigenpal/docx-js-editor** | **0.5.3** | MIT | Edit | Yes | **DEPRECATED on npm** → use `@docx-editor.dev/*` |
| **@eigenpal/docx-editor-react / -core** | **1.9.0** | Apache-2.0 | Edit | Yes | **DEPRECATED on npm** → renamed again |
| **@docx-editor.dev/core + /react** | **2.21.0** | **Apache-2.0** (Pro packages separate) | **Edit** (browser WYSIWYG) | **Yes** (claimed lossless) | **Very active** (pushed 2026-09-20; repo 2 months old) |
| **@docx-editor.dev/pro, /editor-api** | 2.x | **EigenPal Pro Evaluation License** (commercial, **$500/mo**) | Edit + automate | Yes | Active |
| **docx-preview** (`docxjs` repo) | **0.4.0** | **Apache-2.0** | **Render only** | **No** | Maintained; API unversioned/experimental |
| **superdoc** | **2.16.0** | **AGPL-3.0** / commercial | **Edit** | **Yes** (DOCX-native, writes back to XML) | **Very active** (published 2026) |
| `docxjs` (npm pkg) | — | — | — | — | **Does not exist (404)** |
| `@superdoc/super-editor`, `@harbour-enterprises/super-editor` | — | — | — | — | **Do not exist (404)**; SuperEditor is an internal layer of SuperDoc |
| **docxtemplater** | **3.70.1** | **MIT** core + paid modules | Generate/template | No | Active |

### 5.2 The `mammoth → Tiptap → docx` chain: what visually breaks

The chain is **two lossy conversions with a lossy editor schema in between**, and the export side is a **re-authoring**, not a rewrite of the original file. Concretely, for a real-world Word document:

**Pass 1 — `mammoth` docx→HTML (irreversible by design).** Per the README's own statement that it ignores *"font, text size, colour, etc."*:

1. **Typography is gone.** Fonts, font sizes, letter/line spacing, paragraph spacing, indentation, colours, highlight and paragraph borders are discarded. A branded document arrives looking like default browser HTML.
2. **Underline is dropped** unless you author a style map (`u => em` etc.).
3. **Table appearance collapses.** Borders, widths, cell shading and header-row styling are "currently ignored" — only the text inside cells survives. Column widths and merges are gone.
4. **Headers, footers and page numbers vanish entirely** (issue #9 open since 2013). A contract or report loses its letterhead, footer, confidentiality line and page numbering.
5. **All pagination disappears.** No page breaks, no sections, no columns, no portrait/landscape changes, no fixed page size/margins — the panel becomes an infinite scroll, so what the user sees is *not* what Word will print.
6. **Floating/anchored images, text wrapping and z-order are lost.** A logo anchored top-right with text wrapped around it becomes an inline image in the text flow, moving content around. Image cropping/rotation is gone.
7. **Text boxes are re-flowed.** README: the contents become "a separate paragraph that appears after the paragraph containing the text box" — pull-quotes, sidebars and callouts jump into the main flow.
8. **Shapes, SmartArt, WordArt, charts and equations are not extracted at all** — they simply disappear from the preview.
9. **A field-based Table of Contents is not supported** (no field support) — a TOC either disappears or is frozen as stale text.
10. **Comments are flattened to the end of the document** with reference links, not shown as margin anchors; only if you opt in via a style map.
11. **Footnotes/endnotes** are supported by mammoth, but arrive as plain HTML constructs that the next pass may drop (see below).
12. **Security:** zero sanitisation of the source document. In a product where users upload/AI-generate `.docx`, you must run the HTML through a sanitiser (e.g. DOMPurify) before it enters your React tree — the README explicitly warns about `javascript:` links and pathological CPU/memory documents.

**Pass 2 — Tiptap's schema is the second filter.** After mammoth you have HTML; Tiptap then maps it into ProseMirror nodes/marks. **Anything with no matching extension is silently dropped or flattened** — structural wrappers like `<div class="aside">`, `<sup>` comment references, footnote anchors, `<s>`/`<u>` (unless the corresponding extension is enabled) and any inline `style=""` attributes that mammoth did emit (alignment, colours) are discarded because the rich-text schema does not model them. Even with a perfect importer, **a rich-text editor cannot represent page geometry** — which is why Tiptap itself sells a separate *Pages* extension to get pagination back.

**Pass 3 — `docx.js` export is a fresh document.** Because docx.js cannot read the original file (§3.1), export means walking your Tiptap JSON and building new `Paragraph`/`TextRun`/`Table` objects. Consequences:
- The original `styles.xml`, numbering definitions, theme, fonts and section properties are **thrown away**; Word will show your document in *your* default styles.
- Everything the editor could not hold — comments, tracked changes, headers/footers, footnotes/endnotes, floating images, text boxes, TOC, page setup — is **absent from the exported file** unless you explicitly re-construct it with docx.js APIs (which docx.js does offer as *generation* features: headers/footers, TOC fields, footnotes, comments, change tracking, text frames).
- Round-tripping repeatedly **compounds** the loss: each save re-serialises the already-degraded model.

**Net assessment:** `mammoth + Tiptap + docx` is a **content-editing** pipeline, not a **document-fidelity** pipeline. It is a reasonable choice if the product promise is "edit the *text* of an AI-generated document and hand back a clean, simple `.docx`". It will visibly fail the promise "open the user's real Word file, let them edit it, and give the same file back": letterheads, page numbers, page breaks, fonts, table borders, sidebars, floating logos, footnotes and track-changes all regress on the first round trip. If fidelity is a selling point (legal/contract/finance workflows — note Tiptap's own ["legal document automation"](https://tiptap.dev/use-cases/document-automation-for-legal) pitch), you need an OOXML-native editor.

### 5.3 For contrast — even the *paid* conversion pipeline has documented gaps

Tiptap's own [feature support matrix](https://tiptap.dev/docs/conversion/getting-started/feature-support-matrix) (Beta) is useful because it bounds what a *commercial-grade* DOCX↔rich-text pipeline can do:

- **Not converted at all:** Table of Contents (imported as a node but not rendered and **not exported**), **Text boxes**, **Shapes/SmartArt/WordArt**, document metadata, form fields/content controls, hidden text, and **right-to-left text direction** (`<w:bidi>`/`<w:rtl>` unparsed — Arabic/Hebrew/Urdu render LTR).
- **Floating images:** import `~` (partial), **editor ✕, export ✕** — i.e. even Tiptap cannot render or write floating images.
- **Footnotes / endnotes:** imported as separate REST fields, but **editor ✕ and export ✕**.
- **Sections:** `~` import, **editor ✕, export ✕**. **Paragraph borders:** ✕ everywhere.
- **Math/equations:** import **✕** (export ✓).
- **Tracked changes:** partial both ways — "tracked paragraph splits, formatting changes and moves on import, and formatting revisions and suggestions on images on export" are not carried.
- **Tabs:** import `~`, editor `~`, export **✕**.
- Page layout features (headers/footers, page breaks, page numbers, page size/margins) require the **Team-tier Pages extension** to render at all.

Even at ~$149–999/month, **page-accurate WYSIWYG round-tripping of arbitrary Word files is still an unsolved problem for a rich-text-based architecture.** That is the single most important strategic point in this report.

---

## 6. What I could **not** verify (explicit uncertainty)

1. **SuperDoc commercial pricing** — `https://www.superdoc.dev/pricing` returns 404; the license page only links to `superdocportal.dev`. AGPL-3.0 vs commercial status **is** confirmed; **price is not**.
2. **CKEditor 5 DOCX import/export specifics and current version** — I only confirmed that it is GPL-2.0-or-later-or-commercial from its licensing page. Any claim about its Word import/export fidelity would be **2024/2025-era general knowledge, not verified for 2026** in this session.
3. **OnlyOffice / Collabora 2026 versions, DOCX feature coverage and licensing specifics** — not verified here; the only thing I confirmed is the existence of a WASM client-side wrapper ([ranuts/document](https://github.com/ranuts/document), [onlyoffice-web-comp](https://github.com/electroluxcode/onlyoffice-web-comp)) and their self-descriptions.
4. **`ranuts/document` license** — not retrieved.
5. **mammoth's behaviour on tracked changes / revision marks** — not in its documented supported-feature list, and I did not find a definitive issue confirming the exact behaviour. Treat as unresolved.
6. **Tiptap's exact annual pricing** — the pricing page renders annual totals client-side as `$Value` placeholders; I verified only the monthly figures ($49 / $149 / $999) and the "Yearly (-20%)" toggle.
7. **EigenPal's "lossless roundtrip" and "pixel-perfect OOXML rendering" claims** — vendor marketing copy; no independent testing was performed. Also note the `@docx-editor.dev` brand has renamed twice in two months, so package names may churn again.
8. **Tiptap UI Components React 19 timeline** — docs say support is "currently working on"; no ETA published.
9. **The second-hand claim** that Tiptap's `@tiptap-pro/extension-import` is the correct current package: the Conversion docs supersede it with `@tiptap-pro/extension-convert-kit` + import/export extensions, while the **legacy** `extension-import`/`extension-export` are "being deprecated and will be sunset in 2026". Confirm the exact package set against your Tiptap account's registry before committing.

---

## 7. Sources

**Tiptap**
- https://github.com/ueberdosis/tiptap · https://github.com/ueberdosis/tiptap/blob/main/LICENSE.md
- https://tiptap.dev · https://tiptap.dev/docs
- https://tiptap.dev/pricing · https://tiptap.dev/feature-comparison · https://tiptap.dev/pro-license
- https://tiptap.dev/open-source-to-platform
- https://tiptap.dev/product/conversion · https://tiptap.dev/product/page-layout · https://tiptap.dev/product/review · https://tiptap.dev/product/ai-toolkit · https://tiptap.dev/product/collaboration
- https://tiptap.dev/docs/conversion/getting-started/overview
- https://tiptap.dev/docs/conversion/getting-started/feature-support-matrix
- https://tiptap.dev/docs/editor/extensions/overview
- https://tiptap.dev/docs/guides/pro-extensions
- https://tiptap.dev/docs/ui-components/getting-started/overview
- https://tiptap.dev/docs/editor/getting-started/install/react
- https://tiptap.dev/blog/release-notes/were-open-sourcing-more-of-tiptap
- https://github.com/ueberdosis/tiptap/issues/7543
- https://registry.npmjs.org/@tiptap/core/latest · https://registry.npmjs.org/@tiptap/react/latest · https://registry.npmjs.org/@tiptap/extension-collaboration/latest · https://registry.npmjs.org/@hocuspocus/server/latest
- https://hn.nuxt.dev/item/44202103 (HN thread on the MIT re-licensing)

**mammoth**
- https://github.com/mwilliamson/mammoth.js · https://raw.githubusercontent.com/mwilliamson/mammoth.js/master/README.md
- https://registry.npmjs.org/mammoth/latest · https://api.github.com/repos/mwilliamson/mammoth.js
- https://github.com/mwilliamson/mammoth.js/issues/9 (headers/footers, open since 2013)

**docx (docx.js)**
- https://github.com/dolanmiu/docx · https://raw.githubusercontent.com/dolanmiu/docx/master/README.md
- https://registry.npmjs.org/docx/latest · https://api.github.com/repos/dolanmiu/docx
- https://api.github.com/repos/dolanmiu/docx/contents/docs/usage
- https://docx.js.org/usage/patcher.md · https://docx.js.org/api/functions/patchDocument.html

**EigenPal / docx-editor.dev**
- https://www.docx-editor.dev/ · https://www.docx-editor.dev/pricing
- https://github.com/eigenpal/docx-editor · https://api.github.com/repos/eigenpal/docx-editor
- https://raw.githubusercontent.com/eigenpal/docx-editor/main/LICENSE
- https://registry.npmjs.org/@eigenpal/docx-js-editor/latest · https://registry.npmjs.org/@eigenpal/docx-editor-react/latest · https://registry.npmjs.org/@eigenpal/docx-editor-core/latest
- https://registry.npmjs.org/@docx-editor.dev/core/latest · https://registry.npmjs.org/@docx-editor.dev/react/latest

**docx-preview / docxjs**
- https://github.com/VolodymyrBaydalka/docxjs · https://raw.githubusercontent.com/VolodymyrBaydalka/docxjs/master/README.md
- https://registry.npmjs.org/docx-preview/latest · https://registry.npmjs.org/docxjs/latest (404)

**SuperDoc**
- https://github.com/superdoc/docx-editor · https://raw.githubusercontent.com/superdoc/docx-editor/main/README.md
- https://registry.npmjs.org/superdoc/latest · https://docs.superdoc.dev/resources/license/
- https://docs-v1.superdoc.dev/advanced/supereditor/overview
- https://registry.npmjs.org/@superdoc/super-editor/latest (404) · https://registry.npmjs.org/@harbour-enterprises/super-editor/latest (404)
- https://registry.npmjs.org/-/v1/search?text=super-editor

**Other**
- https://registry.npmjs.org/docxtemplater/latest · https://raw.githubusercontent.com/open-xml-templating/docxtemplater/master/README.md
- https://ckeditor.com/legal/ckeditor-licensing-options/
- https://github.com/ranuts/document · https://github.com/electroluxcode/onlyoffice-web-comp
- https://www.onlyoffice.com/blog/zh-hans/2026/05/onlyoffice-license-and-trademark-policy
