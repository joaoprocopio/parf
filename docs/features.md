# Parf — Features

> **Parf** — from Noldorin _parf_ (n.) — _book_. A self-hosted, open-source documentation manager with a WYSIWYG editor and Git-backed storage. Built so both technical and non-technical people contribute to a shared knowledge base — because organisations must own their documentation, and as many people as possible should be able to view and contribute to it.

---

## Core principles

- **Projection-agnostic storage** — a operation log can project the document tree into any format capable of representing rich text: Markdown, JSON, AST, etc.
- **Conflict-free sync across devices** — CRDT-based.
- **Optional sync with a central authority** — the server is opt-in, not required.
- **Configurable workspaces** — pre-processors, plugins, post-processors.

---

## Tier 1 — MVP: Usable & Safe

### Platform & deployment

- **Self-hosted** — deploy on your own server; documentation never leaves your infrastructure
- **Git as the database** — for storage and versioning; the file system is the data model (no artificial tree)
- **Multiple repository support** — work across multiple repos (by repo address + filepath), or a single repository fully managed by the application
- **Speed as a feature** — faster than every competitor;
- **Cross-platform clients** — terminal (CLI), IDE, text editor, GUI, and mobile
- **Workspace layout — single-root** (`Parf.toml` + `Parf.lock` at root)
- **Global configuration** — `~/.parf/config.toml` shared by all clients; mandatory **user identity** field (the op-log + identity drive versioning and history)

### Workspaces

- Workspace **creation**
- **Document management**

### WYSIWYG editor

- Floating toolbar on selection — the editing state is available at all times
- **Basic editing actions**: undo/redo, select all, copy, cut, paste rich-text, paste plain-text, strip rich-text (rasterize)
- **Inline formatting**: bold, italic, underline, strikethrough, inline code, inline quote, superscript, subscript, `kbd`
- **Content blocks**:
  - Lists — checklist, numbered, bulleted
  - Tables
  - Block quotes
  - Code blocks
  - General typography (headings, body, etc.)
  - Divider/separator
  - Footnotes — the `[1]` marker linked to a note at the bottom of the page
  - Callouts/alerts
- **Inserts/uploads**: images, videos, GIFs, PDFs, attachments in general — via drag & drop or upload
- **Internal links** between pages and documents
- **External links** to pages outside Parf
- **Keyboard shortcuts**
- **Context menu** on right-click wherever possible
- **Auto save**
- **Interaction modes**: View, Edit, Suggest

### Organization & retrieval

- **Document organization**: folders, nested pages, navigation sidebar
- **Full-text search** across all documents

### Collaboration meta-information

- **Simple comments** on documents
- **Annotations / highlights**

### Accounts, history & permissions

- **User authentication** (email/password login)
- **Edit history** with the identity of the editing user; view & restore previous versions
- **Basic permissions**: Admin, Editor, Viewer
- **Basic version control**: view previous versions and restore

### Export

- **Export** to PDF/Markdown

### Sync engine (MVP scope)

- **Op-log** — Operation log as the source of truth
- **CRDT 3-way merge** — bypass Git's merge mechanism
- **Integrity checking** — verify op-log integrity and emit diagnostics
- **Auto-fix** — fix trivial integrity issues
- **Watch mode** — sync driven by a file-system watcher
- **Sidecar files** — metadata co-located with content (format and block-identification scheme TBD)
- **Cross-platform file handling** — resolve CRLF/LF and UTF-8 encoding differences across operating systems
- **Workspace markers** — `.parf/` directory, `Parf.lock` file, or both (TBD)

---

## Tier 2 — Collaborative & Organized

### Administration

- **Workspace administration**
- **Members & teams management**
- **Advanced permissions**: per-folder/per-page roles, guest access
- **Workspace layout — multi-member** — nested `Parf.toml` per book inside the workspace

### Collaboration

- **Real-time collaboration** (simultaneous editing, Google Docs style)
- **Notifications**: email or in-app on changes/comments
- **Document status**: Draft, In Review, Published
- **Tasks/checklists inside documents** (to guide processes)
- **P2P collaboration** — generate a token for peer-to-peer collaboration; peers join with `join <token>`

### Editor power-ups

- **Drag & drop blocks** to reorganize content
- **Transform blocks** into other block types
- **Quick command menu** (`/`) for inserting blocks and elements
- **Smart completion**: auto-continue lists, etc.
- **Smart paste**: links become embeds (e.g. Jira tickets)
- **Emoji picker**
- **Spelling & grammar checking**
- **Find & replace** (advanced search within a document)
- **On-this-page outline** with heading navigation

