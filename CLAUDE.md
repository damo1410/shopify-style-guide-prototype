# Working agreement for this repo

## Ship every update to the live demo

The prototype is a single static file (`index.html`) served via GitHub Pages
from `main`: https://damo1410.github.io/shopify-style-guide-prototype/

For **every** change, after the work is done and committed on the session's
feature branch:

1. Merge the feature branch into `main` (`git merge --no-ff`) and push `main`.
2. If the merge reports **conflicts**, stop and report them to the user with
   the conflicting files/hunks — do not try to resolve them unilaterally, since
   other sessions may be editing other parts of the prototype concurrently.
3. Give the user a **cache-busted view URL** so they see the latest without a
   stale browser cache — the Pages URL with the short commit SHA as a query
   param, e.g. `https://damo1410.github.io/shopify-style-guide-prototype/?v=<sha>`.

Concurrent sessions are expected. Each merges its own branch into `main`;
conflicts only surface for overlapping edits, and those get surfaced to the
user rather than auto-resolved.
