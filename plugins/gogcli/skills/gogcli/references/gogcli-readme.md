# gogcli README (reference copy)

> Source: https://github.com/steipete/gogcli
> This is a copy of the upstream README for offline reference.
> Last synced: 2026-02-13

---

# gogcli -- Google in your terminal.

Fast, script-friendly CLI for Gmail, Calendar, Chat, Classroom, Drive, Docs, Slides, Sheets, Contacts, Tasks, People, Groups (Workspace), and Keep (Workspace-only). JSON-first output, multiple accounts, and least-privilege auth built in.

## Features

- **Gmail** - search threads and messages, send emails, view attachments, manage labels/drafts/filters/delegation/vacation settings, history, and watch (Pub/Sub push)
- **Email tracking** - track opens for `gog gmail send --track` with a small Cloudflare Worker backend
- **Calendar** - list/create/update events, detect conflicts, manage invitations, check free/busy status, team calendars, propose new times, focus/OOO/working-location events, recurrence + reminders
- **Classroom** - manage courses, roster, coursework/materials, submissions, announcements, topics, invitations, guardians, profiles
- **Chat** - list/find/create spaces, list messages/threads (filter by thread/unread), send messages and DMs (Workspace-only)
- **Drive** - list/search/upload/download files, manage permissions/comments, organize folders, list shared drives
- **Contacts** - search/create/update contacts, access Workspace directory/other contacts
- **Tasks** - manage tasklists and tasks: get/create/add/update/done/undo/delete/clear, repeat schedules
- **Sheets** - read/write/update spreadsheets, format cells, create new sheets (and export via Drive)
- **Docs/Slides** - export to PDF/DOCX/PPTX via Drive (plus create/copy, docs-to-text)
- **People** - access profile information
- **Keep (Workspace only)** - list/get/search notes and download attachments (service account + domain-wide delegation)
- **Groups** - list groups you belong to, view group members (Google Workspace)
- **Local time** - quick local/UTC time display for scripts and agents
- **Multiple accounts** - manage multiple Google accounts simultaneously (with aliases)
- **Command allowlist** - restrict top-level commands for sandboxed/agent runs
- **Secure credential storage** using OS keyring or encrypted on-disk keyring (configurable)
- **Auto-refreshing tokens** - authenticate once, use indefinitely
- **Least-privilege auth** - `--readonly` and `--drive-scope` to request fewer scopes
- **Workspace service accounts** - domain-wide delegation auth (preferred when configured)
- **Parseable output** - JSON mode for scripting and automation (Calendar adds day-of-week fields)

## Installation

### Homebrew

```bash
brew install steipete/tap/gogcli
```

### Build from Source

```bash
git clone https://github.com/steipete/gogcli.git
cd gogcli
make
```

Run:

```bash
./bin/gog --help
```

Help:

- `gog --help` shows top-level command groups.
- Drill down with `gog <group> --help` (and deeper subcommands).
- For the full expanded command list: `GOG_HELP=full gog --help`.

## Quick Start

### 1. Get OAuth2 Credentials

Before adding an account, create OAuth2 credentials from Google Cloud Console:

1. Open the Google Cloud Console credentials page: https://console.cloud.google.com/apis/credentials
1. Create a project: https://console.cloud.google.com/projectcreate
2. Enable the APIs you need:
   - Gmail API: https://console.cloud.google.com/apis/api/gmail.googleapis.com
   - Google Calendar API: https://console.cloud.google.com/apis/api/calendar-json.googleapis.com
   - Google Chat API: https://console.cloud.google.com/apis/api/chat.googleapis.com
   - Google Drive API: https://console.cloud.google.com/apis/api/drive.googleapis.com
   - Google Classroom API: https://console.cloud.google.com/apis/api/classroom.googleapis.com
   - People API (Contacts): https://console.cloud.google.com/apis/api/people.googleapis.com
   - Google Tasks API: https://console.cloud.google.com/apis/api/tasks.googleapis.com
   - Google Sheets API: https://console.cloud.google.com/apis/api/sheets.googleapis.com
   - Cloud Identity API (Groups): https://console.cloud.google.com/apis/api/cloudidentity.googleapis.com