### Organization & discovery

- **Tagging & metadata**: label documents by department, system, or priority
- **Search filters**: by tags, authors, date
- **Dashboards**: recent, popular, favorite documents

### Import & theming

- **Import** from Word, Google Docs, Markdown
- **Dark mode / theme customization**

### Version control

- **Advanced version history** with fine-grained control

### Sub-workspaces & sharding

- **Sub-workspaces** — creating a sub-workspace assigns it a new op-log, providing natural sharding without in-system sharding logic

---

## Tier 3 — Enterprise-grade

### Integrations & API

- **Git repository integration** for content sync (including existing-monorepo scenarios)
- **Access API**: create/read/update documents via integration
- **Integrations hub**: Slack, Jira, GitHub, Confluence (import/export)
- **SSO**: Google Workspace, Azure AD, Okta

### Workflows & compliance

- **Advanced approval workflows**: Draft → Review → Publish
- **Detailed audit trails**: full logs for compliance
- **Automatic alerts**: reminders to review old docs

### Performance & reliability

- **Superior performance** vs. competitors — decisive speed advantage
- **Automatic backups & offline access**

### AI & intelligence

- **AI assistance**: document summaries, content generation, Q&A over the knowledge base
- **Analytics & insights**: most-read documents, stale docs, engagement metrics

### Multilingual & export

- **Multilingual support**: maintaining versions in different languages
- **Advanced export**: PDF, Word, and Markdown

---

## CLI (`parf`)

| Command        | Description                                                     |
| -------------- | --------------------------------------------------------------- |
| `new` / `init` | Initialize a workspace                                          |
| `log`          | Browse the op-log in a TUI                                      |
| `sync`         | Bypass Git's merge mechanism; 3-way merge via CRDTs             |
| `sync --watch` | Same, driven by a file-system watcher                           |
| `check`        | Verify op-log integrity, emit diagnostics                       |
| `check --fix`  | Auto-fix the trivial cases                                      |
| `share`        | Generate a token for P2P collaboration (`join <token>` to join) |
| `join`         | Join a peer's workspace using their share token                 |

---

## Exploratory / unscheduled

- **Whiteboard** — flagged as a potential game changer
- **Full P2P sync** without a central authority
- **Additional client surfaces** — TUI, IDE, text-editor, and mobile beyond the primary GUI

---

## Inspirations & competitive landscape

- [gollum](https://github.com/gollum/gollum)
- [GitBook](https://www.gitbook.com/)
- [Obsidian](https://obsidian.md/)
- [home cms](https://github.com/bearcove/home)
- [BookStack](https://www.bookstackapp.com/)
- [Outline](https://www.getoutline.com/)
- [Logseq](https://logseq.com/)
- [Roam Research](https://roamresearch.com/)
- [Scratch](https://www.ericli.io/scratch)

## Technical references

### Libraries

- [TipTap](https://tiptap.dev/) — headless wrapper around ProseMirror
- [ProseMirror](https://prosemirror.net/) — WYSIWYG toolkit (extended take on CodeMirror's ideas)
- [CodeMirror](https://codemirror.net/) — code editor; Obsidian uses it with HyperMD for a "code editor that renders rich text"
- [Prism.js](https://prismjs.com/) — syntax highlighting
- [GitLab's content editor](https://docs.gitlab.com/ee/development/fe_guide/content_editor.html) — uses TipTap and ProseMirror

### Reading list

- [We Built Collaborative Editing for Our Newsroom's CMS. Here's How.](https://open.nytimes.com/we-built-collaborative-editing-for-our-newsrooms-cms-here-s-how-415618a3ec49)
- [The data model behind Notion's flexibility](https://www.notion.com/blog/data-model-behind-notion)
- [Sharding & IDs at Instagram](https://instagram-engineering.com/sharding-ids-at-instagram-1cf5a71e5a5c)
- [How we sped up Notion in the browser with WASM SQLite](https://www.notion.com/blog/how-we-sped-up-notion-in-the-browser-with-wasm-sqlite)
- [Why I rebuilt ProseMirror's renderer in React](https://smoores.dev/post/why_i_rebuilt_prosemirror_view/)

### Key insight

Obsidian uses CodeMirror with HyperMD. CodeMirror's philosophy — a _code editor that renders rich text_ — fits perfectly with the goal of saving human-readable-and-editable Markdown files. TipTap wraps ProseMirror, which extends ideas from CodeMirror.
