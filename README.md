# NoteOne — shared notes on a git repo

An early prototype of a OneNote replacement that is just a git repository
with a front page. The repository can be on **GitHub** or on **Forgejo**.

- a **subject** is a top-level folder, and a tile on the front page
- a **note** is a `.md` (or `.txt`) file in that folder
- a **section** is a subfolder: `Meetings/2026-10/Ops sync.md`
- **saving is a commit**, made as the person who saved

There is no backend and no database. The page is static HTML; each person's
browser talks to GitHub or Forgejo with that person's own token.

## Files

    index.html           the whole app — edit this
    config.example.json  copy to config.json to pin the repo (see below)
    vendor/              marked 15.0.12 (markdown), DOMPurify 3.2.6 (sanitiser)
    fonts/, fonts.css    Public Sans, Source Serif 4, IBM Plex Mono (SIL OFL)

No build step. Serve the folder as static files.

## Try it

    python3 -m http.server 8000

Open <http://localhost:8000>. It starts with **Just try it**: sample notes held
in this browser, same code paths as the real thing. Settings offers three
separate choices: **GitHub**, **Forgejo**, or **Just try it**. Each one uses its
own wording and its own token instructions.

## Permissions

The app has no users and no access rules of its own. **The repository's
permissions are the permission model**, on GitHub and on Forgejo alike:

| Access to the notes repo | In the notebook |
|---|---|
| write | read, edit, create subjects and notes, attach images |
| read | read and search; saves fail |
| none | nothing |

For an **owner and a contributor**:

