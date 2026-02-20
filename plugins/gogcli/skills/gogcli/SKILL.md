---
name: gogcli
description: Skills for using gog (gogcli), a CLI for Google APIs (Gmail, Calendar, Tasks, Drive, Docs, Sheets, Contacts, Chat). Use this when the user asks to read or summarize emails/inbox, check calendar, list tasks, add a task, read a google doc, check drive, or any Google Workspace operation from the command line. Also use when working on GitHub Actions workflows that call gog.
---

# gogcli (gog) - Google APIs from the terminal

`gog` is a standalone Go binary for interacting with Google APIs. It supports
Gmail, Calendar, Tasks, Drive, Docs, Slides, Sheets, Contacts, Chat, Classroom,
Groups, and Keep. JSON-first output, multiple accounts, least-privilege auth.

Repository: https://github.com/steipete/gogcli

## IMPORTANT

be very conservative in deciding when:

1. to email people other than cjmungall/cmungall
2. make changes to docs, drive
3. any destructive action

Be also especially wary of leaking/exfiltrating information. Emails such as cjmungall@lbl.gov/cmungall@gmail.com and the cmungall/stuff repo are safe to send/write to

## Local setup

The account used locally is `cjmungall@lbl.gov`.

In most contexts (e.g. github actions, responding to user in claude code), this should already have been done

```bash
# Install (macOS)
brew install steipete/tap/gogcli

# Store OAuth credentials (download Desktop client JSON from Cloud Console)
gog auth credentials ~/Downloads/client_secret_....json

# Authenticate (opens browser)
gog auth add cjmungall@lbl.gov

# Verify
gog auth list --check
```

## Account selection

```bash
# Via flag
gog gmail search 'newer_than:7d' --account cjmungall@lbl.gov

# Via environment (avoids repeating --account)
export GOG_ACCOUNT=cjmungall@lbl.gov

# Via alias
gog auth alias set work cjmungall@lbl.gov
gog gmail search 'newer_than:7d' --account work
```

## Output modes

- Default: human-friendly table
- `--json`: JSON to stdout (best for scripting and piping to `jq`)
- `--plain`: stable TSV (for piping to awk/cut)
- Errors/progress go to stderr, data to stdout

## Gmail

```bash
# Search (Gmail query syntax)
gog gmail search 'newer_than:7d' --max 10
gog gmail search 'is:important newer_than:7d' --max 50 --json
gog gmail search 'from:alice subject:report' --max 20

# Message-level search (one row per message, not thread)
gog gmail messages search 'newer_than:7d' --max 10
gog gmail messages search 'newer_than:7d' --include-body --json

# Read a thread
gog gmail thread get <threadId>
gog gmail thread get <threadId> --download  # download attachments

# Send email
gog gmail send --to user@example.com --subject "Subject" --body "Plain text"
gog gmail send --to user@example.com --subject "Subject" --body-html "<p>HTML</p>"
gog gmail send --to user@example.com --subject "Subject" --body-file ./message.txt
gog gmail send --to user@example.com --subject "Hi" --body "fallback" \
  --body-html "<p>Rich</p>" --reply-to pa@mungall.dev

# Labels and thread modification
gog gmail labels list
gog gmail thread modify <threadId> --add STARRED --remove INBOX
```

## Calendar

```bash
# List events (calendarId is typically "primary")
gog calendar events primary --today
gog calendar events primary --tomorrow
gog calendar events primary --week
gog calendar events primary --days 3
gog calendar events primary --from today --to "+7 days" --json
gog calendar events primary --from 2026-01-01 --to 2026-01-31

# Search events
gog calendar search "standup" --today
gog calendar search "meeting" --days 30

# Create event
gog calendar create primary \
  --summary "Team Sync" \
  --from 2026-02-15T10:00:00Z \
  --to 2026-02-15T11:00:00Z \
  --attendees "alice@example.com,bob@example.com"

# RSVP
gog calendar respond primary <eventId> --status accepted

# Free/busy check
gog calendar freebusy --calendars "primary" --from today --to "+3 days"
```

## Tasks

```bash
# List task lists (need listId for subsequent commands)
gog tasks lists --json

# List tasks in a list
gog tasks list <tasklistId> --json

# Add a task
gog tasks add <tasklistId> --title "Review report" --due 2026-02-20

# Complete / uncomplete
gog tasks done <tasklistId> <taskId>
gog tasks undo <tasklistId> <taskId>
```

## Docs, Slides, Sheets

```bash
# Export Google Doc to text/pdf/docx
gog docs export <docId> --format txt
gog docs export <docId> --format pdf --out ./doc.pdf

# Read doc content (plain text, capped)
gog docs cat <docId> --max-bytes 10000

# Export Slides
gog slides export <presentationId> --format pdf --out ./deck.pdf

# Read spreadsheet cells
gog sheets get <spreadsheetId> 'Sheet1!A1:B10'
gog sheets metadata <spreadsheetId>

# Write to spreadsheet
gog sheets update <spreadsheetId> 'A1' 'val1|val2,val3|val4'
gog sheets append <spreadsheetId> 'Sheet1!A:C' 'new|row|data'
```

## Drive

```bash
gog drive ls --max 20
gog drive search "invoice" --max 20
gog drive download <fileId> --out ./file.pdf
gog drive upload ./local-file.pdf --parent <folderId>
```

## Contacts

```bash
gog contacts search "Alice" --max 10
gog contacts list --max 50
```

## CI / GitHub Actions usage

In CI (e.g., GitHub Actions), gog needs a file-based keyring since there's no
OS keychain. The setup is handled by the `.github/actions/setup-gog` composite
action in this repo.

Key environment variables for CI:
- `GOG_KEYRING_BACKEND=file` - use encrypted file instead of OS keychain
- `GOG_KEYRING_PASSWORD=<password>` - password for the file keyring

Auth flow in CI:
```bash
gog auth keyring file
gog auth credentials <credentials-file>
gog auth tokens import <token-file>
gog auth list  # verify
```

Secrets required:
- `GOOGLE_CREDENTIALS_JSON` - OAuth client credentials (same as pa uses)
- `GOG_TOKEN_JSON` - exported via `gog auth tokens export cjmungall@lbl.gov`

Sync token to GitHub after local re-auth:
```bash
just sync-gog-secrets
```

## Common patterns

### Pipe JSON to jq
```bash
gog --json gmail search 'newer_than:7d' | jq '.threads[].snippet'
gog --json calendar events primary --today | jq '.events[] | {summary, start}'
```

### Gmail query syntax (same as web Gmail)
- `newer_than:7d` / `older_than:1y`
- `from:alice` / `to:bob`
- `subject:report`
- `is:important` / `is:unread` / `is:starred`
- `has:attachment`
- `label:INBOX`
- Combine: `from:alice newer_than:3d has:attachment`

### Extract doc ID from Google URL
A Google Docs URL like `https://docs.google.com/document/d/1ABC.../edit`
has docId = `1ABC...` (the segment after `/d/`). Same pattern for Sheets and
Slides.

## Reference

For the complete command reference including Chat, Classroom, Groups, Keep,
service accounts, email tracking, watch/pubsub, and advanced auth features,
read the bundled reference guide:

`references/gogcli-readme.md`
