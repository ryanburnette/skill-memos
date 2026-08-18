---
name: memos
description: Read and write notes in the personal Memos instance at memos.ryanburnette.com with the `memo` CLI. Use when asked to save a note, jot something down, look something up in memos, or search past notes.
---

Memos is a self-hosted note app at `https://memos.ryanburnette.com`. Use the
`memo` CLI, which wraps its REST API. It is on PATH in every harness (symlinked
to `~/bin/memo` by `setup.sh` from `memos/bin/memo` in the dotfiles repo), so it
works the same from Claude Code, pi, and a plain shell.

Reach for raw curl only for something `memo` does not cover; see REST fallback
at the end.

## Commands

Read:

    memo list [n]              newest n memos, one per line: id, then title
    memo search <text>         full text search
    memo tag <tag>             memos carrying #tag
    memo get <id>              print a memo's content
    memo shares <id>           share links for a memo
    memo whoami                check the token works

Write:

    memo new [-v VIS] [text]   create; body from args or stdin, prints the id
    memo title <id> <text>     set or replace the H1 title, body untouched
    memo edit <id>             replace the body with stdin
    memo rm [-f] <id>          delete; refuses without a tty unless -f
    memo vis <id> <VIS>        set visibility on an existing memo
    memo share <id> [-e TIME]  create a share link (-e sets an RFC 3339 expiry)
    memo share <id> rm <tok>   revoke a share link

IDs are bare (`AJDYGQZKpSfsT8BRQzGU2j`), not the `memos/...` resource name the
API returns. `memo` strips the prefix if you pass the long form anyway.

Writing a note is usually a heredoc:

    memo new <<'EOF'
    # cogburnbros deploy hangs on the rsync step without SSH agent forwarding

    Symptom, cause, fix. #deploy
    EOF

Visibility defaults to `PRIVATE`. `-v PROTECTED` is signed-in users, `-v PUBLIC`
is still not anonymous — see Sharing and Gotchas.

Change an existing memo's visibility with `vis`:

## Writing a title

Every memo starts with an H1 on the first line. `memo new` and `memo edit`
reject content that does not, so this is enforced, not merely advised. Memos
extracts the H1 into `property.title`; with no H1 the title is empty and the app
shows a truncated snippet of the body instead.

Write the title for a reader who has none of the context you have right now.
Memos are found months later through search or a list of one-line snippets,
stripped of the conversation, the terminal, and the repo that made them
obvious. The title is usually all that reader sees before deciding whether to
open the note, so it has to identify the thing on its own.

Name the specific subject, not a generic category, and say what actually
happened or what was decided. Use the real identifiers (hostname, repo,
service, error string) instead of pronouns and "this". If the title only makes
sense to someone who was in the room, rewrite it.

Too vague to find or trust later:

    # Fixed it
    # ZFS issue
    # Proxmox notes
    # Auth bug

Each of those fails the same way: no subject, or a subject so broad it collides
with every other note on the topic. Nothing tells the later reader which host,
which repo, or what the outcome was.

Self-contained:

    # nassau-pve1 loses its ZFS pool after reboot, fixed by zfs-import-cache.service
    # cogburnbros deploy hangs on the rsync step without SSH agent forwarding
    # Chose Bearer tokens over session cookies for the IT Docs SSO integration

Long is fine. A title that runs a full line and answers "which thing, and what
happened" beats a short one that needs the original conversation to decode.

The body carries the detail: what was tried, what the error said, the commands,
the links. Keep the body self-contained too. Spell out paths and hostnames on
first use rather than referring back to something the reader cannot see.

## Visibility

`PRIVATE` (default) is just you. `PROTECTED` is any signed-in user on the
instance. `PUBLIC` is **not** anonymous — see Gotchas; a `PUBLIC` memo fetched
by id still 401s without a token. Visibility is an access-control flag for the
signed-in audience, not a sharing mechanism.

Set it at creation with `memo new -v VIS`, or change it on an existing memo:

    memo vis <id> <PRIVATE|PROTECTED|PUBLIC>