- **Owner** owns the notes repo.
- **Contributor** is added as a collaborator with **Write**, not Admin.
  Write covers everything the notebook does. Admin would also let them change
  collaborators, webhooks and repo settings. (On a GitHub personal repo,
  collaborators get write and can't be made admin, which is what you want.)

Each person makes their own token and pastes it into the page:

- **GitHub**: a fine-grained token (Settings → Developer settings → Personal
  access tokens) limited to the notes repo, with **Contents: Read and write**.
  If the repo is under someone else's personal account and doesn't show up in
  the list, use a classic token with the **repo** scope instead. Moving the repo
  into a GitHub organisation is the tidier fix.
- **Forgejo**: Settings → Applications, with **repository: Read and write**.

## Pinning the repo (`config.json`)

Without a config, anyone can switch the page between GitHub, Forgejo and
demo mode. For a team deployment, put a `config.json` next to `index.html`:

    { "service": "github",  "repo": "owner/notes", "branch": "main" }
    { "service": "forgejo", "base": "https://forgejo.example.net", "repo": "owner/notes", "branch": "main" }

`service` is required and must be `github` or `forgejo`. `base` is optional:
for GitHub it is a GitHub Enterprise address (leave it out for github.com);
for Forgejo, leave it out when the page reaches Forgejo through a proxy (see
Hosting).

With a config, the page always uses that repo. The button says "Sign in" and
only asks for the person's token; the service, address, repo and branch can't
be changed, and anything saved in the browser earlier is ignored. A
`config.json` that is broken, or that has no `service`, stops the page with an
error. It does not fall back to letting people choose.

**What this does and doesn't protect.** It's a deployment setting, not a
security boundary. Settings live in each person's own browser, so changing
them only ever affected that browser, never anyone else's. Someone who wants
to could still open a different copy of the page. What actually protects the
notes is the repository's permissions, so:

- **Keep the app and the notes in separate repos.** If the page were served out
  of the notes repo itself, anyone with write access could edit `config.json`.
- Give contributors no write access to the app repo.

## Hosting

### GitHub

Nothing to set up: `api.github.com` accepts calls from any web page. Host the
app anywhere, including **GitHub Pages** from the app repo. The page itself
can be public: it holds no notes and no secrets, and the notes repo stays
private.

### Forgejo

The browser has to be allowed to call Forgejo from wherever the page is
served. Pick one.

**A. Same-origin proxy (cleanest).** Serve the page and forward `/api/` to
Forgejo, so the browser never makes a cross-site call. Leave the address
blank.

    location / { root /var/www/notebook; }
    location /api/ {
        proxy_pass https://forgejo.example.net;
        proxy_set_header Host forgejo.example.net;
        proxy_ssl_server_name on;
    }

The "History" link is hidden in this mode, because the proxy only forwards
the API.

**B. CORS on Forgejo.** In `app.ini`, then restart Forgejo:

    [cors]
    ENABLED = true
    ALLOWED_DOMAINS = https://notes.example.net
    HEADERS = Content-Type,User-Agent,Authorization

`Authorization` is not in the default `HEADERS`, and without it every call
fails before it starts. Put the real Forgejo address in the config.

It cannot run *as* a README on either service: rendered markdown has its
scripts stripped. Forgejo has no built-in Pages.

## How it works

Both services use the same contents API, with three differences that the
code handles:

| | GitHub | Forgejo |
|---|---|---|
| API address | `api.github.com` (Enterprise: `<address>/api/v3`) | `<address>/api/v1` |
| token header | `Bearer <token>` | `token <token>` |
| create a file | `PUT` | `POST` |

- **Listing**: one `GET /git/trees/{branch}?recursive` call gives every path
  and blob sha. Tiles, note lists and title search all come from that.
- **Conflicts**: the sha is an optimistic lock. If someone else saved first,
  the server rejects the update; the editor keeps your text and says so. No
  silent overwrites, but no automatic merge either.
- **No HTTP caching** of API calls. GitHub allows 60 s of caching, which
  would show a note as it was before your own save.
- **New subject**: git has no empty folders, so a subject starts as
  `<Subject>/README.md`, shown as "Overview".
- **Images**: paste or drop into the editor. They go to
  `<Subject>/assets/<timestamp>.png` and a relative link is inserted, so the
  service's own file view renders them too.
- **Links**: `[[Note]]` links to a note in the same folder, `[[/Subject/Note]]`
  from the repo root, `[[Note|label]]` with a label. A link to a note that
  doesn't exist yet shows red; click it to create the note.
- **Task lists**: ticking a checkbox in read mode edits the source and commits.
- **Search**: typing filters tiles and note titles. Enter searches inside
  notes by fetching each one once and caching it by blob sha, so later
  searches only fetch what changed.

## Where SQLite would come in

Not needed to start. Full-text search in the browser is fine into the low
thousands of notes. Past that, a small service that keeps a SQLite FTS5 index
up to date from a push webhook takes over search only. The files stay the
source of truth, and the index can always be rebuilt from the repo.

## Tested so far

- **Just try it** (demo): editing, saving, ticking checkboxes, the conflict
  warning, new subjects and sections, links, search, image paste, script
  sanitising, dark mode. All driven in headless Chrome.
- **GitHub**: listing, reading, raw files and missing-file handling against a
  real public repo, from a browser page. Bad-token wording. **Saving to
  GitHub has not been tried yet**: it needs a token and a test repo.
- **Forgejo**: API shapes checked against the published API spec of a
  Forgejo 13.0.2 server. **No real reads or saves yet.**
- **Pinning**: a pinned config ignores earlier browser settings, only ever
  talks to the pinned repo, and a broken config stops the page.

## Known limits (it is a prototype)

- **Tokens are pasted and stay in the browser's localStorage.** The next step
  is OAuth sign-in (GitHub OAuth app, or Forgejo OAuth2 with PKCE).
- No rename or move yet.
- A brand-new GitHub repo with no commits may refuse the first save. Create
  it with a README.
- GitHub lists at most about 100,000 files in one call; past that the page
  warns that some notes are missing.
- One editor at a time per note. Simultaneous edits get a conflict, not a
  merge.
- Large binaries bloat the repo. If people attach lots of PDFs, use Git LFS.

## License

BSD 2-Clause — see [LICENSE](LICENSE).

Bundled third-party files keep their own licences:

- `vendor/marked.min.js` — MIT (header in the file)
- `vendor/purify.min.js` — Apache 2.0 or MPL 2.0 (header in the file)
- `fonts/` — SIL Open Font License 1.1, one licence file per family
  (`fonts/OFL-*.txt`)
