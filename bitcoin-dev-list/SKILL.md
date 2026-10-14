---
name: bitcoin-dev-list
description: Read, search and fetch bitcoin-dev mailing list posts and threads (2011 to now) from a local git mirror of the gnusha public-inbox, with optional index and thread-export tools. Use when asked what the list said about a topic, for the rationale behind a BIP or design choice, to find who proposed something and when, to export a thread as text, or when given a gnusha.org, lists.linuxfoundation.org or groups.google.com/g/bitcoindev link.
---

# bitcoin-dev mailing list: local mirror

The whole list (bitcoin-development on SourceForge, bitcoin-dev on Linux
Foundation, bitcoindev on Google Groups) is archived by the public-inbox at
https://gnusha.org/pi/bitcoindev/ and that inbox is a git repository. It is
available as a local mirror when configured. Prefer local searching and
reading; use the archive links below when no mirror is available.

## Local archive setup

Locate an existing mirror and any index/thread-export helpers from the user's
configuration or repository documentation. Do not assume they are installed at
fixed paths. Use these variables in the examples below, replacing the
placeholders with the discovered paths:

```bash
BITCOIN_DEV_LIST_ROOT="<archive-root>"
BITCOIN_DEV_LIST_TOOLS="<helper-tools-directory>"
```

- `$BITCOIN_DEV_LIST_ROOT/mirror.git` — a checkout cloned from
  `https://gnusha.org/pi/bitcoindev`. Public-inbox v1 format: one raw RFC822
  message per file at `xx/<sha1>`. Refresh when needed with
  `git -C "$BITCOIN_DEV_LIST_ROOT/mirror.git" pull`.
- `$BITCOIN_DEV_LIST_ROOT/index.tsv` — one row per message: `date, path,
  message_id, parent, from, subject`.
- `$BITCOIN_DEV_LIST_TOOLS/index.py` and `thread.py` — optional external
  helpers, not bundled with this skill. Inspect their interfaces and confirm
  which archive/output paths they use before running them. The commands below
  assume `index.py` rebuilds the index after a pull and `thread.py` accepts a
  subject regex or Message-ID. If the helpers are unavailable or incompatible,
  use the raw mirror messages or the web archive instead.
- Thread exports conventionally contain one `NNN-<date>-<author>.txt` per
  message plus `thread.txt` (all concatenated). Read the output directory from
  the export tool rather than assuming a fixed location. Each message should
  carry From, Date, Subject, Message-ID, In-Reply-To and its gnusha URL, then
  the decoded body (HTML-only mails flattened to text).

## Workflow

1. **Find the thread** in the index, by subject, author or date window:
   ```
   awk -F'\t' 'NR>1 && $1>="2019-01" && $1<"2020-01" && tolower($6) ~ /taproot/ {print $1"\t"$5"\t"$6}' "$BITCOIN_DEV_LIST_ROOT/index.tsv"
   ```
   For a body-text search, search the raw files of a date window (choose an
   unused scratch filename and remove it afterward):
   ```
   awk -F'\t' 'NR>1 && $1>="2018-06" && $1<"2022-01" {print $2}' "$BITCOIN_DEV_LIST_ROOT/index.tsv" > "$BITCOIN_DEV_LIST_ROOT/search-paths.txt"
   cd "$BITCOIN_DEV_LIST_ROOT/mirror.git"
   xargs rg -l -i "ext_flag|spend_type" < "$BITCOIN_DEV_LIST_ROOT/search-paths.txt"
   rm "$BITCOIN_DEV_LIST_ROOT/search-paths.txt"
   ```
   then map paths back to rows with a small Python join on column 2.
2. **Export the thread** as readable text (subject regex, or a Message-ID
   including the angle brackets; any message of the thread finds its root):
   ```
   python3 "$BITCOIN_DEV_LIST_TOOLS/thread.py" '^\[bitcoin-dev\] Taproot proposal$'
   python3 "$BITCOIN_DEV_LIST_TOOLS/thread.py" '<message-id>'
   ```
   It prints message count, root date and the output directory. Anchor the
   regex; `Taproot` alone matches dozens of threads. Messages are ordered by
   date, so a reply that was archived before its parent can appear first.
3. **Read** `thread.txt`, or search it with context; drop quoted lines with
   `grep -v '^>'` when hunting for an author's own words.
4. **Cite** with the gnusha URL printed in each message header:
   `https://gnusha.org/pi/bitcoindev/<message-id without brackets>/`.
   These are stable and work for every era of the list.

## Mapping other archive links

- `gnusha.org/pi/bitcoindev/<msgid>/` — the Message-ID is the path; look it up
  in column 3 of the index (with angle brackets).
- `lists.linuxfoundation.org/pipermail/bitcoin-dev/YYYY-Month/NNNNNN.html` —
  no Message-ID in the URL; search the index by month and subject. If the
  page is still needed, fetch it: the subject and date are in the title.
- `groups.google.com/g/bitcoindev/c/<thread>/m/<post>` — same: search by
  subject and date. The Google Groups message id is not the RFC822 one, but
  the gnusha copy of a post has the same `Message-ID` as the mail, and
  Google's `d/msgid/bitcoindev/<msgid>` links map directly.
- To fetch one message or thread from the web without the mirror:
  `https://gnusha.org/pi/bitcoindev/<msgid>/raw` (message) and
  `https://gnusha.org/pi/bitcoindev/<msgid>/t.mbox.gz` (whole thread as mbox).
  Public-inbox search: `https://gnusha.org/pi/bitcoindev/?q=<query>&x=m`
  returns matching messages as an mbox.

## Gotchas

- zsh: `echo ====` and unquoted `--include=*.cpp` are globs; quote them.
- Quoted text dominates long threads. Distinguish an author's claim from
  what they quote before attributing it.
- Threading follows In-Reply-To, falling back to the last References entry;
  a reply whose parent never reached the list starts its own thread.
- When a thread was exported before a pull, re-export it. Check whether the
  helper overwrites an existing export before running it.
