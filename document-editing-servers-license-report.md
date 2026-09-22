# Document-Editing Servers for a Commercial Closed-Source React App

**Scope:** ONLYOFFICE Docs / Document Server and Collabora Online / CODE, evaluated for embedding a
right-hand **preview + edit** panel for **docx / xlsx / pptx** inside a **commercial closed-source**
React web app (AI chat product).

**Research date: 2026-09-21.** Versions in play: **ONLYOFFICE Docs 9.4** (released 2026-05);
**Collabora Online 26.04** (latest noted 26.04.3.3, released 2026-09-14).
Anything that could only be confirmed for 2024/2025 is flagged `[STALE-RISK]`.
**I am not a lawyer; this is engineering-grade licence research, not legal advice.**

---

## 0. Bottom line

| | ONLYOFFICE Docs | Collabora Online / CODE |
|---|---|---|
| **Community/OSS licence** | **AGPL-3.0** + §7 additional terms | **MPL-2.0** (+ other OSS for deps) |
| **Copyleft strength** | Strong, network-triggered (AGPL §13) | Weak, file-level (MPL is *not* viral across your app) |
| **Closed-source app, no source release?** | **No — not the Community build.** Vendor says buy a commercial licence. | **Yes, if you build from MPL source yourself and publish only modifications to MPL-covered files.** Collabora's *own binaries* are proprietary. |
| **Vendor binaries usable commercially?** | Only with paid licence | Only with paid COOL subscription |
| **White-label / re-brand** | Enterprise/Developer only (**forbidden** in Community) | Allowed if you build from source (must strip Collabora Marks) |
| **Free build OK for production?** | Community: discouraged (no support/SLA, no scaling) | CODE: explicitly **not recommended for production** |
| **Indicative commercial price** | Docs Enterprise **from $1500**; Developer configurator showed **$3500** | COOL Business **€3.00 / £2.60 / $3.40 per user / month** (≤99 users) |
| **Backend required?** | **Yes** — storage + save callback | **Yes** — a **WOPI host** (storage + auth) |
| **Integration shape** | `api.js` + `DocsAPI.DocEditor` iframe → separate Document Server | iframe → `coolwsd` on :9980 via WOPI + postMessage |
| **Native format** | **OOXML-native** (.docx/.xlsx/.pptx/.pdf are the internal formats) | **ODF-native** (LibreOffice engine, OOXML via import/export filters) |
| **VBA macros** | **Never executed.** JS macros only, sandboxed | LibreOffice Basic / partial Excel VBA — **disabled by default since 22.04** (CVE-2025-24796) |
| **Fonts** | OS fonts; real MS fonts only if the admin installs them; silent nearest-substitute | Ships metric-compatible **Carlito/Caladea/Liberation**; genuine MS fonts absent |
| **Rendering model** | **Client-side** (browser) — lighter server, heavier browser | **Server-side tiles** — heavier server, lighter client |

**Neither product is a client-side React component.** Both are *iframe-embedded editors backed by a separate
server*, and both **require you to build and operate a file-storage + auth backend**.

### Recommendation signal
- **Lowest legal risk with a closed-source product → Collabora.** MPL-2.0 imposes no obligation on *your*
  application code. Either build from source and strip Collabora branding, or just pay ~€3/user/month.
- **Highest OOXML fidelity and least integration friction → ONLYOFFICE Docs Developer** (commercial). It is
  the edition explicitly built for embedding under your own brand in a SaaS product. But it is the one that
  **requires** a paid licence — the AGPL + additional terms make Community unusable for you.

---

## 1. ONLYOFFICE Docs / Document Server

### 1.1 Licence — exact and verified

**Community Edition = GNU Affero General Public License v3.0, with additional terms under AGPL §7.**