3. Configure OAuth consent screen: https://console.cloud.google.com/auth/branding
4. If your app is in "Testing", add test users: https://console.cloud.google.com/auth/audience
5. Create OAuth client:
   - Go to https://console.cloud.google.com/auth/clients
   - Click "Create Client"
   - Application type: "Desktop app"
   - Download the JSON file (usually named `client_secret_....apps.googleusercontent.com.json`)

### 2. Store Credentials

```bash
gog auth credentials ~/Downloads/client_secret_....json
```

For multiple OAuth clients/projects:

```bash
gog --client work auth credentials ~/Downloads/work-client.json
gog auth credentials list
```

### 3. Authorize Your Account

```bash
gog auth add you@gmail.com
```

This will open a browser window for OAuth authorization. The refresh token is stored securely in your system keychain.

Headless / remote server flows (no browser on the server):

Manual interactive flow (recommended):

```bash
gog auth add you@gmail.com --services user --manual
```

- The CLI prints an auth URL. Open it in a local browser.
- After approval, copy the full localhost redirect URL from the browser address bar.
- Paste that URL back into the terminal when prompted.

Split remote flow (`--remote`, useful for two-step/scripted handoff):

```bash
# Step 1: print auth URL (open it locally in a browser)
gog auth add you@gmail.com --services user --remote --step 1

# Step 2: paste the full redirect URL from your browser address bar
gog auth add you@gmail.com --services user --remote --step 2 --auth-url 'http://localhost:1/?code=...&state=...'
```

### 4. Test Authentication

```bash
export GOG_ACCOUNT=you@gmail.com
gog gmail labels list
```

## Authentication & Secrets

### Accounts and tokens

`gog` stores your OAuth refresh tokens in a "keyring" backend. Default is `auto` (best available backend for your OS/environment).

Before you can run `gog auth add`, you must store OAuth client credentials once via `gog auth credentials <credentials.json>` (download a Desktop app OAuth client JSON from the Cloud Console). For multiple clients, use `gog --client <name> auth credentials ...`; tokens are isolated per client.

List accounts:

```bash
gog auth list
```

Verify tokens are usable (helps spot revoked/expired tokens):

```bash
gog auth list --check
```

Show current auth state/services for the active account:

```bash
gog auth status
```

### Multiple OAuth clients

Use `--client` (or `GOG_CLIENT`) to select a named OAuth client:

```bash
gog --client work auth credentials ~/Downloads/work.json
gog --client work auth add you@company.com
```

### Keyring backend: Keychain vs encrypted file

Backends:

- `auto` (default): picks the best backend for the platform.
- `keychain`: macOS Keychain (recommended on macOS; avoids password management).
- `file`: encrypted on-disk keyring (requires a password).

Set backend via command (writes `keyring_backend` into `config.json`):

```bash
gog auth keyring file
gog auth keyring keychain
gog auth keyring auto
```

Non-interactive runs (CI/ssh): file backend requires `GOG_KEYRING_PASSWORD`.

```bash
export GOG_KEYRING_PASSWORD='...'
gog --no-input auth status
```

Force backend via env (overrides config):

```bash
export GOG_KEYRING_BACKEND=file
```

### Environment Variables

- `GOG_ACCOUNT` - Default account email or alias to use
- `GOG_CLIENT` - OAuth client name (selects stored credentials + token bucket)
- `GOG_JSON` - Default JSON output
- `GOG_PLAIN` - Default plain output
- `GOG_COLOR` - Color mode: `auto` (default), `always`, or `never`
- `GOG_TIMEZONE` - Default output timezone for Calendar/Gmail (IANA name, `UTC`, or `local`)
- `GOG_ENABLE_COMMANDS` - Comma-separated allowlist of top-level commands (e.g., `calendar,tasks`)

### Config File (JSON5)

Typical paths:

- macOS: `~/Library/Application Support/gogcli/config.json`
- Linux: `~/.config/gogcli/config.json` (or `$XDG_CONFIG_HOME/gogcli/config.json`)

Example (JSON5 supports comments and trailing commas):

```json5
{
  keyring_backend: "file",
  default_timezone: "UTC",
  account_aliases: {
    work: "work@company.com",
    personal: "me@gmail.com",
  },
  account_clients: {
    "work@company.com": "work",
  },
  client_domains: {
    "example.com": "work",
  },
}
```

