# video-edit skill

Cut, resize, trim, thumbnail and publish an mp4, from a Zoom export to a finished
file, with a human review gate before anything reaches a public or member-facing
surface.

Built for anyone who records recurring sessions (classes, workshops, coaching calls)
and turns them into an archive plus short clips. Every command and trap in the skill
was measured on a real working library of weekly recordings. The skill assumes
nothing about your folders, your naming, or your platforms until you answer four
setup questions.

## What it does

- Ingests a Zoom export set (video, audio, captions, chat) into a dated, named master
- Protects the master and records its identity (name, bytes, duration, fingerprint)
- Runs a head-and-tail review gate that catches recording bleed from persistent
  meeting rooms before anything is published
- Derives the standard set: a size-targeted re-encode, a thumbnail, a 1080p feed
  clip, and vertical story cuts, all from the master
- Verifies every derived file by probing it (dimensions, duration, audio loudness,
  hidden material behind edit lists)
- Confirms uploads at the far end, because some platforms accept a file and store
  nothing
- Keeps a one-row-per-recording index so the library stays findable

## Install

Copy the `video-edit` folder into your agent's skills directory. For Claude Code:

```bash
cp -R video-edit ~/.claude/skills/
```

Any agent that reads SKILL.md files can use it the same way. The scripts inside are
plain bash plus ffmpeg and ffprobe.

Requirements: ffmpeg and ffprobe on the machine doing the work. Commands are written
for macOS; the skill's intro lists the three GNU/Linux swaps.

## Before the first run

The skill opens with a setup phase. It will ask where masters live, where derived
files live, what your naming convention is, and what each variant is for. Until those
are answered it plans and reports only, and applies nothing. That is deliberate: the
one destructive mistake in a video library is moving or overwriting the only copy of
a session, and the skill is built to make that mistake hard to make.

## The part worth stealing even if you skip the rest

Phase 3, the review gate. Persistent Zoom rooms keep recording past the booked
session, and the extra material lands in the same file as the class. The gate sweeps
the head and tail of every recording as contact sheets (about 2 seconds of compute
per hour of video) and a human reads them before anything is derived or published.
Silence detection cannot find the seam; Zoom room tone sees to that. The sheets can.


---
Free to use and share.

---

Maintained by Katie Van Nes of Most Referred Expert. The Lofty setups get taught live and free
every week at [LoftyMasterClass.com](https://LoftyMasterClass.com), and the recordings live in the
Most Referred Expert Community.