- Source of truth: [`ONLYOFFICE/DocumentServer` → `LICENSE`](https://raw.githubusercontent.com/ONLYOFFICE/DocumentServer/master/LICENSE)
  (HTTP 200, fetched 2026-09-21). The file is the full AGPL-3.0 text followed by
  "Copyright (C) 2009-2026 Ascensio System SIA" and an **"Additional Terms"** block explicitly stated to be
  "pursuant to Section 7 of the License".
- Additional terms, in substance:
  1. **Retention of notices/attribution** (§4, §5, §7(b)).
  2. **Modification notice** — modified versions must state modification + dates and "clearly indicate that
     they are based on the original ONLYOFFICE software developed by Ascensio System SIA".
  3. **Appropriate Legal Notices in UI** — users must be able to (i) identify ONLYOFFICE as original
     developer, (ii) understand the version may be modified, (iii) access licence information.
  4. **No trademark licence** — see [Trademark Policy](https://www.onlyoffice.com/trademark-policy).
  5. **Non-code content** (illustrations, icons, docs) under **CC BY-SA 4.0**; source code stays AGPLv3.
- Vendor-confirmed: the [Docs 9.4 release blog (19 May 2026)](https://www.onlyoffice.com/blog/2026/05/onlyoffice-docs-9-4)
  has a section "**Important license update**": *"The software is licensed under the GNU Affero General Public
  License v3.0 (AGPLv3), with additional terms that must be included in all copies and distributions."*
  → **The licence changed in May 2026. Any 2024/2025 licence analysis is `[STALE-RISK]`.**
- [Compare editions](https://www.onlyoffice.com/compare-editions) lists Community as "GNU AGPL v.3" and
  Enterprise/Developer as "Commercial".

**Enterprise/Developer = commercial proprietary licence.** Per the
[Docs Enterprise FAQ](https://helpcenter.onlyoffice.com/docs/faq/docs-enterprise.aspx): *"distributed under the
commercial proprietary license"*, for organisations *"whose policy doesn't favor public licensed software"*.

### 1.2 The AGPL problem for a closed-source SaaS (vendor's own stated position)

The [Docs Community Edition Licensing FAQ](https://helpcenter.onlyoffice.com/docs/faq/docs-community.aspx)
is unusually direct:

- *"I want to integrate your open-source product into my commercial application. Do I need to make my app's
  code public?"* → *"If you use our open-source products under the AGPL v3 license, and your application is
  distributed or made available over a network (SaaS model), you must make the source code of the modified
  version and any derivative work available under AGPL v3. **If you do not want to open your source code,
  you should consider obtaining a commercial license**."*
- Under AGPL v3 per the FAQ: modify ⇒ publish; network interaction ⇒ users must have source access;
  **you need to keep the branding**; derivative works must be AGPLv3; no additional restrictions.
- *"If your product is proprietary and you do not want to disclose source code, **a commercial license is
  required**."*
- *"Can I white-label or remove branding in the Community version?"* → *"**No.** The Community version must
  retain original branding and copyright notices. White-labeling and branding removal are available only
  under commercial licensing terms."*

**Engineering nuance (raise with counsel, don't rely on it):** a strict reading of AGPL-3.0 §13 triggers the
source-offer obligation *"if you modify the Program"*. Running an **unmodified** Document Server and merely
`iframe`-embedding its editor is arguably neither modification nor conveyance, so some integrators take the
position that pure unmodified iframe embedding is permissible. **But** (a) ONLYOFFICE's own FAQ asserts the
broader "network availability ⇒ source" framing, and (b) the §7 additional terms' UI-attribution and
no-rebranding requirements would still bite. Because the vendor's stated position is the one you would have to
argue against, the **commercial licence is the practical answer**.

### 1.3 Community vs Enterprise vs Developer — what is actually limited

Sources: [Compare editions](https://www.onlyoffice.com/compare-editions),
[Community Licensing FAQ](https://helpcenter.onlyoffice.com/docs/faq/docs-community.aspx),
[Developer Edition](https://www.onlyoffice.com/developer-edition),
[Docs Enterprise](https://www.onlyoffice.com/docs-enterprise).

**Absent/limited in Community, present in Enterprise & Developer:**
- **Mobile web editors** — Community has none. FAQ: *"Enterprise Edition includes mobile web editors not
  present in Community Edition."*
- **White Label** — Enterprise/Developer only.
- **Admin Panel** — Community configures via config files/server requests; E/D get the web Admin Panel
  (statistics, health checks, security settings, WOPI settings, AI settings, file-size limits, final server
  config, PDF-form signing certificate, font management).
- **Clustering / enterprise scalability / HA / multi-server / disaster recovery / multi-tenancy** — E/D.
- **Live Viewer** — Enterprise "upon request".
- **Support & SLA** — Community has *"no guaranteed technical support or SLAs"*.
- Community is *"intended for testing, personal use, development, and small-scale deployments."*

**Shared by all three:** Document/Spreadsheet/Presentation/PDF editors, Form creator, Diagram viewer,
conversion service, document builder service, Automation API, two co-editing modes, comments, chat, review &
track changes, version history, document comparison & combining, plugins, macros, JWT, HTTPS.

**⚠️ Concurrency limit — CHANGED IN 9.4; vendor pages are stale.**
- Changelog **9.4.0**, *Back-end*: *"**Removed the limitation of 20 simultaneously opened documents**"*
  ([`CHANGELOG.md`](https://raw.githubusercontent.com/ONLYOFFICE/DocumentServer/master/CHANGELOG.md)).
- Blog: *"Starting with version 9.4, the open-source Community version also removes the 20 simultaneous
  connection limit."*
- **BUT** the [Enterprise FAQ](https://helpcenter.onlyoffice.com/docs/faq/docs-enterprise.aspx) still says
  *"(only 20 for the free Community version)"* and [Compare editions](https://www.onlyoffice.com/compare-editions)
  still says Community "Number of users: up to 20 recommended". **Both are out of date as of 2026-09-21.**
  `[STALE-RISK]` applies to those two pages only.
- The Enterprise page still frames licensing as *"Simultaneous connection based license"* — *"one document
  opened by two users means two simultaneous connections"*; over the limit *"each next document opens in
  read-only mode."*

### 1.4 Pricing (published, but configurator-driven)

Both pages live-fetched 2026-09-21; prices render via JavaScript. Treat figures as **indicative defaults**.

- **[Docs Enterprise pricing](https://www.onlyoffice.com/docs-enterprise-prices):** headline **"From $1500"**.
  The on-page configurator's default selection (On-premises; **50 connections**; 1-year licence; 1 year of
  support & updates; Basic; no scalability add-ons) displayed **Total: $2100**. Options: Cloud/On-premises,
  connection count, 1-year or **lifetime**, 1 or 3 years of updates, Basic/Plus/Premium, disaster recovery,
  multi-server deployment, Live viewer, training.
- **[Docs Developer pricing](https://www.onlyoffice.com/developer-edition-prices):** **no "From" price is
  published.** Default configuration displayed **Total: $3500**. Options: Development vs Production purpose,
  **"20 connections per each server"**, Standard vs **White Label**, multi-tenancy, disaster recovery,
  multi-server, support level, Automation API access, Live viewer, native mobile apps, desktop apps, training.
- **Per-connection unit pricing is NOT published** — it is a configurator/quote flow. Tier breakpoints and
  volume discounts: **not published / not verified.**
- A commercial proprietary licence text exists at
  [help.onlyoffice.com](https://help.onlyoffice.com/products/files/doceditor.aspx?fileid=4995927) (login-gated;
  contents **not verified**).

### 1.5 Deployment & server requirements

Sources: [Community Docker reqs](https://helpcenter.onlyoffice.com/docs/installation/docs-community-sys-reqs-docker.aspx),
[Enterprise Docker reqs](https://helpcenter.onlyoffice.com/docs/installation/docs-enterprise-sys-reqs-docker.aspx),
[Enterprise Docker install](https://helpcenter.onlyoffice.com/docs/installation/docs-enterprise-install-docker.aspx),
[Docker-DocumentServer README](https://raw.githubusercontent.com/ONLYOFFICE/Docker-DocumentServer/master/README.md).

**Docker images:** `onlyoffice/documentserver` (Community) · `onlyoffice/documentserver-ee` (Enterprise) ·
`onlyoffice/documentserver-de` (Developer). Base image `ubuntu:24.04`.

**Hardware:**
- Baseline: single core 2 GHz (dual core recommended), **RAM 4 GB+**, **SWAP ≥ 4 GB**, **HDD ≥ 40 GB free**.
- Community scaling table: <100 concurrent → 1 core @2.8 GHz / 4 GB / 40 GB; 100–200 → dual core / 4 GB /
  80 GB; 200–400 → quad core / 4 GB / 160 GB; **400+ → cluster/Kubernetes recommended**.
- **Discrepancy:** the Docker README says **HDD ≥ 2 GB**, swap ≥ 2 GB; the Help Center says **40 GB**.
  Unreconciled — plan for 40 GB.
- "Concurrent active user" = any user with a document **open** (editing *or* viewing).

**Bundled services — changed in 9.4:**
- **Enterprise/Developer images bundle PostgreSQL + RabbitMQ + Redis (+ nginx)**. Docker README:
  *"Enterprise/Developer Edition — PostgreSQL, RabbitMQ and Redis are bundled in the image."* Volumes:
  `/var/lib/postgresql`, `/var/lib/rabbitmq`, `/var/lib/redis`, `/var/log/onlyoffice`,
  `/var/www/onlyoffice/Data`, `/var/lib/onlyoffice`.
- **Community 9.4+ no longer needs RabbitMQ or a database.** Changelog 9.4.0: *"Consolidated components into
  a single process… **Removed dependency on RabbitMQ**… **Removed dependency on databases**… Removed code
  minification."* Blog: *"Simplified architecture… Removed dependencies on RabbitMQ and databases."*
  → **Major change vs. all pre-9.4 documentation.**
- Enterprise can use external services: `DB_TYPE` = `postgres|mariadb|mysql|mssql|oracle`,
  `AMQP_TYPE` = `rabbitmq|activemq`, plus `DB_HOST`, `REDIS_SERVER_HOST`, …
- **Admin panel and test example are OFF by default** (`ADMINPANEL_ENABLED=false`, `EXAMPLE_ENABLED=false`).
- **⚠️ Kubernetes:** the Help Center states the *"image is not compatible with Kubernetes installation"*;
  a separate guide exists ([ONLYOFFICE/kubernetes-docs](https://github.com/ONLYOFFICE/kubernetes-docs)).
- Port 80 default; HTTPS via mounted certs or Let's Encrypt. Set `JWT_SECRET` explicitly — since 7.2 a
  random secret is regenerated on reboot if unset.

**Docker image size: NOT VERIFIED.** `hub.docker.com` (UI and `/v2/` JSON API) was unreachable from this
environment for all three images.

### 1.6 Integration model — a backend is mandatory

Sources: [How it works](https://api.onlyoffice.com/docs/docs-api/get-started/how-it-works/),
[Basic concepts](https://api.onlyoffice.com/docs/docs-api/get-started/basic-concepts/),
[Callback handler](https://api.onlyoffice.com/docs/docs-api/usage-api/callback-handler/),
[Saving file](https://api.onlyoffice.com/docs/docs-api/get-started/how-it-works/saving-file/),
[Security](https://api.onlyoffice.com/docs/docs-api/get-started/how-it-works/security/).

**Shape:** a **separate Document Server host** embedded as an **iframe** driven by the JavaScript API
(`DocsAPI.DocEditor`). Official React guidance:
[Frontend frameworks → React](https://api.onlyoffice.com/docs/docs-api/get-started/frontend-frameworks/react/).

**Division of responsibility (official):**
> "ONLYOFFICE Docs includes the **document editor, document editing service, document command service,
> document conversion service, and document builder service**. The **document manager and document storage
> service must be provided by an integrator.**"

So **you must build**: (1) a **document storage service**; (2) a **document manager** (your document list UI);
(3) a **callback handler** at `editorConfig.callbackUrl`.

**Save round-trip (official):** user edits → editor sends changes to the editing service → user closes →
service compiles the final document → POSTs to `callbackUrl` with `status` and a `url` → **your server
downloads that `url` and stores it** → your server **must respond `{"error": 0}`** or the editor errors.
Save fires ~**10 s** after editing ends (5 s conversion start delay + conversion time). `forcesave` (status 6)
snapshots before close, via the command service, a Save button, or auto-assembly.

**Config essentials:** `document.fileType`, `document.key`, `document.title`, `document.url`,
`documentType: "word" | "cell" | "slide" | "pdf"`, `editorConfig.callbackUrl`, `token`.

**JWT is on by default.** *"If the token is missing or invalid, the request will be rejected."* Header
defaults to `Authorization` (`JWT_HEADER`); `JWT_IN_BODY` default `false`; **`ALLOW_PRIVATE_IP_ADDRESS`
defaults to `false`**, so the Document Server cannot fetch from private IPs unless you opt in — relevant to
your own storage-service design.

**Alternative: WOPI.** ONLYOFFICE also supports WOPI
([Using WOPI](https://api.onlyoffice.com/docs/docs-api/using-wopi/overview/), `WOPI_ENABLED` default `false`).
Either protocol still requires you to be the host/storage.

### 1.7 docx / xlsx / pptx fidelity notes

**Architecture advantage — OOXML-native.** ONLYOFFICE's internal formats are `.docx/.xlsx/.pptx` (and `.pdf`).
Official: *"the edited file is converted into one of the editors' native formats (`.docx`, `.xlsx`, `.pptx`, or
`.pdf`)."* Vendor history: DOCX/XLSX/PPTX were chosen as native internal formats in 2011, interim formats
abandoned in 2013, and it does *"native OOXML editing without any intermediate conversion"*
([16th-anniversary post](https://www.onlyoffice.com/blog/2026/06/the-moment-we-chose-compatibility-over-isolation)).
The [9.0 FAQ](https://www.onlyoffice.com/blog/zh-hans/2025/06/onlyoffice-docs-9-0-faq) states: *"DOCX, XLSX and
PPTX are the native formats of the document, spreadsheet and presentation editors respectively."*
Vendor marketing claims *"100% view, print, and pagination fidelity"* on the
[Developer Edition page](https://www.onlyoffice.com/developer-edition) — **vendor claim, not independently verified.**

**Supported input formats** include `doc, docm, docx, dot/dotm/dotx, odt, ott, rtf, txt, epub, fb2, hwp/hwpx,
pages, wps/wpt, md, fodt, xml` (documents); `xls, xlsb, xlsm, xlsx, xlt/xltm/xltx, ods, ots, csv, numbers,
et/ett` (sheets); `ppt, pptm, pptx, pot/potm/potx, pps/ppsm/ppsx, odp, otp, fodp, key, dps/dpt, sxi` (slides)
([Document config reference](https://api.onlyoffice.com/docs/docs-api/usage-api/config/document/)).

**Verified caveats:**

- **Non-OOXML formats are converted for the session.** Help Center (Document Editor / Presentation Editor):
  *"While uploading or opening the file for editing, it will be converted to the Office Open XML (DOCX)
  format. It's done to speed up the file processing and increase the interoperability."*
  ([Document](https://helpcenter.onlyoffice.com/docs/userguides/document_editor/SupportedFormats.aspx),
  [Presentation](https://helpcenter.onlyoffice.com/docs/userguides/presentation_editor/SupportedFormats.aspx))
  → **OOXML in = native round trip. ODF/legacy/other in = conversion drift.** Third party (Aug 2026):
  *"ODF files are internally converted to OOXML — complex LibreOffice templates can suffer during round-trips."*
  ([datazone.de](https://datazone.de/en/aktuelles/nextcloud-office-collabora-onlyoffice/))
- **`.doc` cannot be saved as `.doc`.** *"the `.doc` format, which is not available for saving in ONLYOFFICE
  Docs"*. With `assemblyFormatAsOrigin` (default `true` since 7.0) a legacy original is converted back, and
  **if that conversion fails the editor shows a "rollback to save changes to ooxml" warning and keeps the
  OOXML file** — i.e. **silent format substitution is possible for legacy files.** Do not treat
  `.doc/.xls/.ppt` as byte-stable round-trip.
- **`.docx` round-trip is not lossless — one vendor-confirmed bug.** Issue
  [#3382](https://github.com/ONLYOFFICE/DocumentServer/issues/3382) (v9.0.3, created 2025-08-19): a complex
  `.docx` rendered perfectly on open, but *"after saving the document in ONLYOFFICE and reopening it again in
  MS Word, the layout is affected, especially in the header and footer sections… the round-trip back to MS
  Word breaks the formatting."* Labelled confirmed-bug and fixed; closed 2025-09-12. **Failure mode is
  layout/pagination (notably headers/footers), not content loss.** Vendor acknowledges residual Word gaps but
  provides **no compatibility warning feature** — asked in the 9.0 FAQ whether they could flag incompatible
  components, ONLYOFFICE answered only *"if you encounter Word compatibility issues, please tell us the details
  via the forum."*
  Counter-evidence (favourable): an independent 2026 test reports a 30-page contract `.docx` with tracked
  changes, tables, headers/footers and images *"rendered nearly identically to Word"* in ONLYOFFICE, while
  Collabora shifted a few page breaks ([selfhosting.sh](https://selfhosting.sh/compare/collabora-vs-onlyoffice/)).
- **⚠️ VBA macros never run. JS macros only.** Per the
  [Macros FAQ](https://api.onlyoffice.com/docs/macros/more-information/faq/): macros use **JavaScript** via the
  Office JS API. *"Can I use my Microsoft Office (VBA) macros in ONLYOFFICE? **Not directly**… You can convert
  them… using the built-in AI plugin converter… or manually."* Macros *"run as JavaScript code inside the
  editor window and have no access to the system. They cannot read or write files, make network requests, or
  interact with the operating system directly"*, are **document-scoped (cannot be global)**, and since v7.1 run
  in **strict mode**.
  **VBA payload preservation is NOT guaranteed by any official statement.** Hard evidence is bug
  [#3466](https://github.com/ONLYOFFICE/DocumentServer/issues/3466) *"OnlyOffice breaks Excel VBA macros after
  editing"* (v8.3.2.19, 2025-10-20): saving added a spurious `ThisWorkbook1` module and **all VBA stopped
  working in Excel** — reporter called it permanent corruption. Confirmed-bug, fixed, closed 2025-12-03.
  → **Treat macro-enabled files as high-risk.** (Note: a third-party comparison claiming ONLYOFFICE has
  "VBA-compatible macros" **contradicts ONLYOFFICE's own docs** and is unreliable.)
- **Fonts: no Microsoft fonts by default; silent nearest-substitute.** Official install guides: *"By default,
  ONLYOFFICE Docs uses embedded free fonts from the operating system where ONLYOFFICE Docs is installed."* The
  installer looks for **real MS font files** — `arial.ttf`, `calibri.ttf`, `cour.ttf`, `symbol.ttf`,
  `times.ttf`, `wingding.ttf` — and uses them if present. *"In case the document contains fonts absent from the
  ONLYOFFICE Docs computer, it will upload the closest font substitute (the document layout and display might
  suffer from such substitution)."*
  ([Windows](https://helpcenter.onlyoffice.com/docs/installation/docs-community-install-fonts-windows.aspx),
  [Linux](https://helpcenter.onlyoffice.com/docs/installation/docs-community-install-fonts-linux.aspx))
  My own primary-source check of the bundled font source tree
  ([`ONLYOFFICE/core-fonts`](https://api.github.com/repos/ONLYOFFICE/core-fonts/contents/)) shows
  **metric-compatible open substitutes**: `crosextra` (**Carlito** ≡ Calibri metrics), `caladea`
  (≡ Cambria), `liberation` (≡ Arial/Times/Courier), `dejavu`, `noto`, `openoffice`, `opensans`,
  `ubuntu-font-family`, plus non-Latin sets — and **no Arial/Times/Calibri/Cambria**.
  → Expect **metric-compatible substitution, not identical glyphs/line breaks**. Admins can install real fonts
  into the OS and re-run `documentserver-generate-allfonts.sh`. **Note:** no *official* page explicitly states
  that Carlito/Caladea ship with Community — the repo contents are the evidence; the docs frame it as "OS fonts".
  **Licensing MS fonts for server-side rendering is your problem.**
- **PPTX: natively modelled, but several *still-open* vendor-confirmed defects.**
  - [#3539](https://github.com/ONLYOFFICE/DocumentServer/issues/3539) — complex-script **theme fonts ignored**
    in PPTX (Hebrew gets the Latin theme font; DOCX works). v9.1.0, opened 2025-12-16, **open**, updated 2026-08-08.
  - [#3349](https://github.com/ONLYOFFICE/DocumentServer/issues/3349) — no support for
    `blipFill → blip → duotone → schemeClr` in **slide background**, causing **unreadable text** on a stock
    Microsoft template; also images not displayed. Opened 2025-07-17, **open**.
  - [#3218](https://github.com/ONLYOFFICE/DocumentServer/issues/3218) — on export, **some images missing or
    stretched and some equations missing**; *"the PPTX version has all images, but the equations are not visible."*
  - [#3206](https://github.com/ONLYOFFICE/DocumentServer/issues/3206) comments not visible in `.pptx` (**open**).
  - [#2654](https://github.com/ONLYOFFICE/DocumentServer/issues/2654) background image missing (**open**).
  - [#2092](https://github.com/ONLYOFFICE/DocumentServer/issues/2092) indentation,
    [#1971](https://github.com/ONLYOFFICE/DocumentServer/issues/1971) gradient display,
    [#1902](https://github.com/ONLYOFFICE/DocumentServer/issues/1902) table size changes — 2022–2023,
    **still open**.
  - Vendor's own 9.0 FAQ backlog lists *"full SmartArt editor"* and *"chart editing"* as **requested** features
    → incomplete SmartArt/chart authoring.
  - Third-party ranking: PPTX "Very good — close to PowerPoint rendering"
    ([selfhosting.sh](https://selfhosting.sh/compare/collabora-vs-onlyoffice/)).
- **Embedding/charts/equations/OLE:** OLE objects are first-class in the API
  ([CreateOleObject](https://api.onlyoffice.com/docs/office-api/usage-api/document-api/Api/Methods/CreateOleObject/),
  [InsertOleObject](https://api.onlyoffice.com/docs/plugins/interacting-with-editors/document-api/Methods/InsertOleObject/),
  [AddOleObject](https://api.onlyoffice.com/docs/plugin-and-macros/interacting-with-editors/methods/common-api/api/addoleobject/)) —
  this confirms **API support, not rendering fidelity**. Vendor feature-request backlog lists
  *"drawing trend lines in charts"* as outstanding.
- **Large files:** no official ONLYOFFICE file-size page found. Third party (updated 2026-06-07): the server
  rejects with *"file size exceeds the limit set for the server"*, and browsers OOM/crash (*"Aw, Snap!"*) on
  ~25 MB `.docx`; cause is decompressed XML + client-side layout memory (rule of thumb 1 MB file ≈ 5–20 MB RAM),
  amplified by high-res images, OLE objects (Visio/Excel/PDF), track-changes and nested tables. Limits live in
  `/etc/onlyoffice/documentserver/default.json` → `inputLimits` on **uncompressed zip** size
  ([source](https://cloud.tencent.cn/developer/article/2645964)). **Since rendering is client-side, browser
  memory — not server RAM — is your scaling ceiling for big documents.**
- Vendor 9.0 performance claims: DOCX creation 35% faster, XLSX 10%, PPTX 27%. Third-party: 5 MB XLSX opens in
  3–5 s (ONLYOFFICE) vs 5–8 s (Collabora); ~500 MB idle RAM vs ~1.3 GB.

---

## 2. Collabora Online / Collabora Online Development Edition (CODE)

### 2.1 Licences — three distinct layers

**Layer 1 — The `online` source repository: MPL-2.0.**
[`CollaboraOnline/online` → `COPYING`](https://github.com/CollaboraOnline/online/blob/main/COPYING) contains in
full: *"This Source Code Form is subject to the terms of the Mozilla Public License, v. 2.0."* (GitHub's licence
API reports SPDX `NOASSERTION` because it is a one-line pointer file, not the full text; the text is unambiguous.)
The [mirror README](https://raw.githubusercontent.com/CollaboraOnline/online.mirror/main/README.md) confirms:
*"Open Source – **primarily under the MPLv2** license. Some parts are under other open source licences, see e.g.
[browser/LICENSE](https://github.com/CollaboraOnline/online/blob/main/browser/LICENSE)."*
→ **Not all files are MPL-2.0.** Audit `browser/LICENSE`, `THIRDPARTYLICENSES`, `CODA-THIRDPARTYLICENSES.html`
and per-component `LICENSE` files before redistributing a build.

**Layer 2 — The LibreOffice core engine: MPL-2.0.**
[LibreOffice Licenses](https://www.libreoffice.org/licenses/): *"LibreOffice is made available subject to the
terms of the **Mozilla Public License v2.0**… based on code from Apache OpenOffice… but also includes software
which differs from version to version under a large variety of other Open Source licenses."* The
[Collabora Online MPLv2 terms page](https://www.collaboraonline.com/terms/collabora-online-mplv2/) lists MPL 2.0
alongside MPL 1.1, Apache 2.0, CPL 1.0, SIL OFL 1.1, CC-BY-SA 3.0/4.0, and notes *"Copyright © 2000–2025
LibreOffice contributors"*.

**Layer 3 — Collabora's distributed binaries: proprietary, with additional conditions.**
This is the finding that most changes the decision. The
[Collabora Online MPLv2 terms page](https://www.collaboraonline.com/terms/collabora-online-mplv2/)
(page last modified 2026-07-02; fetched 2026-09-21) states:

> *"Collabora Online MPLv2 **Source Code Form** is licensed primarily pursuant to the Mozilla Public License
> v2.0 licence… In contrast **Executable Forms**, as set out in MPLv2 paragraph 3.2(b), are distributed with
> additional conditions under a proprietary license. As such the Source Code Form of the Software is made
> available at all times under the terms of such licences."*

and

> *"**No right or licence**, express or implied, is granted… with respect to any trademark, trade name or
> service mark ("Mark") of Collabora. **If You distribute any open source component of the Software, You must
> remove all Marks** except those used to identify Collabora's ownership or licensing of the component."*

Collabora staff on the [forum](https://forum.collaboraonline.com/t/code-binary-license/179):
*"The trademark and CSS theming are owned by Collabora and are proprietary… The intention is that our CODE
binaries are not suitable for use in production in the enterprise as a 'free of charge' alternative that allows
companies to avoid contributing to COOL. **If you want that — you need to build, brand and support it yourself
from the MPLv2 source.**"* `[STALE-RISK: statement is from 2021]` — but it matches the current 2026 terms text.

**Net effect:** the **MPL-2.0 source** is what you may use freely; **Collabora's binaries** carry Collabora's
own proprietary conditions.

### 2.2 MPL-2.0 vs AGPL-3.0 for a closed-source app

MPL-2.0 §3.2 ("Distribution of Executable Form") requires covered software distributed in executable form to
*also* be made available in **Source Code Form**, and that the executable licence *"does not attempt to limit or
alter the recipients' rights in the Source Code Form"*. Critically:
- The obligation is **per-file / file-level** ("Covered Software"; "Modifications" = changed or new files
  containing Covered Software). It does **not** reach your separate application code.
- MPL §3.3 explicitly permits a **"Larger Work"** combining Covered Software with other material *"under terms
  of Your choice"*.
- There is **no network/SaaS clause** equivalent to AGPL §13.

→ **A closed-source commercial app can embed a from-source MPL-2.0 Collabora build without releasing its own
source.** Conditions: (1) **modifications to MPL-covered files** must be offered in source form with notices
intact; (2) don't strip licence notices (MPL §3.4); (3) **remove Collabora trademarks/branding** if you
distribute an open-source component; (4) audit the non-MPL components.

**Practical compliance mechanics:** community guidance
([forum, Aug 2026](https://forum.collaboraonline.com/t/questions-about-mpl-2-0-source-availability-for-a-modified-collabora-online-web-ui/4945))
is that files delivered to the browser (HTML/CSS/JS/translations) count as "distributed", so you must offer their
corresponding source by reasonable means. **Collabora declined to give case-specific compliance advice** and
recommended publishing modifications publicly or buying the commercial white-label service. Treat ZIP-on-request
vs public-repo mechanics as a lawyer question.

### 2.3 CODE vs Collabora Online (COOL)

Sources: [Collabora FAQs](https://www.collaboraonline.com/faqs/) (modified 2026-07-06),
[CODE page](https://www.collaboraonline.com/code/) (modified 2026-09-10).

**CODE — free:**
- *"CODE is the development version of Collabora Online… **not recommended for production environments**."*
- *"a continuously updated, **rolling release**… **no SLA or long term support**."* Analogy: *"CODE would be
  like our Fedora or openSUSE version — rather than RHEL or SLES."*
- *"Most of the core functionality will be identical… in many cases CODE will have more up-to-date functionality
  than Collabora Online even, but… not recommended for anyone who needs a stable supported release."*
- Feature bullets: free to test/sandbox, first access to cutting-edge features, community, *"supports
  collaboration within small teams or individual users."*
- Staff ([Sep 2025](https://forum.collaboraonline.com/t/questions-about-using-collabora-code-in-production-licensing-branding-nextcloud/4033)):
  *"for production use, we recommend COOL, which is the still open source version with paid support"* and
  *"CODE is the community version for testing / home use, COOL is meant to be used for production."*

**COOL — commercial subscription:** SLA, maintenance, **Long Term Support**, software and security updates,
**signed security updates**, lifetime maintenance, customer portal, accessibility certification. FAQ: *"minor
upgrade roughly every month… major releases every six months or so. Users are supported on each major version
for three years."*

**Branding:** you **may** modify and rebrand **provided you comply with the licence** — i.e. build from MPL
source and strip the Marks. Collabora also sells *"a maintained, white-labelled version of COOL"* commercially.
**If you use Collabora's binaries, the Marks must go.**

**Current release line: 26.04** (latest noted 26.04.3.3, 2026-09-14).

### 2.4 Pricing

From the official [Subscriptions page](https://www.collaboraonline.com/subscriptions/) (modified 2026-07-08;
price markup extracted 2026-09-21). Three currency tabs (EUR / GBP / USD):

| Tier | Users | Price (as published) |
|---|---|---|
| **CODE** | — | **FREE** (all three tabs) |
| **Collabora Online for Business** | **up to 99** | **€3.00 / £2.60 / $3.40 per user per month** |
| **Collabora Online for Enterprise** | **100+** | **"Personalised Volume Discounts"** — contact sales |

Business tier features: On premise, Up to 99 users, Long Term Support, Service Level Agreement, Signed Security
Updates, Lifetime Maintenance, Customer Portal, Accessibility Certified. Enterprise adds: 100+ users, Roadmap
Input, Integration Support, Customisation, Desktop Version, Mobile Support. Page also states: education/NGO
special pricing on request, and *"Three year multi-year discounts are available on request."*

**Not verified:** VAT treatment, minimum seats, minimum term, or whether a cheaper editor-only tier exists.
**Any invoice-level detail: price not published / not verified.**

### 2.5 Docker images & resource requirements

**Images:** `collabora/code` for CODE (confirmed via Docker Hub layer metadata, e.g.
[collabora/code image layers](https://hub.docker.com/layers/collabora/code/21.11.1.4.1/images/sha256-82db36547434ed5fc826328be41658955b5505eaf5b7700de98d03eee1e16827)).
The commercial build ships to subscribers. **The exact COOL image name and all image sizes could NOT be verified**
— Docker Hub's API/UI was unreachable from this environment.

**Install paths** (from the [CODE page](https://www.collaboraonline.com/code/)):
- **Docker / Docker Compose**: official compose bundles for CODE + OpenCloud, ownCloud, Seafile, HumHub,
  GroupOffice. Service exposed on **`:9980`**.
- **Native Linux packages**: deb at `https://www.collaboraoffice.com/repos/CollaboraOnline/CODE-deb`, rpm at
  `.../CODE-rpm`, GPG key `collaboraonline-release-keyring.gpg`. Minimal install: **`coolwsd`** + **`code-brand`**;
  full install adds `collaboraoffice*`. deb supports amd64/ppc64/arm64; rpm amd64 only.
- Config: **`/etc/coolwsd/coolwsd.xml`**; service via **systemd** (`systemctl restart coolwsd`); logs
  `journalctl -u coolwsd`. Default config expects an SSL cert that isn't present — the guide suggests disabling
  SSL in coolwsd and terminating TLS at a reverse proxy.
- Also available as a **virtual appliance**.
- One 2026 datapoint surfaced via search indexing of the official manual (**not directly verifiable**):
  *"Starting with Collabora Online 26.04.2.2.1, the Docker images are built on a minimal, distroless base image."*

**⚠️ Resource sizing — two official-ish sources disagree by ~3×.** Flagging this explicitly:

| Source | CPU | RAM | Notes |
|---|---|---|---|
| **openDesk / ZenDiS** (German gov, Apache-2.0) — [scaling.md](https://raw.githubusercontent.com/opendesk-edu/opendesk-edu/main/docs/scaling.md) | **1 vCPU per 15 active users** | **50 MB per active user** | ~1 Mbit/s per 10 active users |
| Third-party/vendor-partner synthesis — [datazone.de](https://datazone.de/en/aktuelles/nextcloud-office-collabora-onlyoffice/), [selfhosting.sh](https://selfhosting.sh/compare/collabora-vs-onlyoffice/) | **~1 core per 5 active editors** | **~300–500 MB per active session**, ~1.3 GB idle baseline | caches the LibreOffice core in shared memory |

**I could not read Collabora's own official sizing guidance**: `sdk.collaboraonline.com` (including
`CO-SDK-manual.pdf`) is behind **Anubis proof-of-work bot protection** and returned a challenge page every time.
A HCL Connections
[sizing guide](https://help.hcl-software.com/connections/latest/admin/install/t_collabora_online_resource_requirements.html)
also exists but its content was not reachable. **Plan your capacity test around the higher figure and validate
empirically.** `[STALE-RISK]` on any pre-2026 number.

### 2.6 Integration model — a WOPI host is mandatory

Sources: [online.mirror README](https://raw.githubusercontent.com/CollaboraOnline/online.mirror/main/README.md),
[Collabora FAQs](https://www.collaboraonline.com/faqs/), [SDK docs](https://sdk.collaboraonline.com/) (bot-protected).

**Shape:** `coolwsd` ("COOL Web Services Daemon") is a **standalone server** the browser reaches over
WebSocket/HTTP (default **port 9980**); your app embeds it in an **iframe** and communicates via the
**postMessage API**. `/hosting/discovery` is the WOPI discovery endpoint.

**A WOPI host is mandatory.** Collabora staff: *"You need some file storage & authentication solution, since COOL
only provides document editing for you. It may be something else than Nextcloud… In an extreme case, you can also
implement your own integration, as long as you have a preferred solution that handles authentication & file
storage."* → You must implement the **WOPI endpoints** (`CheckFileInfo`, `GetFile`, `PutFile`, …) plus auth and
storage. The SDK documents a
[step-by-step tutorial](https://sdk.collaboraonline.com/docs/Step_by_step_tutorial.html),
[postMessage API](https://sdk.collaboraonline.com/docs/postmessage_api.html) and
[available integrations](https://sdk.collaboraonline.com/docs/available_integrations.html) — **these are behind
Anubis and were NOT read; cited as pointers only.** Note `help.collaboraonline.com` does not resolve (DNS
ENOTFOUND) — Online admin docs live on the bot-protected SDK host; Collabora *Office* help is at
`help.collaboraoffice.com`.

**SSL must match:** the README warns that if SSL is enabled on either side, **both** Collabora Online and the
integration must use HTTPS.

**Security / operational facts:**
- The document engine runs inside a **chroot jail** ("kit") per document — the README documents
  `coolwsd-systemplate-setup` and a `kit/` component, and Docker ships a **seccomp profile**
  (`docker/cool-seccomp-profile.json`). Meaningful hardening for hosting untrusted documents.
- **Admin console:** `browser/dist/admin/admin*.html` (HTTP Basic auth), WebSocket at `/adminws/` requiring a
  JWT that expires every 30 minutes.
- **Macro execution is OFF by default** — see §2.7; this is a deliberate security posture after CVE-2025-24796.
- The repo generates **SBOMs** for server, browser client, engine and container images
  ([SBOM.md](https://github.com/CollaboraOnline/online.mirror/blob/main/SBOM.md)) — useful for your own licence audit.
- **Source lives on Gerrit, not GitHub**: *"Active development of Collabora Online has moved to our Gerrit
  instance at gerrit.collaboraoffice.com"*; GitHub is for issues + release artifacts (nightly container image,
  Helm chart). Read-only mirror: [CollaboraOnline/online.mirror](https://github.com/CollaboraOnline/online.mirror).
  **Build from the Gerrit/mirror tree, not the issue repo.**

### 2.7 docx / xlsx / pptx fidelity notes

**Architecture:** Collabora Online is **LibreOffice in the browser** — headless LibreOffice streaming rendered
tiles, with every keystroke processed server-side. LibreOffice's **native/default formats are ODF**
(`.odt/.ods/.odp`); OOXML support is via **import/export filters**. So `.docx/.xlsx/.pptx` are converted into an
internal model on open and re-serialised on save — the round-trip risk that does **not** apply to
ONLYOFFICE's OOXML-native engine.

Evidence that OOXML is genuinely round-tripped through ODF, from Collabora's own release notes: 26.04.3.3 fixed
*"**Saving a PPTX to ODF and back** could turn every shape green, and shapes that used a theme colour lost the
link to the theme."* ([26.04 release notes](https://www.collaboraonline.com/collabora-online-26-04-release-notes/))

**Officially listed supported formats** (identical on the [CODE page](https://www.collaboraonline.com/code/) and
[Collabora Online page](https://www.collaboraonline.com/collabora-online/), fetched 2026-09-21):

| App | Listed formats |
|---|---|
| Writer | `.odt, .docx, .doc, .pdf, .rtf` |
| Calc | `.ods, .xlsx, .xls, .xlsm, .csv` |
| Impress | `.odp, .ppt, .pptx` |
| Draw | `.odg, .vsd, .vsdx` |

**⚠️ The vendor's own conversion-limitations page is the single most important fidelity document.**
[About Converting Microsoft Office Documents](https://help.collaboraoffice.com/latest/en-GB/text/shared/guide/ms_import_export_limitations.html) (26.04):
> *"some layout features and formatting attributes in more complex Microsoft Office documents are handled
> differently in Collabora Office or are unsupported. As a result, **converted files require some degree of
> manual reformatting. The amount of refactoring that can be expected is proportional to the complexity of the
> structure and formatting of the source document.**"*

Explicit conversion-challenge lists from that page:
- **Word:** AutoShapes; revision marks; OLE objects; certain controls and MS Office form fields; indexes; tables,
  frames and multi-column formatting; hyperlinks and bookmarks; WordArt graphics; animated characters/text.
- **PowerPoint:** AutoShapes; tab/line/paragraph spacing; **master background graphics**; grouped objects;
  certain multimedia effects.
- **Excel:** AutoShapes; OLE objects; certain controls/form fields; **pivot tables**; new chart types;
  conditional formatting; some functions/formulae.
- Also: *"**Collabora Office cannot run Visual Basic Scripts**, but can load them for you to analyse."*

**Official round-trip integrity data — read with care.** Collabora runs
[mso-test](https://www.collaboraoffice.org/mso-test/): *"round-trips document files with Collabora Online and
tests their integrity in MS Office."* Latest column 2026-09-03, version 26.04.3.2:

| Round trip | Tested | Conversion failed | **Open-failed-after-conversion** | Succeeded |
|---|---|---|---|---|
| DOCX→DOCX | 13,022 | 27 | **0** | 12,995 |
| DOC→DOCX | 12,879 | 153 | 11 | 12,715 |
| XLSX→XLSX | 51,060 | 39 | 177 | 50,844 |
| XLS→XLSX | 113,389 | 28 | 192 | 113,169 |
| PPTX→PPTX | 1,603 | 1 | 2 | 1,600 |
| PPT→PPTX | 1,855 | 18 | 17 | 1,820 |
| ODT→DOCX | 28,256 | 327 | 52 | 27,877 |
| ODS→XLSX | 10,054 | 37 | 664 | 9,353 |
| ODP→PPTX | 4,013 | 1 | 1 | 4,011 |

**⚠️ This measures "does MS Office open the converted file without failure" — OOXML *integrity*, NOT visual or
formatting fidelity.** Do not read it as a fidelity score. Vendor claim in the
[26.04 announcement](https://www.collaboraonline.com/blog/cool-26-04-release/): working toward *"zero"* failures
across ~243,000 documents converted to OOXML, and *"opened, rendered, saved, and then returned to a Microsoft
Office work environment without loss in data or functionality"*, while pushing back on the ODF criticism:
*"Just because we're great at ODF doesn't stop us being great at interop too!"*

**Verified caveats:**
- **Fixed fidelity defects that enumerate the real failure classes** (25.04 release notes): *"multiple DOCX
  roundtrip issues involving date pickers, content controls, bookmarks, and **document corruption reported by
  Microsoft Word**"* (tdf #169101, #168988, #170686); *"PPTX files importing font size as 10 pt instead of
  18 pt"* (tdf #163741); *"PPTX master slide backgrounds were not preserved correctly"*; *"Chartex charts were
  lost when importing from OOXML and re-exporting"* (tdf #165742); *"PPTX with failed embedded fonts crashed on
  save"* (tdf #167214); *"heading style changes to default (FILESAVE DOCX tdf #167082)"*; *"Floating table
  rendered at the top of the next page"*; *"SLOW loading of very large documents with many images"* (tdf #170595).
  ([25.04 release notes](https://www.collaboraonline.com/collabora-online-25-04-release-notes/))
- **The vendor treats accidental OOXML layout rewriting as a real risk**: 25.04.9.3 added
  *"View Mode: Set as the default mode to **prevent broken layout autosaves (mainly affecting OOXML documents)**."*
- **26.04.3.3 current-state** improvements: *"**DOCX export** now reports progress in the browser… saved state is
  shown for **non-ODF formats**"*; interoperability fixes for DOCX table layout next to anchored frames,
  auto-width frames, VML textbox wrapping, list bullets, custom XML via XPath, zero default tab stop distance;
  *"Setting a password on binary MS Office documents is no longer offered, as it cannot be done securely."*
- **Live residual divergence:** a May 2026 forum thread, *"There is a significant difference in how the same DOCX
  document is rendered in Collabora Online versus Microsoft Office"* (text placement relative to an image), with
  a core issue filed and an engineer suggesting Word 2007-vs-2013+ wrapping behaviour may be the cause.
  ([forum](https://forum.collaboraonline.com/t/there-is-a-significant-difference-in-how-the-same-docx-document-is-rendered-in-collabora-online-versus-microsoft-office/4664))
- **Macros: disabled by default since 22.04.** Collabora's own
  [macro article](https://www.collaboraonline.com/blog/how-to-use-and-manage-macros-in-collabora-online/)
  (last modified **2026-07-02**, so current): macros can run in Online since CODE/COOL 6.4.7, BUT *"Due to the
  security disclosure **CVE-2025-24796** that highlighted the ability to run malicious code remotely from a
  macro, the **default setting for macros has been disabled from 22.04 onwards**."* Config in
  `/etc/coolwsd/coolwsd.xml`: `<enable_macros_execution>` (default **false**) and `<macro_security_level>`
  (0 = Low/not recommended; **1 = Medium/default** = confirmation required for macros from untrusted sources).
  *"Be aware the interface to run macros… is only available in some integrations."*
  **"Macro editing is not possible online and needs to be done in the desktop application."**
  Vendor's list of macro limitations in Online (**"not exhaustive"**): cannot access database sources;
  XForms/Forms/Controls/button clicks; access other or external documents; create a document from a template;
  Mail Merge; call an external program; use the Shell command; extract a Zip file; get/set the current
  directory; connect to a remote OOO server via Basic; create a toolbar for a component type; toggle design mode
  or access toolbars.
  On **preservation** (desktop engine semantics): Collabora Office help says *"Change only the normal contents
  (text, cells, graphics), and do not edit the macros… Open the file in Microsoft Office, and **the VBA macros
  will run as before**"* — i.e. pass-through is deliberate, but *"**Collabora Office cannot run Visual Basic
  Scripts**"* (some Excel VBA can run if enabled at Tools ▸ Options ▸ Load/Save ▸ VBA Properties).
  **CVE-2025-24796 CVSS/affected-version detail: NOT verified** (NVD and the GitHub advisory were unfetchable);
  only Collabora's own description was available.
- **Fonts: metric-compatible substitutes ship by default.** Default CODE font set, quoted verbatim from the
  official SDK "Fonts" page via a forum post (**2022 source — `[STALE-RISK]`**, live page behind Anubis):
  `Caladea` and `Carlito` (metric-compatible with `Cambria` and `Calibri`); `Déjà Vu`; `Emoji One`; `Gentium`;
  `Google Open Sans` and `PT Serif`; `Google Noto` (full Unicode); `Karla`; `Liberation Sans` and
  `Liberation Serif` (metric-compatible with `Arial` and `Times New Roman`); `Linux Libertine G`;
  `Source Code Pro` and `Source Sans Pro`.
  ([forum quote](https://forum.collaboraonline.com/t/whats-the-problem-that-we-cant-see-the-letters-in-pptx/812/10))
  → Arial/Times/Courier/Calibri/Cambria get metric-compatible clones; **genuine MS fonts are absent and cannot
  legally be redistributed**. Admins can install fonts on the server, but must **update the systemplate** for
  them to be picked up. **Silent substitution remains a live UX problem**: an open issue
  ([#6351](https://github.com/CollaboraOnline/online/issues/6351)) requests a missing-font notification for
  Segoe/Segoe UI/Segoe Light, which are silently substituted and render differently from LibreOffice desktop
  ([forum thread](https://forum.collaboraonline.com/t/replacement-fonts-for-segoe-light-and-segoe-ui/3787)).
  26.04 improved font pickup at service startup and added "Filter Fonts by Typing".
- **PPTX (Impress) caveats:** the vendor's PowerPoint conversion-challenge list above (AutoShapes; tab/line/
  paragraph spacing; **master background graphics**; grouped objects; multimedia effects) sits inside a document
  that otherwise says converted files *"require some degree of manual reformatting."* Vendor-fixed PPTX defects
  show the failure classes: master slide backgrounds not preserved; font size imported as 10 pt vs 18 pt
  (tdf #163741); PPTX→PPTX export of internal names in custom shape geometry (tdf #170035); soft-edge image
  rendering (tdf #170095); font size too big in a shrink-on-overflow textbox (tdf #165712); crash on save with
  failed embedded fonts (tdf #167214); Chartex charts lost on re-export (tdf #165742); 26.04.3.3 shape colours
  turning green on PPTX→ODF→PPTX. LibreOffice/Impress Bugzilla items (**titles only — pages returned 403, so
  dates/status UNVERIFIED**): *"FILEOPEN PPTX: equation not displayed because **Impress doesn't support inline
  formulas**"* ([#129061](https://bugzilla.documentfoundation.org/show_bug.cgi?id=129061)), text box expands/
  shrinks on both sides ([#162571](https://bugs.documentfoundation.org/show_bug.cgi?id=162571)), paragraphs in
  shapes get 0.6 cm before-indent on FILESAVE PPTX ([#169524](https://bugs.documentfoundation.org/show_bug.cgi?id=169524)),
  rounded rectangle wrongly exported to ODF ([#173005](https://bugs.documentfoundation.org/show_bug.cgi?id=173005)).
  Third-party verdict: Collabora `.pptx` **"Acceptable — animations may differ"**
  ([selfhosting.sh](https://selfhosting.sh/compare/collabora-vs-onlyoffice/)).
- **Large files / performance:** server-side rendering means per-keystroke server work (see the sizing
  disagreement in §2.5). Vendor-recognised issues and fixes: slow loading of very large image-heavy documents
  (tdf #170595), slow first tiles in Writer for large documents (cool#12184), improved stability on large/complex
  documents, and in 26.04 *"Faster and much leaner XLSX export… one test document went from 8.6 to 2.0 seconds
  and from 1.6 GB to 415 MB"*, plus asynchronous save so *"saving no longer blocks the editor."*
  **No official numeric max-document-size limit could be retrieved** (SDK manual behind Anubis).

---

## 3. What I could NOT verify (explicit list)

1. **Docker image sizes** for `onlyoffice/documentserver`, `-ee`, `-de`, and `collabora/code`/COOL.
   `hub.docker.com` and its JSON API were unreachable from this sandbox.
2. **ONLYOFFICE per-connection unit pricing**, tier breakpoints, volume discounts, and the full commercial
   licence text (login-gated). Only "From $1500" (Enterprise) and configurator totals ($2100 / $3500) were read.
3. **Collabora's official SDK docs and its official sizing guidance.** `sdk.collaboraonline.com` (including
   `CO-SDK-manual.pdf`) is behind **Anubis proof-of-work** and returned a challenge page every attempt.
   WOPI/postMessage/tutorial URLs are pointers only — **not read**. `help.collaboraonline.com` does not resolve.
4. **Collabora's exact COOL (commercial) Docker image name**, and whether CODE vs COOL images differ beyond
   branding/support.
5. **Collabora's *current* official Fonts page** — the quoted default font list comes from a **2022** forum post
   citing it `[STALE-RISK]`.
6. **CVE-2025-24796 and CVE-2026-77276 detail** — CVSS, affected/fixed versions. NVD and the GitHub advisory
   pages were unfetchable; only titles/vendor descriptions available.
7. **LibreOffice/Impress Bugzilla details** (403) — the Impress "inline formulas unsupported" claim rests on the
   bug **title** alone; dates/status unconfirmed.
8. **ONLYOFFICE's helpcenter "Supported formats" tick-marks** — the per-format "edit natively" vs "edit after
   conversion" ticks are images and did not survive text extraction; prose statements were used instead.
9. **Any ONLYOFFICE guarantee that VBA inside `.docm`/`.xlsm` is preserved** through edit+save — no official
   statement found. Only bug #3466 (fixed) as evidence.
10. **Collabora price fine print** — VAT, minimum seats, minimum term.
11. **Official numeric max-document-size limits** for either product.
12. **Any independent hands-on fidelity benchmark** (formal visual diffing) for either product. Vendor
    marketing ("100% fidelity") and the mso-test integrity numbers are **not** visual-fidelity proof. I did not
    test either product.
13. **Truly current status of some vendor pages**: the ONLYOFFICE Enterprise FAQ and `compare-editions` still
    assert the **20-connection** Community limit, which the **9.4 changelog and release blog contradict**.
    `[STALE-RISK]` on those two pages.

---

## 4. Practical guidance for this specific product

1. **Budget a backend either way.** Both need a real server: file storage, auth, and a save round-trip
   (ONLYOFFICE `callbackUrl`) or a **WOPI host** (Collabora). A "preview + edit in a right-hand panel" is an
   iframe pointing at a **separate origin**, plus CSP / `X-Frame-Options` / cookie-`SameSite` work — not a React
   component. Plan a second deployable (and, for Collabora, a second hostname with matching TLS).
2. **Licence decision tree:**
   - **Zero source-release obligation on your app → Collabora**, built from MPL-2.0 source (or buy COOL at
     ~€3/user/month). MPL is file-level; your React app stays proprietary.
   - **Maximum OOXML fidelity and least integration work → ONLYOFFICE Docs Developer** (commercial). Its own
     page describes it as *"for a software developer seeking powerful document-editing capabilities to extend
     your service functionality and provide it to customers **under your brand**"* — your exact scenario, with
     white-label, any connection count, mobile web editors and Admin Panel.
   - **Do not** ship ONLYOFFICE **Community** in a closed-source product and white-label it. The FAQ's "No" on
     branding removal plus the AGPL network clause make this the one clearly wrong combination.
3. **Normalize uploads to OOXML (`.docx/.xlsx/.pptx`) before opening in either editor.** This sidesteps
   ONLYOFFICE's `.doc`-can't-save-as-`.doc` rollback *and* avoids Collabora's ODF-side conversion surface. It is
   the single highest-leverage fidelity hardening step for both products.
4. **Set macro and font expectations with stakeholders now.** Neither runs MS **VBA**; ONLYOFFICE macros are
   JavaScript-only and sandboxed, and Collabora's are **off by default with no online macro editing**. Neither
   ships Microsoft fonts — layout will shift. If your users' files depend on Calibri/Cambria/Arial, decide early
   whether you (a) accept metric-compatible substitution, (b) license and install the real fonts (ONLYOFFICE's
   Admin Panel font management + OS fonts makes this least painful), or (c) reject macro-enabled files at upload.
5. **Treat `.pptx` as the highest-risk format in both products**, and device-test a corpus of *your users' real
   decks*. ONLYOFFICE has several **still-open** PPTX bugs (background fills/theme colours, complex-script theme
   fonts, equations lost on export, comments not shown, table sizing); Collabora's own docs put **master
   background graphics** and **AutoShapes** on the must-reformat list and Impress lacks inline formulas.
6. **Choose the rendering model deliberately.** ONLYOFFICE renders **client-side** → cheap server, but big
   documents can OOM the browser (~25 MB complex `.docx` reported). Collabora streams **server-side tiles** →
   cheap client, but you buy CPU/RAM per concurrent editor (and the two sizing estimates disagree ~3×; capacity-test
   it). This is a real cost/scaling decision, not a footnote.
7. **Security:** both products render untrusted OOXML/ODF server-side and both have a live CVE history (macro
   sandbox/RCE, XLS/PDF parser OOB reads, XSS). If you host user uploads: isolate the editing server (separate
   host/network; note ONLYOFFICE's `ALLOW_PRIVATE_IP_ADDRESS=false` default and Collabora's chroot+seccomp
   design), keep it patched, and prefer a **supported release over rolling CODE** — CODE's monthly rolling
   cadence makes patch management harder.
