# Notebook — shared notes on a git repo

A throwaway prototype of a OneNote replacement that is just a Forgejo
repository with a front page.

- a **subject** is a top-level folder, and a tile on the front page
- a **note** is a `.md` (or `.txt`) file in that folder
- a **section** is a subfolder: `Meetings/2026-10/Ops sync.md`
- **saving is a commit**, made as the person who saved

There is no backend and no database. The page is static HTML; each person's
browser talks to the Forgejo API with that person's own token.

## Files

    index.html        the whole app — edit this
    config.example.json  copy to config.json to pin the repo (see below)
    vendor/           marked 15.0.12 (markdown), DOMPurify 3.2.6 (sanitiser)
    fonts/, fonts.css same fonts as the matter board

No build step. Serve the folder as static files.

## Try it

    python3 -m http.server -d notes 8000

Open <http://localhost:8000>. It starts in **demo** mode: sample notes held in
this browser's localStorage, same code paths as the real thing. Settings
switches to a Forgejo repository.

## Permissions

The app has no users and no access rules of its own. **Forgejo's repo
permissions are the permission model**:

| Forgejo access to the notes repo | In the notebook |
|---|---|
| write | read, edit, create subjects and notes, attach images |
| read | read and search; saves fail with 403 |
| none | nothing |

Put people in an org team with access to the repo. That's the admin job.

Each person makes a token in Forgejo (Settings → Applications) with
`write:repository` (or `read:repository` for readers) and pastes it into
Settings. Commits and history then show who changed what.

## Pinning the repo (`config.json`)

Without a config, anyone can switch the page between demo mode and any
repo. For a team deployment, put a `config.json` next to `index.html`:

    { "base": "", "repo": "owner/notes", "branch": "main" }

Then the page always uses that repo. Settings becomes "Sign in" and only asks
for the person's token. The storage, server, repo and branch fields are hidden,
and anything saved in the browser earlier is ignored. `config.example.json` is
the template. `base` works the same as the Forgejo URL field below.

**What this does and doesn't protect.** It's a deployment setting, not a
security boundary. Settings live in each person's own browser, so changing
them only ever affected that browser, never anyone else's. Someone who wants
to could still open a different copy of the page. What actually protects the
notes is Forgejo:

- **Keep the app and the notes in separate repos.** `config.json` is on the
  web server, which contributors can't reach. If the page were served out of
  the notes repo itself, anyone with write access could edit the config.
- **Owner**: owns the notes repo (or is an org owner), so has admin on it.
- **Contributor**: added as a collaborator with **Write**, not Admin. Write
  covers editing notes. Admin would also let them change collaborators,
  webhooks and repo settings.
- Give contributors no access to the app repo.

## Hosting

The browser has to be allowed to call Forgejo from wherever the page is
served. Pick one.

**A. Same-origin proxy (cleanest).** Serve the page and forward `/api/` to
Forgejo, so there is no CORS at all. Leave "Forgejo URL" blank in Settings.

    location / { root /var/www/notebook; }
    location /api/ {
        proxy_pass https://forgejo.406dc.net;
        proxy_set_header Host forgejo.406dc.net;
        proxy_ssl_server_name on;
    }

The "History" link is hidden in this mode, because the proxy only forwards
the API.

**B. CORS on Forgejo.** In `app.ini`, then restart Forgejo:

    [cors]
    ENABLED = true
    ALLOWED_DOMAINS = https://notes.406dc.net
    HEADERS = Content-Type,User-Agent,Authorization

`Authorization` is not in the default `HEADERS`, and without it every call
fails preflight. Put the real Forgejo URL in Settings.

It cannot run *as* a README or a Forgejo page: rendered markdown has its
scripts stripped, and Forgejo has no built-in Pages. GitHub Pages would host
it fine, but the store would need a small GitHub adapter (see below).

## How it works

- **Listing**: one `GET /git/trees/{branch}?recursive=true` gives every path
  and blob sha. Tiles, note lists and title search all come from that.
- **Read/write**: the contents API. Create is `POST`, update is `PUT` with the
  file's current sha.
- **Conflicts**: the sha is an optimistic lock. If someone else saved first,
  Forgejo rejects the update; the editor keeps your text and says so. No
  silent overwrites, but no automatic merge either.
- **New subject**: git has no empty folders, so a subject starts as
  `<Subject>/README.md`, shown as "Overview".
- **Images**: paste or drop into the editor. They go to
  `<Subject>/assets/<timestamp>.png` and a relative link is inserted, so
  Forgejo's own file view renders them too.
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
up to date from Forgejo's push webhook takes over search only. The files stay
the source of truth, and the index can always be rebuilt from the repo.

## Known limits (it is a prototype)

- **Tokens are pasted and stay in localStorage.** The next step is Forgejo
  OAuth2 with PKCE, so people sign in rather than make tokens.
- No rename or move yet. The API supports it (`from_path` on update).
- Forgejo only. GitHub's contents API is close, with `PUT` for both create
  and update and a different base URL, but it isn't written or tested.
- One editor at a time per note. Simultaneous edits get a conflict, not a
  merge.
- Large binaries bloat the repo. If people attach lots of PDFs, use Git LFS.
