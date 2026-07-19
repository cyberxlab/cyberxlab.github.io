---
title: "The 43-Byte Convention That Protects Backups: Cache Directory Tagging in Practice"
date: 2026-07-19T20:25:08Z
draft: false
tags: ["Backup", "CACHEDIR.TAG", "Interoperability"]
categories: ["Technical Practices"]
author: "Cyber·X·Lab"
description: "A practical engineering walkthrough of the Cache Directory Tagging Specification (CACHEDIR.TAG): the 43-byte signature, application semantics, adopter behavior in tar/Borg/restic, and the pitfalls of blind exclusion."
summary: "How a 43-byte magic header — Signature: 8a477f597d28d172789f06886806bc55 — lets apps mark cache trees so tar, Borg and restic skip them by default; the spec's exact semantics, adopter quirks, and the security tradeoff that makes blind exclusion dangerous."
toc: true
---

## Background and Motivation

In 2004 Bryan Ford proposed a deliberately tiny convention to solve a problem that had been quietly bothering backup software authors for years: applications create *cache directories* — browser on-disk caches, thumbnail trees, Gradle's `~/.gradle/caches`, Mozilla's `~/.mozilla/.../Cache`, ccache's compiler-output store — that have no long-term archival value, balloon to gigabytes, churn on every page view (defeating incremental backup deltas), and use cryptic hash-derived filenames that confuse users browsing their own home directory ([Ford 2004](https://bford.info/cachedir/)). The location standards of the era pulled in conflicting directions: the *Filesystem Hierarchy Standard* prescribed a system-wide `/var/cache` (root-only, useless for per-user apps), and *XDG Base Directory* recommended `$XDG_CACHE_HOME` — but each user could override the env var in their shell profile, so backup tools scanning `$HOME` could not reliably enumerate per-user caches from one machine to the next.

Ford's proposal sidesteps the location debate entirely: instead of arguing about *where* caches live, agree on **how a directory identifies itself as a cache**. The result is the **Cache Directory Tagging Specification** ([`bford.info/cachedir/`](https://bford.info/cachedir/)) — a four-line rule that 20 years later is honored by GNU tar, BorgBackup, restic, Gradle, ccache, Git's object cache, and several browsers. The spec is small enough to fit on a sticky note, which is precisely why it succeeded where grander, normative "put your cache *here*" standards had failed.

This article unpacks the engineering behind the convention: the exact byte layout of `CACHEDIR.TAG`, the directory-level semantics (tag the *topmost* cache directory, the tag covers the whole subtree), adopter behavior (restic excludes the *contents* but keeps `CACHEDIR.TAG` itself; tar offers three flavors of exclusion), and the under-discussed security tradeoff that means backup software must *verify* the 43-byte header before honoring the tag — otherwise a single attacker-dropped `CACHEDIR.TAG` can blind your backups to arbitrary data.

## Prerequisites

