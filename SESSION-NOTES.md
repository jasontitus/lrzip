# lrzip compression research — session notes

Working notes from a Claude Code session, kept so work can resume on another
machine (or in a fresh session) with full context. Not intended as permanent
project documentation — fold anything durable into `WHATS-NEW` or
`doc/README.benchmarks` and delete this file once the PRs below are settled.

## Where things live

- **This branch** (`claude/lrzip-compression-research-6wc1rf`): the full
  research tree — zstd backend, retuned lzma, `--ultra` single-block mode,
  overlapped threading, per-block BCJ/delta filters, pre-rzip branch
  conversion (format 0.8). Everything described below except the upstream PR
  itself lives here. `doc/README.benchmarks` has the full internal
  before/after numbers for every stage.
- **`claude/max-compression-ultra`**: a deliberately minimal branch cut from
  clean `master`, containing *only* the `-u`/`--ultra` modifier — see
  "PR status" below. This is what's actually up for review upstream.
- **Fork**: `jasontitus/lrzip` (github.com/jasontitus/lrzip)
- **Upstream**: `ckolivas/lrzip` (github.com/ckolivas/lrzip) — Con Kolivas
  pushed a major 0.7.0 release (new AEAD encryption, LZMA SDK 26.02, zpaq
  7.15, streaming stdio format) the same week this session started. Fork was
  created fresh off that release.

## PR status (as of this note)

- **`jasontitus/lrzip#2`** — https://github.com/jasontitus/lrzip/pull/2
  Same content as the upstream PR below, opened first against the fork.
- **`ckolivas/lrzip#268`** (upstream, the one that matters) —
  https://github.com/ckolivas/lrzip/pull/268
  Title: "Add -u/--ultra opt-in maximum compression mode"
  Branch: `jasontitus:claude/max-compression-ultra` → `ckolivas:master`
  Status: **open, submitted, awaiting review.**
  Con Kolivas replied:
  > Thanks. Looks good in principle. Will review. If I was to extend the
  > file format it would be now before the 0.7 format propagates in the
  > wild, but I'm unsure there is demand for such changes.

  **This needs a reply posted** (drafted below, not yet confirmed posted —
  do that first on the laptop). The short version: PR #268 makes **no**
  format change at all — verified by decompressing a `-L9 --ultra` archive
  with an unmodified pre-PR lrzip binary, byte-identical. His format-window
  question only applies to the two follow-ups that were deliberately held
  back (see below), and is really a question about whether *those* should
  become PRs, not a blocker for #268.

### Reply to post on ckolivas/lrzip#268

```
Good question, and worth being precise about: this PR does not touch the
file format at all.

--ultra only changes encoder-side choices that the existing format already
accommodates — one block per stream instead of one per thread, a larger
dictionary, and 273 fast bytes for lzma. Those are ordinary LZMA/ZPAQ
parameters carried in the per-block header the format already has. I
confirmed this concretely: I built an unmodified pre-PR lrzip binary and
used it to decompress a -L9 --ultra archive — it came back byte-identical
to the source, with zero code changes needed on the reader side. No
magic-header, chunk-header, or block-type byte is touched anywhere in this
diff (5a71187).

So there's no urgency tied to this PR specifically — it's safe to merge on
its own regardless of what you decide about the format window.

The format-timing question does apply to two follow-ups I mentioned in the
PR description but held back, precisely because of the tradeoff you're
describing:

1. Per-block BCJ/delta filters (executable code and delta-friendly data
   like imagery) — this adds a few new block-type values. An old reader
   would only fail on a block that actually used one of the new filters;
   unfiltered blocks in the same archive stay fully compatible. Small,
   additive extension to the existing block-type byte.
2. Pre-rzip branch conversion — the one that would actually need a new
   byte in the chunk header itself. This is the real "extend the format"
   case.

If there's appetite for either, now would indeed be the moment to do it
before 0.7 sees wider adoption. But that's a separate decision from this
PR, and I'd rather let you weigh whether the extra 2-5% (filters) or ~2%
(pre-rzip) is worth it before building out either as a proper follow-up —
happy to open them if you want to see the numbers in more depth first.
```

Commit hash `5a71187` referenced above is the tip of `claude/max-compression-ultra`
at time of writing (`git log -1 --format=%H` on that branch to reconfirm
before posting, in case anything changed).

## What was built, in order

1. **zstd backend** (`-Z`/`--zstd`) — long-distance matching, per-level window
   log, optional at configure time (`--without-zstd`; verified byte-identical
   lzma output either way).
2. **LZMA retune** — direct level→dictionary mapping instead of the old 7/9
   rescale, larger dictionaries. *Not* in the upstream PR — see "held back"
   below; this branch has it unconditionally, the PR branch gates it.
3. **`-L9` single-block max mode** → generalized into **`-u`/`--ultra`** after
   a note that changing `-L9`'s own wall-clock behavior was a bad idea. This
   is the one thing that made it into the upstream PR.
4. **Overlapped threading** — half the blocks, double the size, two threads
   each, at non-ultra levels.
5. **Per-block prefilters** (`filters.c`, vendored public-domain LZMA SDK
   `Bra.c`/`Bra86.c`/`Delta.c`/`RotateDefs.h`) — x86 BCJ, ARM64 BCJ, delta
   1-4, chosen per block by trial compression. Extended to both lzma and
   zstd backends.