## Commands

Flag aliases:
- `--out` also accepts `--output`.
- `--out-dir` also accepts `--output-dir` (Gmail thread attachment downloads).

### Authentication

```bash
gog auth credentials <path>           # Store OAuth client credentials
gog auth credentials list             # List stored OAuth client credentials
gog --client work auth credentials <path>  # Store named OAuth client credentials
gog auth add <email>                  # Authorize and store refresh token
gog auth service-account set <email> --key <path>  # Configure service account (Workspace)
gog auth service-account status <email>            # Show service account status
gog auth service-account unset <email>             # Remove service account
gog auth keyring [backend]            # Show/set keyring backend (auto|keychain|file)
gog auth status                       # Show current auth state/services
gog auth services                     # List available services and OAuth scopes
gog auth list                         # List stored accounts
gog auth list --check                 # Validate stored refresh tokens
gog auth remove <email>               # Remove a stored refresh token
gog auth tokens                       # Manage stored refresh tokens
gog auth tokens export <email> --out <file>  # Export tokens for CI
gog auth tokens import <file>         # Import tokens (CI setup)
```

### Gmail

```bash
# Search and read
gog gmail search 'newer_than:7d' --max 10
gog gmail messages search 'newer_than:7d' --max 10 --include-body --json
gog gmail thread get <threadId>
gog gmail thread get <threadId> --download --out-dir ./attachments
gog gmail get <messageId>
gog gmail get <messageId> --format metadata
gog gmail attachment <messageId> <attachmentId> --out ./file.bin
gog gmail url <threadId>
gog gmail thread modify <threadId> --add STARRED --remove INBOX

# Send and compose
gog gmail send --to a@b.com --subject "Hi" --body "Plain text"
gog gmail send --to a@b.com --subject "Hi" --body-file ./message.txt
gog gmail send --to a@b.com --subject "Hi" --body-file -  # stdin
gog gmail send --to a@b.com --subject "Hi" --body "fallback" --body-html "<p>Rich</p>"
gog gmail send --to a@b.com --subject "Hi" --body "fallback" --body-html "<p>Rich</p>" --reply-to pa@mungall.dev

# Drafts
gog gmail drafts list
gog gmail drafts create --to a@b.com --subject "Draft" --body "Body"
gog gmail drafts update <draftId> --subject "Updated"
gog gmail drafts send <draftId>

# Labels
gog gmail labels list
gog gmail labels get INBOX --json
gog gmail labels create "My Label"
gog gmail labels modify <threadId> --add STARRED --remove INBOX

# Batch operations
gog gmail batch delete <messageId> <messageId>
gog gmail batch modify <messageId> <messageId> --add STARRED --remove INBOX

# Filters
gog gmail filters list
gog gmail filters create --from 'noreply@example.com' --add-label 'Notifications'
gog gmail filters delete <filterId>

# Settings
gog gmail autoforward get
gog gmail autoforward enable --email forward@example.com
gog gmail autoforward disable
gog gmail forwarding list
gog gmail forwarding add --email forward@example.com
gog gmail sendas list
gog gmail sendas create --email alias@example.com
gog gmail vacation get
gog gmail vacation enable --subject "Out of office" --message "..."
gog gmail vacation disable

# Delegation (Workspace)
gog gmail delegates list
gog gmail delegates add --email delegate@example.com
gog gmail delegates remove --email delegate@example.com

# Watch (Pub/Sub push)
gog gmail watch start --topic projects/<p>/topics/<t> --label INBOX
gog gmail history --since <historyId>
```

### Email Tracking

```bash
gog gmail track setup --worker-url https://gog-email-tracker.<acct>.workers.dev
gog gmail send --to recipient@example.com --subject "Hello" --body-html "<p>Hi!</p>" --track
gog gmail track opens <tracking_id>
gog gmail track opens --to recipient@example.com
gog gmail track status
```

### Calendar

