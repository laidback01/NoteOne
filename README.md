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
    .nojekyll            tells GitHub Pages to serve the files as they are
    vendor/              marked 15.0.12 (markdown), DOMPurify 3.2.6 (sanitiser)
    fonts/, fonts.css    Public Sans, Source Serif 4, IBM Plex Mono (SIL OFL)

No build step. Serve the folder as static files.

## Try it

    python3 -m http.server 8000

Open <http://localhost:8000>. It starts with **Just try it**: sample notes held
in this browser, same code paths as the real thing. Settings offers three
separate choices: **GitHub**, **Forgejo**, or **Just try it**. Each one uses its
own wording and its own token instructions.

To connect, paste the notes repository's link exactly as GitHub or Forgejo
gives it: the browser address, the HTTPS clone URL, or the SSH `git@…`
address. The page works out the server and repository from it (a github.com
link picks GitHub by itself; any other server is Forgejo, or GitHub
Enterprise if you chose GitHub) and uses the repository's default branch
unless *More options* names another.

## Permissions

The app has no users and no access rules of its own. **The repository's
permissions are the permission model**, on GitHub and on Forgejo alike:

| Access to the notes repo | In NoteOne |
|---|---|
| write | read, edit, create subjects and notes, attach images |
| read | read and search; saves fail |
| none | nothing |

For an **owner and a contributor**:

- **Owner** owns the notes repo.
- **Contributor** is added as a collaborator with **Write**, not Admin.
  Write covers everything NoteOne does. Admin would also let them change
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

    { "service": "github",  "repo": "https://github.com/you/notes" }
    { "service": "forgejo", "repo": "git@forgejo.example.net:team/notes.git" }

`service` is required and must be `github` or `forgejo`. `repo` is the notes
repository's link, copied from its page or its clone button (HTTPS or SSH),
or just `owner/name`. `branch` is optional; without it, the repository's own
default branch is used. For Forgejo behind a same-origin proxy (see Hosting),
use plain `owner/name` so no server is named.

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

### GitHub Pages — the easy way

The app and the notes live in **two repos**: this one (public, the app) and a
**private** one that holds the notes. The page only shows the repo you point
it at; it never shows its own files.

1. **Turn on Pages for this repo.** On GitHub: the repo → **Settings** →
   **Pages**. Under *Build and deployment*, set *Source* to **Deploy from a
   branch**, *Branch* to **main** and the folder to **/ (root)**, then
   **Save**. After a minute or two the address appears at the top of that
   page: `https://<your-username>.github.io/NoteOne/`.
2. **Make the notes repo.** New repository, e.g. `notes`, set to
   **Private**. It can start empty; the first subject you make becomes its
   first commit.
3. **Add people.** In the notes repo → **Settings** → **Collaborators** →
   *Add people*. Collaborators get write access, which is what they need.
4. **Each person makes a token** (see Permissions above), opens the Pages
   address, clicks **Settings**, pastes the notes repo's link (from the
   address bar, or the green **Code** button — HTTPS or SSH), and pastes
   their token. The form says back what it understood, e.g.
   *✓ GitHub · repository you/notes*.

`.nojekyll` in this repo tells Pages to serve the files exactly as they are.
Nothing else needs setting up: `api.github.com` accepts calls from any web
page.

**Pointing it at this repo instead?** It would work, because folders with no
notes (`fonts/`, `vendor/`) are hidden, but this repo is public, so the notes
would be public too. Use a private notes repo.

**Pinning it for your team.** Commit a `config.json` here (see above) and the
Pages site only ever opens your notes repo. The repo's *name* becomes visible
to anyone who visits, but not its notes. Anyone who forks this repo should
delete that file.

**One caution.** Every Pages site under one account shares the address
`<your-username>.github.io`, and with it the browser storage where the token
is kept. Any other Pages site you publish there could read that token. Only
publish pages there you trust, or give NoteOne its own domain (Settings →
Pages → *Custom domain*).

Private repos can only use Pages on a paid plan, which is another reason the
app repo is the public one.

### GitHub, hosted elsewhere

Any static web host works. Serve the files as they are; there is no build
step.

### Forgejo

The browser has to be allowed to call Forgejo from wherever the page is
served. Pick one.

**A. Same-origin proxy (cleanest).** Serve the page and forward `/api/` to
Forgejo, so the browser never makes a cross-site call. Leave the address
blank.

    location / { root /var/www/noteone; }
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
- **GitHub, end to end** against a real private repo with a fine-grained
  token, in a browser: connecting by pasted link, the first save into an
  empty repo, editing, ticking tasks, sections, pasting an image and showing
  it back from the private repo, search, the History link, and sign out. A
  conflict (someone else saved first) is refused by GitHub, warned about,
  and leaves both versions safe. Refused writes from a read-only token are
  explained in plain words.
- **Forgejo**: API shapes checked against the published API spec of a
  Forgejo 13.0.2 server. **No real reads or saves yet.**
- **Pinning**: a pinned config ignores earlier browser settings, only ever
  talks to the pinned repo, and a broken config stops the page.
- **Sign out**: deletes the saved token, clears notes from memory, and
  either falls back to the demo or reopens the sign-in box (pinned).

## Known limits (it is a prototype)

- **Tokens are pasted and stay in the browser's localStorage** until someone
  clicks **Sign out**, which deletes the token and clears every note from the
  page. Notes themselves are never saved on the computer. On a shared PC,
  sign out when you're done. The next step is OAuth sign-in (GitHub OAuth
  app, or Forgejo OAuth2 with PKCE).
- No rename or move yet.
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