6. **Pre-rzip branch conversion (format 0.8)** — one new chunk-header byte;
   a duplicate-window probe decides whether to branch-convert a whole chunk
   *before* rzip sees it, since post-rzip filtering can't recover matches
   rzip already discarded. Refuses to fire when conversion would break
   existing long-range dedup (verified against a 3x-duplicated-file corpus).
7. **Minimal upstream PR carve-out** — realized mid-session that offering a
   PR which silently changes `-L9`'s behavior (speed *and* implicitly
   dictionary/fb settings even without `--ultra`) was not acceptable for an
   upstream contribution. Rebuilt from clean `master` as
   `claude/max-compression-ultra`: strictly opt-in, byte-identical to stock
   without `-u`, verified explicitly. This is the ~180-line diff that's
   actually up for review.

## Held back from the upstream PR (deliberately)

Offered as three follow-ups in the PR body, each independently gated, none
silent:

1. **LZMA level retune as its own flag** (e.g. `--lzma-tune`) — direct level
   mapping + bigger dictionaries at *normal threaded* levels. ~2.5% smaller
   at the default level. Exists unconditionally on this branch; would need
   to be re-gated behind a flag if pursued as a separate upstream PR, same
   as `--ultra` was.
2. **Per-block filters** (~250 new lines + vendored SDK sources) — 2-5% on
   executables/imagery. New block types; only unfiltered-archive
   compatibility is preserved with old readers, filtered blocks need the new
   version.
3. **Pre-rzip conversion** (~400 new lines, format 0.8) — last ~2% on x86,
   ~3.2% on aarch64 (which gets nothing from post-rzip filtering). Needs the
   new chunk-header byte. This is the one Con Kolivas's format-timing
   question is actually about.

Both 2 and 3 are fully implemented, tested, and benchmarked on this branch
already — turning either into a standalone PR is mostly a rebase-from-master
exercise (same technique used for the ultra PR: cut from clean master,
cherry-pick just that series, re-verify non-invocation is a no-op where
possible, write a fresh PR body).

## Benchmark numbers not yet committed anywhere

`doc/README.benchmarks` on this branch has all the internal before/after
numbers. The **head-to-head vs zstd and xz** competitor numbers were only
ever computed in the session scratchpad and are not persisted in any repo.
Reproduced here so they aren't lost. All on a 4-core/15GB VM, 1GB+ real
corpora, `lrzip -L9 -u` using the final gated (strictly-opt-in) binary from
`claude/max-compression-ultra`:

| Corpus | lrzip `-L9 -u` | zstd `-19` (plain) | zstd `--ultra -22 --long` | xz `-9e` |
|---|---|---|---|---|
| enwik9 (1GB wikipedia text) | 206.8 MB | 235.5 MB (+13.9%) | 214.0 MB (+3.4%) | 214.2 MB (+3.6%) |
| 1GB tar of ELF binaries | 224.7 MB | 286.0 MB (+27.3%) | 244.9 MB (+9.0%) | 244.8 MB (+8.9%) |
| single kernel tar (linux-6.6, 1.4GB) | 137.6 MB | 144.4 MB (+4.9%) | 136.4 MB (-0.9%) | 132.6 MB (-3.6%) |
| two kernel versions in one tar (linux-6.1 + 6.6, 2.8GB) | 146.8 MB | 283.7 MB (+93.3%) | 146.1 MB (-0.5%) | 260.7 MB (+77.6%) |

Takeaways: lrzip only clearly loses to xz on single-version source (no
long-range redundancy for rzip to find). The two-kernel row is the
demonstration case for lrzip's actual value proposition — competitors need
to be told to use a long-distance window (`--long=31` for zstd) and even
then only roughly tie; the tools people reach for by default
(`zstd -19`, `xz -9e`) roughly double in size because neither can see across
the ~1.4GB gap between the two archived trees.

## Test infra notes

- `tests/regression.sh` gained a "Part 3: ultra suite" (9 tests) on the PR
  branch: single-block round-trips for lzma/zpaq, `-p` override, encryption
  and stdio over ultra, a ratio sanity check, and two constrained-memory
  tests (`-m 3` must cap the dictionary and still round-trip). Run with
  `SKIP_GOLD=1 bash tests/regression.sh`.
- Test data used throughout: enwik9 (mattmahoney.net), linux-6.1/6.6 kernel
  tarballs (cdn.kernel.org), a tar of `/usr/bin`+`/usr/lib` binaries, ARM64
  kernel modules + Ubuntu base rootfs (ports.ubuntu.com), Silesia corpus
  (16-bit imagery for delta filter testing). None of this is committed —
  regenerate from the URLs in shell history if needed, or see
  `doc/README.benchmarks` for the exact commands.

## Suggested next steps

1. Post the reply above on `ckolivas/lrzip#268`.
2. Watch for Con Kolivas's review — GitHub PR subscription/notifications
   from the laptop, since this session's tool scope is locked to
   `jasontitus/lrzip` and can't reach `ckolivas/lrzip` directly (hit a
   "cross-tier adds are not supported" error trying to add it mid-session).
3. If he wants either follow-up, cut a fresh branch from `master` (or from
   wherever #268 lands after merge) the same way `claude/max-compression-ultra`
   was cut — don't just open the full research branch as a PR, it carries
   the lzma retune unconditionally which was deliberately excluded from the
   opt-in-only story upstream wants.
4. Delete this file once the above is resolved and durable info is folded
   into `WHATS-NEW`.