```bash
# List calendars
gog calendar calendars
gog calendar acl <calendarId>
gog calendar colors

# Events
gog calendar events <calendarId> --today
gog calendar events <calendarId> --tomorrow
gog calendar events <calendarId> --week
gog calendar events <calendarId> --days 3
gog calendar events <calendarId> --from today --to friday
gog calendar events <calendarId> --from today --to "+7 days" --json
gog calendar events --all  # all calendars

# Single event
gog calendar event <calendarId> <eventId>
gog calendar get <calendarId> <eventId> --json

# Search
gog calendar search "meeting" --today
gog calendar search "meeting" --days 365 --max 50

# Create
gog calendar create <calendarId> \
  --summary "Meeting" \
  --from 2026-01-15T10:00:00Z \
  --to 2026-01-15T11:00:00Z \
  --attendees "alice@example.com,bob@example.com" \
  --location "Zoom"

# Update
gog calendar update <calendarId> <eventId> \
  --summary "Updated Meeting" \
  --from 2026-01-15T11:00:00Z \
  --to 2026-01-15T12:00:00Z
gog calendar update <calendarId> <eventId> \
  --add-attendee "alice@example.com,bob@example.com"

# Delete
gog calendar delete <calendarId> <eventId>

# RSVP
gog calendar respond <calendarId> <eventId> --status accepted
gog calendar respond <calendarId> <eventId> --status declined
gog calendar respond <calendarId> <eventId> --status tentative

# Availability
gog calendar freebusy --calendars "primary,work@example.com" \
  --from 2026-01-15T00:00:00Z --to 2026-01-16T00:00:00Z
gog calendar conflicts --calendars "primary" --today

# Special event types
gog calendar create primary --event-type focus-time --from ... --to ...
gog calendar create primary --event-type out-of-office --from ... --to ... --all-day
gog calendar focus-time --from ... --to ...
gog calendar out-of-office --from ... --to ... --all-day

# Recurrence + reminders
gog calendar create <calendarId> \
  --summary "Payment" \
  --from 2026-02-11T09:00:00Z --to 2026-02-11T09:15:00Z \
  --rrule "RRULE:FREQ=MONTHLY;BYMONTHDAY=11" \
  --reminder "email:3d" --reminder "popup:30m"

# Team calendars (Workspace)
gog calendar team <group-email> --today
gog calendar team <group-email> --freebusy
```

### Drive

```bash
gog drive ls --max 20
gog drive ls --parent <folderId> --max 20
gog drive search "invoice" --max 20
gog drive get <fileId>
gog drive url <fileId>
gog drive copy <fileId> "Copy Name"

gog drive upload ./path/to/file --parent <folderId>
gog drive upload ./path/to/file --replace <fileId>
gog drive upload ./report.docx --convert
gog drive download <fileId> --out ./downloaded.bin
gog drive download <fileId> --format pdf --out ./exported.pdf

gog drive mkdir "New Folder" --parent <parentFolderId>
gog drive rename <fileId> "New Name"
gog drive move <fileId> --parent <destinationFolderId>
gog drive delete <fileId>

gog drive permissions <fileId>
gog drive share <fileId> --to user --email user@example.com --role reader
gog drive unshare <fileId> --permission-id <permissionId>
gog drive drives --max 100  # shared drives
```

### Docs / Slides

```bash
gog docs info <docId>
gog docs cat <docId> --max-bytes 10000
gog docs create "My Doc"
gog docs copy <docId> "My Doc Copy"
gog docs export <docId> --format pdf --out ./doc.pdf
gog docs export <docId> --format docx --out ./doc.docx
gog docs export <docId> --format txt --out ./doc.txt

gog slides info <presentationId>
gog slides create "My Deck"
gog slides copy <presentationId> "My Deck Copy"
gog slides export <presentationId> --format pdf --out ./deck.pdf
gog slides export <presentationId> --format pptx --out ./deck.pptx
```

### Sheets

```bash
gog sheets metadata <spreadsheetId>
gog sheets get <spreadsheetId> 'Sheet1!A1:B10'
gog sheets export <spreadsheetId> --format pdf --out ./sheet.pdf
gog sheets export <spreadsheetId> --format xlsx --out ./sheet.xlsx

gog sheets update <spreadsheetId> 'A1' 'val1|val2,val3|val4'
gog sheets update <spreadsheetId> 'A1' --values-json '[["a","b"],["c","d"]]'
gog sheets append <spreadsheetId> 'Sheet1!A:C' 'new|row|data'
gog sheets clear <spreadsheetId> 'Sheet1!A1:B10'
gog sheets format <spreadsheetId> 'Sheet1!A1:B2' --format-json '{"textFormat":{"bold":true}}'
gog sheets create "My New Spreadsheet" --sheets "Sheet1,Sheet2"
gog sheets copy <spreadsheetId> "My Sheet Copy"
```