`VIS` must be exactly `PRIVATE`, `PROTECTED`, or `PUBLIC`.

## Sharing

Share links are the only way to make a memo readable by someone with no account
and no token. A link is `/memos/shares/<token>` and is anonymously readable:
the SPA fetches the shared memo through an unauthenticated call to
`POST /memos.api.v1.MemoService/GetSharedMemo` with `{"shareToken":"<token>"}`,
so anyone holding the URL can read the memo. Treat a share link as outward-
facing — it exposes the note to anyone who has the URL, and needs explicit
sign-off before you create one.

The CLI covers create, list, and revoke:

    memo share <id> [-e EXPIRE]   create a share link; prints token to stdout,
                                  public URL to stderr. -e takes an RFC 3339
                                  expiry, e.g. 2026-08-19T00:00:00Z
    memo shares <id>              list share tokens and create times
    memo share <id> rm <token>    revoke a share link

The public URL is `https://memos.ryanburnette.com/memos/shares/<token>`. There
is no GET on an individual share (501); use `memo shares` to recover a token.
A memo need not be `PUBLIC` to be shared — the share token is what grants
access, not visibility.

## Gotchas

- `PUBLIC` does not mean anonymously readable. Every `/api/v1` read on this
  instance returns 401 without a token, including a `PUBLIC` memo fetched by
  its exact id, and `https://memos.ryanburnette.com/memos/<id>` requires
  sign-in. Sharing is a separate mechanism: share links of the form
  `/memos/shares/<token>`, created over the API (see Sharing), which can
  expire. Never tell someone a memo URL is shareable because its visibility is
  `PUBLIC` — only `/memos/shares/<token>` is.
- Tags come from `#hashtag` text in the body; the `tags` field is read-only
  output. To tag a note, put the hashtag in the content.
- The human-facing URL is `https://memos.ryanburnette.com/memos/<id>`. Not
  `/m/<id>`, which returns 200 to curl because the frontend is a SPA, then
  404s in the browser.
- Editing replaces the whole body. To change only the title, use `memo title`,
  which swaps the H1 and leaves everything below it byte-identical.

## Auth

`memo` resolves the token itself: `MEMOS_TOKEN` from the environment if a caller
set one, otherwise `secret memos`, which reads `~/.config/secrets/memos` (mode
0600) or the OS keychain. The token is deliberately **not** in the ambient
environment, so `echo $MEMOS_TOKEN` is empty in a normal shell and that is
correct, not a misconfiguration. Nothing to set up per session.

Never echo the token or paste it into a message. It is an admin PAT with delete
rights over every note, which is also why `memo rm` refuses to run unattended
without `-f`.

For the raw curl fallback below, get it the same way rather than reaching into
any env file:

    A="Authorization: Bearer $(secret memos)"

## REST fallback

For anything the CLI lacks (comments, reactions, attachments, relations,
pinning, paging past the first page):

    B=https://memos.ryanburnette.com/api/v1
    A="Authorization: Bearer $(secret memos)"
    curl -s -H "$A" "$B/memos/$ID/comments"
    curl -s -X PATCH -H "$A" -H 'Content-Type: application/json' \
      -d '{"pinned":true}' "$B/memos/$ID?updateMask=pinned"

`filter` is CEL. Verified working: `content.contains("text")`,
`tag in ["cogburn"]`, `visibility == "PRIVATE"`, `pinned == true`.
`tags.contains(...)` does not compile.

`orderBy` accepts only `pinned`, `create_time`, `update_time`, `name`, each
optionally with ` desc`. Paging is `pageSize` plus `pageToken` from
`.nextPageToken`.

Errors sometimes arrive as HTTP 200 with a `.message` field, so check the body,
not just the status.

## Safety

Read freely. Creating a note is fine when asked. Editing or deleting an existing
note needs explicit confirmation first; there is no undo and the token reaches
every note in the account. Creating a share link is outward-facing — confirm
with the user first, since it exposes the note to anyone holding the URL.
