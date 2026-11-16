# Personal threat intelligence pipeline (Go)

A local, read-only collector for public security reporting. It polls IFIN, SANS ISC, the public local timeline at InfoSec Exchange, Ransomware.live's leak-site claim feed, optional X watchlists, and explicitly configured public onion pages or feeds. Findings go into SQLite with deduplication and source-health state. The CLI writes a review inbox and source material for a daily analyst brief. **A source claim is a lead, not proof of an incident.**

## Build and start

Requires Go 1.27 or newer. From this directory on Windows:

```powershell
go build -o bin/intel.exe .
Copy-Item config.example.json config.json
.\bin\intel.exe run --bootstrap
.\bin\intel.exe run
.\bin\intel.exe status
.\bin\intel.exe list --unread
```

On macOS or Linux use `go build -o bin/intel .` and `cp config.example.json config.json`. The first run seeds existing posts without reporting them as new. The Go collector uses the same `data/intel.sqlite3` schema as the former Python collector; existing findings and review flags are preserved. `config.json`, the database, generated reports, build outputs, and `.secrets/` are ignored by Git.

The default configuration intentionally leaves two direct onion collectors disabled until you supply specific public sources. X watchlists remain enabled but report `needs_setup` until you add an X developer token. Other sources continue when X is unavailable.

## Sources and rate limits

| Source | Method | Default cadence and scope |
| --- | --- | --- |
| IFIN | Public Discourse category JSON | Each scheduled run; recent topic pages and replies |
| SANS ISC | Public RSS | Each scheduled run |
| InfoSec Exchange | Public Mastodon local timeline API | At most one page of 40 posts about every 55 minutes, then security-term filtering |
| Ransomware.live | Public RSS of leak-site claims | About every 115 minutes |
| X watchlists | Official recent-search API | Each scheduled run once configured; bounded pagination |
| Public onion RSS and HTML | Tor SOCKS5 proxy | Disabled until configured; at least 175 minutes between successful polls |

The InfoSec Exchange source is deliberately **sampled**: one public API request per hourly run is unlikely to cover every local post. `status` records `partial=true` and the digest says `sampled` when that page fills. This saves requests and makes the coverage limit visible. The collector does not need your InfoSec Exchange login for the public local timeline. If the instance later closes public access, use its documented app-token process rather than sharing a password in chat; this implementation currently does not use an InfoSec Exchange token.

Ransomware.live republishes claims from leak sites. Its entries are **unverified actor claims**, not confirmed breaches. The collector stores their text and source link; an analyst must seek independent evidence before citing a victim as compromised.

### Direct public onion pages

To enable a configured onion source, install and run Tor with a SOCKS5 listener, replace the placeholder URL in `config.json` with a valid v3 `.onion` address, then set `enabled` to `true`. Tor commonly listens at `socks5h://127.0.0.1:9050`; adjust `tor_proxy` to your local port. The `onion_feed` adapter parses RSS/Atom. The `onion_html` adapter reads only the configured listing page and extracts text, timestamps, and same-site links from matching elements (`entry_tag` and optional `entry_class`). It does not follow post links, download files, execute JavaScript, or log in. It refuses a clearnet URL or an off-site redirect for a direct onion source. Validate each site's current address and terms before enabling it.

These adapters are intended for **public announcement or forum listing pages** that you are authorized to read. Do not configure credentialed areas or download links. A generic HTML selector may need adjustment when a site's layout changes. If you have a few specific public pages, add each as its own source and keep the polling interval conservative.

The researcher's [candidate underground source list](darkweb-sources.md) records forum and leak-site names without guessing current addresses or enabling unverified endpoints.

### X and local exports

The two X watchlists cover underground reporting (`@vxunderground`, `@DarkWebInformer`) and grounded research/advisories (`@abuse_ch`, `@TheDFIRReport`, `@MsftSecIntel`, `@CISACyber`). To enable official recent search, put only the bearer token in `.secrets/x_bearer_token` or set `X_BEARER_TOKEN` in the process environment. Do not put credentials in `config.json` or Git. The query uses a bounded page count; a partial scan does not advance the cursor. [X recent-search documentation](https://docs.x.com/x-api/posts/search/introduction).

For Signal groups you participate in, import an authorized JSON Lines export:

```powershell
.\bin\intel.exe import-jsonl path\to\messages.jsonl --source "Signal research group"
```

Each line needs `id`, `text`, and ISO 8601 `published_at`, with optional `title`, `author`, and `url`. Stable IDs allow deduplication. The collector does not access private chats on its own.

## Review and daily briefing

```powershell
.\bin\intel.exe list --unread --limit 30
.\bin\intel.exe show 42
.\bin\intel.exe mark 42 --reviewed
.\bin\intel.exe list --search clickfix --limit 20
.\bin\intel.exe brief-input --hours 24 --limit 100
```

`run` writes `output/latest.md`, `output/latest.json`, and `output/inbox.md`. `brief-input` writes `output/daily-input.json`. The [briefing protocol](briefing-protocol.md) describes how a person or scheduled assistant verifies claims and writes `output/briefs/YYYY-MM-DD.md`; the collector does not produce a narrative brief by itself. Treat all collected text as untrusted, and do not visit payload links or execute attachments.

For hourly collection, schedule `bin/intel.exe run` with this directory as the working directory. The separate daily brief can use `bin/intel.exe brief-input --hours 24`. Existing Codex desktop automations are local settings and are not included in a Git clone. GitHub Actions runs offline tests only; running collection in ephemeral Actions jobs would lose SQLite cursors unless you add persistent storage.

## Tests and GitHub

```powershell
go test ./...
go vet ./...
```

The GitHub workflow runs the offline Go tests on pushes and pull requests. Before pushing, review `git status --short` to confirm no newly added source path contains credentials or research data. To publish this directory as a separate repository, create an empty GitHub repository, then:

```powershell
git remote add origin https://github.com/YOUR-ACCOUNT/YOUR-REPO.git
git push -u origin main
```