### Contacts

```bash
gog contacts list --max 50
gog contacts search "Ada" --max 50
gog contacts get people/<resourceName>
gog contacts get user@example.com

gog contacts other list --max 50
gog contacts other search "John" --max 50

gog contacts create --given "John" --family "Doe" --email "john@example.com" --phone "+1234567890"
gog contacts update people/<resourceName> --given "Jane" --email "jane@example.com"
gog contacts delete people/<resourceName>

gog contacts directory list --max 50  # Workspace
gog contacts directory search "Jane" --max 50
```

### Tasks

```bash
gog tasks lists --max 50
gog tasks lists create <title>

gog tasks list <tasklistId> --max 50
gog tasks get <tasklistId> <taskId>
gog tasks add <tasklistId> --title "Task title" --due 2026-02-20
gog tasks add <tasklistId> --title "Weekly sync" --due 2026-02-01 --repeat weekly --repeat-count 4
gog tasks update <tasklistId> <taskId> --title "New title"
gog tasks done <tasklistId> <taskId>
gog tasks undo <tasklistId> <taskId>
gog tasks delete <tasklistId> <taskId>
gog tasks clear <tasklistId>
```

### Chat (Workspace only)

```bash
gog chat spaces list
gog chat spaces find "Engineering"
gog chat spaces create "Engineering" --member alice@company.com --member bob@company.com

gog chat messages list spaces/<spaceId> --max 5
gog chat messages list spaces/<spaceId> --thread <threadId>
gog chat messages list spaces/<spaceId> --unread
gog chat messages send spaces/<spaceId> --text "Build complete!"

gog chat threads list spaces/<spaceId>

gog chat dm space user@company.com
gog chat dm send user@company.com --text "ping"
```

### Groups (Workspace)

```bash
gog groups list
gog groups members engineering@company.com
```

### Classroom (Workspace for Education)

```bash
gog classroom courses list
gog classroom courses get <courseId>
gog classroom courses create --name "Math 101"
gog classroom roster <courseId>
gog classroom coursework list <courseId>
gog classroom submissions list <courseId> <courseworkId>
gog classroom announcements list <courseId>
```

### People

```bash
gog people me
gog people get people/<userId>
gog people search "Ada Lovelace" --max 5
gog people relations
```

### Keep (Workspace only)

```bash
gog keep list --account you@yourdomain.com
gog keep get <noteId> --account you@yourdomain.com
gog keep search <query> --account you@yourdomain.com
```

### Time

```bash
gog time now
gog time now --timezone UTC
```

## Global Flags

- `--account <email|alias|auto>` - Account to use (overrides GOG_ACCOUNT)
- `--enable-commands <csv>` - Allowlist top-level commands
- `--json` - Output JSON to stdout
- `--plain` - Output stable TSV
- `--color <mode>` - Color mode: `auto`, `always`, or `never`
- `--force` - Skip confirmations for destructive commands
- `--no-input` - Never prompt; fail instead (CI)
- `--verbose` - Enable verbose logging
- `--help` - Show help for any command

## Service Scope Matrix

| Service | User | APIs | Notes |
| --- | --- | --- | --- |
| gmail | yes | Gmail API | modify + settings |
| calendar | yes | Calendar API | |
| chat | yes | Chat API | Workspace only |
| classroom | yes | Classroom API | Education only |
| drive | yes | Drive API | |
| docs | yes | Docs + Drive API | Export via Drive |
| contacts | yes | People API | + directory |
| tasks | yes | Tasks API | |
| sheets | yes | Sheets + Drive API | Export via Drive |
| people | yes | People API | OIDC profile |
| groups | no | Cloud Identity API | Workspace only |
| keep | no | Keep API | Workspace + service account |

## License

MIT