- A Unix-like system with `tar` 1.28+ (for `--exclude-caches-all`) or a Go runtime if you want restic, or Python 3 if you want Borg (`pip install borgbackup`).
- A test directory tree with at least one cache-shaped subdirectory you can dispense with; do **not** try the destructive variants on your home directory first time out.
- Optional: a hex editor or `xxd` to inspect the exact 43-byte header.
- Read access to the canonical spec at [`https://bford.info/cachedir/`](https://bford.info/cachedir/) (version 0.6, unchanged since 2004). The reference text lives on Bryan Ford's home page and is mirrored in the spec's `Changes` section.

## Step 1: Preparation

Decide what you want to tag. The spec is explicit about the noun: a *cache directory* is a directory whose entire contents (including all subdirectories, transitively) are **regenerable** from source material located elsewhere ([Ford 2004, "Application Semantics"](https://bford.info/cachedir/)). If any artifact under the directory is a *primary* output — a Git repo's working files, an `~/Documents` tree, a database dump — the directory is **not** a cache and must not be tagged. Tagging the wrong directory is the failure mode the 43-byte signature exists to prevent.

Set up a scratch tree to play with:

```bash
mkdir -p ~/taggedir-demo/{real-data,~/cache/gradle-build,~/cache/thumbs}
echo "primary content"          > ~/taggedir-demo/real-data/notes.txt
echo "blob A"                   > ~/taggedir-demo/cache/gradle-build/aux.json
echo "thumbnail bytes go here" > ~/taggedir-demo/cache/thumbs/cover.png.json

# Verify you can see everything first
find ~/taggedir-demo -type f | sort
```

Pick a target you do not mind losing. In this walkthrough the `cache/` subtree is the sacrificial cache; `real-data/` is what backup must never drop. You will run the same `tar`/`borg`/`restic` command twice — first *without* tagging (everything is archived) and then *with* a `CACHEDIR.TAG` placed in `cache/` (the caches disappear from the archive but `real-data/` survives, untouched).

## Step 2: Core Implementation

### 2.1authoring the tag

The spec pins down exactly two things: the filename (`CACHEDIR.TAG`, uppercase, no extension) and the first 43 octets of the file. Everything else — comments, application name, branding — is informational.

```bash
write_cachedir_tag() {
  local dir="$1" app="$2"
  [ -d "$dir" ] || { echo "no such dir: $dir" >&2; return 1; }
  printf 'Signature: 8a477f597d28d172789f06886806bc55\n# This file is a cache directory tag created by %s.\n# For information about cache directory tags, see https://bford.info/cachedir/\n' "$app" \
    > "$dir/CACHEDIR.TAG"
}

write_cachedir_tag ~/taggedir-demo/cache "demo-script"
```

Three implementation details worth memorizing:

- The 43-octet header is the **literal** string `Signature: 8a477f597d28d172789f06886806bc55`. The spec mandates ASCII, mandates case (`Signature` capital-S, lowercase hex), mandates that there is **exactly one** space after the colon, and forbids any byte before the `S` — no BOM, no leading whitespace, no shebang ([Ford 2004](https://bford.info/cachedir/)). The Hacker News discussion's BOM suggestion was correctly rejected for this reason: a "run of Unicode whitespace" forces every consumer to ship a Unicode database, which defeats the point of a 43-byte contract ([HN thread, Nov 2024](https://news.ycombinator.com/item?id=42093541)).
- The hex value is the MD5 of the string `.IsCacheDirectory`. Ford picked a magic value rather than a bare sentinel word so that an accidental, unrelated file named `CACHEDIR.TAG` — for instance a README renamed by a confused script — would **not** silently cause backup software to drop a directory. A 32-hex-digit digest is astronomically unlikely to collide by accident.
- The file **must** be a regular file, not a symlink. Backup tools that blindly follow a symlink with the right name would otherwise let an attacker point a single `CACHEDIR.TAG` symlink at `/etc` and exclude arbitrary trees — the spec removes that foot-gun at the protocol level.

### 2.2 Verifying the header

Adopters vary in how strict they are. The spec only *recommends* checking the 43-byte header; the HN thread notes GNU tar's `--exclude-caches` does verify it, and BorgBackup's `--exclude-caches` explicitly "verify the signature of the `CACHEDIR.TAG` file wherever it's found" before excluding the directory and its sub-directories ([Unix.SE answer on CACHEDIR.TAG vs .nobackup](https://unix.stackexchange.com/questions/724711/are-cachedir-tag-and-nobackup-evaluated-differently)). You want the verification on by default — see the security section.

Quick self-check, with no dependencies beyond `xxd`:

```bash
verify_cachedir_tag() {
  local f="$1/CACHEDIR.TAG"
  [ -f "$f" ] || { echo "missing or not regular file: $f"; return 1; }
  local first43
  first43=$(head -c 43 "$f")
  if [ "$first43" = 'Signature: 8a477f597d28d172789f06886806bc55' ]; then
    echo "OK: $f recognized as cache directory tag"
    return 0
  else
    echo "BAD header: '$first43'"; xxd "$f" | head -3; return 1
  fi
}

verify_cachedir_tag ~/taggedir-demo/cache
```

### 2.3 The application-side invariants

Anyone writing the tag on behalf of a real app has to honor three invariants the spec pins down in its "Application Semantics" section:

1. **One tag per cache subtree.** Write a single `CACHEDIR.TAG` into the *topmost* directory whose entire contents represent cached information. Do **not** litter tags into every leaf cache directory — that wastes space and confuses some consumers that disable exclusion once they descend.
2. **The tag covers the entire subtree.** Backup tools that honor the tag exclude the directory *and all subdirectories* without re-walking. If you mix cache content with primary content under one tagged directory, the primary content will also be excluded — silently. The directory layout is the contract.
3. **Regenerate the tag if it disappears.** If the user (or `rm -rf`) clears the cache but does not delete the directory itself, the application should rewrite `CACHEDIR.TAG` on next launch, otherwise the directory stops being recognized as a cache until the next full run. Gradle, ccache, and Borg's own cache directory all do this; a from-scratch script that forgets this is the common bug.

### 2.4 Backup-tool invocation

Three of the most common adopters, with the differences that bite:

```bash
# GNU tar — three closely-related flags, each with different file-retention behavior:
tar -czf ~/backup.tar.gz -C ~/taggedir-demo .                       # baseline: archive everything
tar -czf ~/backup.tar.gz --exclude-caches-all -C ~/taggedir-demo .  # exclude cache dirs AND the CACHEDIR.TAG itself
tar -czf ~/backup.tar.gz --exclude-caches      -C ~/taggedir-demo .  # exclude cache contents, KEEP CACHEDIR.TAG (default safe)
tar -czf ~/backup.tar.gz --exclude-caches-under -C ~/taggedir-demo . # exclude contents, keep tag, do not recurse into subtree

# restic — single flag, "keep the tag" semantics:
restic -r /srv/restic-repo backup ~/taggedir-demo --exclude-caches
# Per restic docs: "exclude a folder's content if it contains the special CACHEDIR.TAG file, but keep CACHEDIR.TAG."
# (https://restic.readthedocs.io/en/latest/040_backup.html)

# BorgBackup — analogous flag, signature-verified:
borg create --exclude-caches ~/borg-repo::'{now}' ~/taggedir-demo
```

The three `tar` flags (`--exclude-caches`, `--exclude-caches-all`, `--exclude-caches-under`) differ only in whether the cache directory *itself* and the `CACHEDIR.TAG` file survive into the archive. For reproducible backups of a developer tree prefer `--exclude-caches` (the default-safe middle option): it preserves enough metadata that the cache directory still exists on restore and the owner app can repopulate it transparently.

The Unix.SE comparison also clarifies how `--exclude-caches` differs from Borg's generic `--exclude-if-present NAME`. The latter matches any file with a given name — including `.nobackup` — and does **not** verify the 43-byte header; it is a free-form hint mechanism, not a spec-governed cache marker. `--exclude-caches` is the strict, signature-checked variant; `--exclude-if-present=NAME` is the tolerant escape hatch ([Unix.SE](https://unix.stackexchange.com/questions/724711/are-cachedir-tag-and-nobackup-evaluated-differently)).

## Step 3: Verification and Tuning

Now exercise the full round-trip to make sure your tag is doing what you think:

```bash
# 1. Baseline (no tag anywhere): the whole tree is archived
( cd ~/taggedir-demo && rm -f cache/CACHEDIR.TAG; \
  tar -czf /tmp/full.tar.gz -C ~/taggedir-demo . )
tar -tzf /tmp/full.tar.gz | grep -E 'cache/(gradle-build|thumbs|CACHEDIR.TAG)' | sort
# expect: ./cache/gradle-build/aux.json, ./cache/thumbs/cover.png.json  (cache contents present)

# 2. Add the tag in cache/ only:
write_cachedir_tag ~/taggedir-demo/cache "demo-script"

# 3. Archive with --exclude-caches:
tar -czf /tmp/skip.tar.gz --exclude-caches -C ~/taggedir-demo .
tar -tzf /tmp/skip.tar.gz | grep -E 'cache/(gradle-build|thumbs|CACHEDIR.TAG|.*\.txt)' | sort
# expect: ONLY ./cache/CACHEDIR.TAG  (content gone, tag survives, real-data untouched)
tar -tzf /tmp/skip.tar.gz | grep real-data
# expect: ./real-data/notes.txt  (never excluded because no CACHEDIR.TAG there)

# 4. Try --exclude-caches-all (everything in the cache subtree disappears, including the tag):
tar -czf /tmp/all.tar.gz --exclude-caches-all -C ~/taggedir-demo .
tar -tzf /tmp/all.tar.gz | grep -E 'cache/' || echo "no cache entries — tag itself excluded"
```

### Tuning pitfalls

- **Nested caches with extra tags are wasteful.** If `cache/gradle-build/` also contains its own `CACHEDIR.TAG`, the inner tag never changes behavior (the outer one already excluded the subtree), but it survives into other archivers' trees and can confuse `du --exclude-caches` accounting. The spec is explicit: one tag at the top.
- **`cache/` is a *hint*, not permission for safe deletion.** System software "should **not** periodically go through and unilaterally delete 'old' files in arbitrary cache directories on a purely automatic basis" ([Ford 2004, "Application Semantics"](https://bford.info/cachedir/)). Different apps age caches differently; a clean-tmp cron that nukes tagged trees will eventually eat a cache whose content takes minutes-per-file to regenerate.
- **Don't trust the header lazily.** If you write a custom `find`-based exclude loop that sniffs for `CACHEDIR.TAG` by name only — without validating the first 43 bytes — an attacker who can write to `~/.config/CACHEDIR.TAG` can make your backup silently ignore `~/.config`, which then cannot be restored. Always pair "found by name" with "verified by header" before excluding.
- **Test against `.nobackup` users.** Some workflows historically used an empty `.nobackup` sentinel with the same intent. They are **not** the same: `.nobackup` is honored only via `--exclude-if-present .nobackup`, has no header to verify, and has no spec. Migrating a team from `.nobackup` to `CACHEDIR.TAG` requires both writing the tag *and* flipping the backup command from `--exclude-if-present .nobackup` to `--exclude-caches`.

## Best Practices Summary

- **Tag the topmost cache directory, once.** A single `CACHEDIR.TAG` at the cache root excludes the whole subtree for every compliant adopter; nested tags add zero protection and confuse space accounting.
- **Always verify the 43-byte header.** Adopters that skip this step (custom scripts, naive `find` filters) open the door to data-hiding attacks. Match the strictness tar and Borg already ship.
- **Prefer `--exclude-caches` over `--exclude-caches-all`** for restore fidelity — the cache directory and tag survive, so apps repopulate transparently on next start.
- **Treat the tag as a backup hint, not a deletion license.** Never auto-delete expired cache files based on the tag alone; apps disagree on what "old" means and regeneration cost varies by orders of magnitude.
- **Migrate teams off `.nobackup`.** Replace name-based sentinels with the signature-verified spec wherever the backup tool chain allows; document the flag change in your runbook.

## References

- Bryan Ford, *Cache Directory Tagging Specification* (v0.6, 2004) — [`https://bford.info/cachedir/`](https://bford.info/cachedir/)
- Hacker News discussion (Nov 9 2024, `networked`) — [`https://news.ycombinator.com/item?id=42093541`](https://news.ycombinator.com/item?id=42093541)
- `telcoM`, "Are CACHEDIR.TAG and .nobackup evaluated differently?" Stack Exchange answer — [`https://unix.stackexchange.com/questions/724711/are-cachedir-tag-and-nobackup-evaluated-differently`](https://unix.stackexchange.com/questions/724711/are-cachedir-tag-and-nobackup-evaluated-differently)
- restic, *Backing up: excluding files* (`--exclude-caches`) — [`https://restic.readthedocs.io/en/latest/040_backup.html`](https://restic.readthedocs.io/en/latest/040_backup.html)
- Gradle, `MarkingStrategy.CACHEDIR_TAG` reference — [`https://docs.gradle.org/current/kotlin-dsl/gradle/org.gradle.api.cache/-marking-strategy/-c-a-c-h-e-d-i-r_-t-a-g.html`](https://docs.gradle.org/current/kotlin-dsl/gradle/org.gradle.api.cache/-marking-strategy/-c-a-c-h-e-d-i-r_-t-a-g.html)
- GNU tar manual, *Excluding Some Files* (`--exclude-caches`, `--exclude-caches-all`, `--exclude-caches-under`) — [`https://www.gnu.org/software/tar/manual/html_node/exclude.html`](https://www.gnu.org/software/tar/manual/html_node/exclude.html)
