---
name: video-edit
description: Cut, resize, trim, thumbnail and publish an mp4, from a Zoom export to a finished file, with a mandatory head-and-tail review before anything reaches a public or member-facing surface. Use when the operator says "cut me a clip", "edit this mp4", "trim the recording", "make a story cut", "shrink this video", "grab a thumbnail", "rename the Zoom exports", or asks whether a recording is safe to post. Requires ffmpeg and ffprobe.
---

# Video edit: Zoom export to finished file

This skill grew out of a working library of weekly class recordings: long Zoom sessions
that get archived in full, cut into short clips for social, and published to a community
platform every week. Every behaviour in it was measured on real files, and the traps are
written down because each one produced a bad file that looked fine.

The pipeline has seven phases. Phases 1, 2, 4, 5 and 7 are mechanical. Phase 3 is a human
review gate and never auto-passes. Phase 6 is a human decision with a mechanical
confirmation at the far end.

Commands are written for macOS (BSD userland). On GNU/Linux, swap `stat -f %z` for
`stat -c %s`, `date -j -u -f` for `date -u -d`, and `shasum -a 256` for `sha256sum`.
The ffmpeg and ffprobe calls are identical.

---

## Phase 0: setup. Answer these before the first apply

The skill parameterises everything it cannot know. Until the variables are set, run
every phase in plan mode and apply nothing. Reading, probing, sweeping and reporting
are always safe.

```bash
export VID_MASTERS="${VID_MASTERS:?ask the operator where masters live}"
export VID_DERIVED="${VID_DERIVED:?ask the operator where derived files live}"
export VID_REVIEW="${VID_REVIEW:-$HOME/Video-Review}"   # local evidence, never in git
```

Four decisions belong to the operator, and the skill asks rather than guessing:

1. **Where masters and derived files live.** Pick a root outside any git repo (a 1.2 GB
   mp4 should never be one `git add` away from a commit) and outside the downloads
   folder (browsers write there and people bulk-delete there).
2. **The naming convention.** The reference convention below keys the filename to a
   weekly series and a date. Adapt the series prefixes to your own schedule.
3. **What each variant is for.** If a smaller re-encode exists to clear an upload size
   ceiling, name the ceiling in bytes. The probe in Phase 4.2 aims at a stated target
   far better than at a percentage.
4. **Captions: sidecar, burned in, or discarded.** Zoom already ships a caption file
   with most sessions. Sidecar is the cheap default. Burned-in captions need an ffmpeg
   build with libass or libfreetype, which many stock builds lack. Check before
   promising them: `ffmpeg -filters | grep -E 'subtitles|ass|drawtext'`.

One finding outranks the rest of this skill: **a library of sessions that happened once
should exist in two places.** Long recordings accumulate quietly (one working library
measured 97 GB across 35 sessions). Space is rarely the constraint. Singularity is.
Get a second copy before optimising anything.

---

## The golden rules

1. **Never mutate a master.** The Zoom export is the only copy of a session that
   happened once. Every derived file is a new file at a new path. ffmpeg refuses a
   literal input-equals-output, but that is a path comparison, not a policy. Outputs go
   to the derived tree and never share a name with the input.
2. **Exit 0 proves nothing.** Measured three ways: `ffmpeg -n` against an existing
   output exits 0 having written nothing; no overwrite flag with stdin at `/dev/null`
   exits 0 having written nothing; a derive whose audio died exits 0 with a perfect
   duration. Probe the output (Phase 5). Never branch on `$?` alone.
3. **`-nostdin` on every ffmpeg invocation.** Without it, an overwrite prompt with
   stdin on a pipe hangs forever (measured: still blocked after 8 s on a one-second
   job), and inside a `while read` loop it eats the list: a four-item batch ran only
   items a and c, and said nothing. With `-nostdin` all four ran.
4. **A stream-copy cut keeps material past the cut.** A head cut with `-c copy` hides
   up to a whole GOP of removed material behind an mp4 edit list. Measured: `-ss 25 -i
   src -t 5 -c copy` reports 5.000 s and carries 10.011 s of packets, first PTS -5.000.
   It plays correctly and the file misrepresents its contents. Phase 5 has the check
   that catches it.
5. **Identify a file by name plus size plus duration.** Libraries grow duplicate
   basenames: masters and re-encodes sharing a name in different folders. A name alone
   identifies nothing, and a variant token in the name identifies less.
6. **Report bytes beside every count.** "Rendered the clean" is not a result.
   "982,123,456 bytes, 129m00s, 1280x720, mean -24 dB" is.
7. **Confirm every upload at the far end.** A copy finishing without an error says
   nothing about what the destination stored. Some platforms accept a file and store
   nothing. Phase 6 asks the far end directly.
8. **"Not found" means "not found here".** A missing variant for a session means no
   variant was found in the library. It never proves one was not made.
9. **Mechanics here, destinations yours.** Where the evidence stops, the skill asks.
   Guessing a destination, a naming rule or a publish target is how a private recording
   lands on a public surface.

---

## The reference variant map

One naming shape, measured on the library this skill came from. Adapt the prefixes and
keep the shape: series, date, optional part, variant.

`<series>-<YYYY-MM-DD>[-p<N>]-<variant>.<ext>`

| Variant | Dims | FPS | What it is | Typical destination |
|---|---|---|---|---|
| `full` | 1280x720 | 25 | the Zoom export, untouched | the master |
| `clean` | 1280x720 | 25 | same duration and dims, ~18% smaller | upload under a size ceiling |
| `thumb` | source dims | n/a | PNG still pulled from the video | cover art |
| `horizontal` | 1920x1080 | 30 | ~40 s clip, feed framing | feed social |
| `story` | 1080x1920 | 30 | ~40 s clip, vertical reframe | Stories, Reels, Shorts |
| `story30` | 1080x1920 | 30 | ~26 s cut of the same clip | a 30-second slot |

Two size conventions do different jobs, and telling them apart matters:

| Convention | Audio | Reduction | Job |
|---|---|---|---|
| light re-encode (the `clean` above) | untouched | ~18% | a gentler file, quality kept |
| aggressive compress folder | dropped to 64 kbps | ~67% | getting under a hard cap, audio quality spent |

The three clip bitrates in the reference library land in one narrow band across three
different framings of the same content, the signature of one fixed quality setting
applied three times. Phase 4 therefore uses a single CRF across the clip trio.

---

## Phase 1: ingest

### 1.1 A Zoom export is a set, not a file

One `GMT<YYYYMMDD>-<HHMMSS>` prefix, four members:

| Zoom file | Ingested name |
|---|---|
| `..._Recording[...]_<W>x<H>.mp4` | `<stem>-full.mp4` |
| `..._Recording[...].m4a` | `<stem>-full.m4a` |
| `..._Recording[...].cc.vtt` | `<stem>-full.vtt` |
| `..._RecordingnewChat.txt` | `<stem>-chat.txt` |

The `.vtt` matters out of proportion to its size: Zoom already made captions, free.
Rename the sidecars with the master or they scatter.

### 1.2 The three date traps, all real

**Trap A: the GMT stamp is UTC.** The first eight digits are the UTC date. Measured:
`GMT20260821-020000` is local Thursday 2026-08-20 19:00 in Pacific time, not Friday
the 21st. It bites on evening recordings, and it bites hardest on a weekend series,
because the derived date lands on the wrong weekday and any weekday-based naming rule
stops working.

```bash
STAMP=20260821-020000                                     # from GMT20260821-020000_...
EPOCH=$(date -j -u -f "%Y%m%d-%H%M%S" "$STAMP" "+%s")     # -u parses the stamp AS UTC
date -r "$EPOCH" "+%Y-%m-%d %a"                           # -> 2026-08-20 Thu
```

**Trap B: the `cutfile` timestamp is a processing time.** An export named
`GMT20260325-180126_Recording.cutfile.20260510195621583_...` carries two date-like
strings. The later one is when Zoom processed a cut, in that file 46 days after the
session. A pattern reaching for "the long number" names a March recording May. Strip
it before deriving anything.

**Trap C: the filesystem cannot answer either.** `stat` modification time on a real
master read 49 days after the date in its own name. Mtime is the download time. Never
use it for a session date, and never use it in evidence shown to a human who is
judging a split (1.4).

### 1.3 Series naming comes from the weekday, and the weekday comes from Trap A

In the reference library the rule is absolute: Saturday sessions take one prefix,
Wednesday sessions another, and a topic series takes a third. Write your own version
of that rule down before ingesting, in the planner below.

> **A recording whose weekday matches no series stops the ingest. Ask.**

This rule does real work. A weekday recording from a persistent meeting room is often
a private call: a client session, a sales call, something that was never a candidate
for a public surface. The planner refuses to name it, and that refusal is the point.

### 1.4 A part 2 is proposed, never applied

Two export sets sharing a local date and a series may be parts of one session, ordered
by GMT start time. They may equally be a dropped call restarted, a false start, or an
unrelated second meeting, and no filename can tell those apart. Print the evidence
and let a human call it. Derive start times from the stamp, never from `stat` (Trap C):

```bash
for F in "$@"; do
  S=$(sed -E 's/^GMT([0-9]{8}-[0-9]{6})_.*/\1/' <<<"$(basename "$F")")
  E=$(date -j -u -f "%Y%m%d-%H%M%S" "$S" "+%s")
  D=$(ffprobe -v error -show_entries format=duration -of csv=p=0 "$F")
  printf '%s  start %s  %.0f min  %s bytes\n' "$(basename "$F")" \
     "$(date -r "$E" '+%Y-%m-%d %a %H:%M')" "$(bc -l <<<"$D/60")" "$(stat -f %z "$F")"
done
```

A second file of a few minutes is a restart. A second file of an hour is a part 2.
Between those is a human decision.

### 1.5 The planner

Plans by default; `--apply` moves. Edit the weekday map to your own series before
first use. The planner refuses on an unmatched weekday, on a multi-session date, on
an existing destination file, and on a ` (N)` copy. `--apply` refuses entirely
without `VID_MASTERS`.

```bash
#!/bin/bash
# zoom_ingest.sh [--apply]   reads $PWD, writes nothing without --apply.
set -u
APPLY=0; [ "${1:-}" = "--apply" ] && APPLY=1
if [ "$APPLY" = 1 ]; then DEST="${VID_MASTERS:?--apply needs VID_MASTERS, Phase 0 is unanswered}"
else DEST="${VID_MASTERS:-<VID_MASTERS unset, plan only>}"; fi
T=$(mktemp -d); trap 'rm -rf "$T"' EXIT

ls -1 | grep '^GMT[0-9]\{8\}-[0-9]\{6\}_' \
  | sed -E 's/^(GMT[0-9]{8}-[0-9]{6})_.*/\1/' | sort -u > "$T/s"
while read -r P; do
  E=$(date -j -u -f "%Y%m%d-%H%M%S" "${P#GMT}" "+%s" 2>/dev/null) || { echo "UNPARSEABLE $P"; continue; }
  printf '%s\t%s\t%s\t%s\n' "$(date -r "$E" +%Y-%m-%d)" "$E" "$P" "$(date -r "$E" +%a)" >> "$T/p"
done < "$T/s"

sort -k1,1 -k2,2n "$T/p" | awk -F'\t' '{d[$1]++; L[NR]=$0} END{
  for(i=1;i<=NR;i++){split(L[i],f,"\t"); print f[1]"\t"f[3]"\t"f[4]"\t"(d[f[1]]>1?"MULTI":"")}}' > "$T/n"

while IFS=$'\t' read -r DATE P DOW MULTI; do
  case "$DOW" in Sat) SER=ws ;; Wed) SER=ama ;; *) SER=ASK ;; esac   # EDIT: your series map
  echo "-- $P  ($DOW $DATE)  ->  ${SER}-${DATE}-*"
  [ "$SER" = ASK ]   && echo "   ! $DOW matches no series. Series cannot be derived. ASK."
  [ -n "$MULTI" ]    && echo "   ! more than one session on $DATE. p1/p2 vs restart vs unrelated"
  [ -n "$MULTI" ]    && echo "     meeting cannot be decided from filenames. Run 1.4, rename by hand."
  for F in "$P"_Recording*; do
    [ -e "$F" ] || continue
    case "$F" in *"("[0-9]")".*) echo "   =  $F  (duplicate copy, dedupe first, 1.6)"; continue ;; esac
    case "$F" in
      *_[0-9]*x[0-9]*.mp4) N="${SER}-${DATE}-full.mp4" ;; *.cc.vtt) N="${SER}-${DATE}-full.vtt" ;;
      *.m4a)               N="${SER}-${DATE}-full.m4a" ;; *newChat.txt) N="${SER}-${DATE}-chat.txt" ;;
      *) echo "   ?  $F  (unmapped, left alone)"; continue ;;
    esac
    if [ "$APPLY" = 1 ] && [ "$SER" != ASK ] && [ -z "$MULTI" ]; then
      mkdir -p "$DEST"
      [ -e "$DEST/$N" ] && { echo "   !! $DEST/$N exists, REFUSING"; continue; }
      mv -n "$F" "$DEST/$N" && echo "   -> $DEST/$N"
    else echo "   -> $N"; fi
  done
done < "$T/n"
```

### 1.6 Duplicates: size first, then hash, never by pattern

A ` (N)` suffix is a browser refusing to overwrite, which makes it a candidate marker
and never a verdict. Measured in a real library: files carrying the suffix pointed in
three directions at once. Some twins were byte-identical, some differed in size by
140x, and some had no sibling in their own directory. Deleting by the pattern can
delete the only copy of something.

Group by exact size (free, and necessary), hash only inside the collision groups:

```bash
find "$ROOT" -type f \( -iname '*.mp4' -o -iname '*.mov' -o -iname '*.m4v' \) \
  -exec stat -f '%z%t%N' {} + > all.tsv
awk -F'\t' 'NR==FNR{c[$1]++;next} c[$1]>1' all.tsv all.tsv | cut -f2 \
  | tr '\n' '\0' | xargs -0 shasum -a 256 | sort \
  | awk '{h=substr($0,1,64); p=substr($0,67);
          if(h==ph){if(!s){printf "\nIDENTICAL BYTES\n  %s\n",pp; s=1} printf "  %s\n",p} else s=0
          ph=h; pp=p}'
```

Identifying is not deleting. The command prints groups; it never picks a survivor.
When a human does act, send files to the system Trash through the Trash API (on macOS,
`/usr/bin/trash`), which renames on collision and records Put Back. Never move files
into the Trash folder by hand: a plain move clobbers a same-named file silently, and
duplicate basenames are exactly what this library grows.

---

## Phase 2: the master

```bash
chmod 444 "$VID_MASTERS"/*-full.mp4                       # blocks ffmpeg overwrite
for F in "$VID_MASTERS"/*-full.mp4; do                    # identity, recorded once
  printf '%s\t%s\t%s\t%s\n' "$(basename "$F")" "$(stat -f %z "$F")" \
    "$(ffprobe -v error -show_entries format=duration -of csv=p=0 "$F")" \
    "$(cat <(head -c 4194304 "$F") <(tail -c 4194304 "$F") | shasum -a 256 | cut -c1-16)"
done | tee "$VID_MASTERS/_masters.tsv"
```

On macOS, `chflags uchg` additionally blocks moves and deletes; `chmod` alone does
not. A Zoom master carries no origin timestamp: `format_tags=creation_time` measures
empty on real exports. Identity is therefore name plus bytes plus duration plus the
head/tail fingerprint above. The fingerprint is blind to the middle of the file, which
is acceptable only because bytes and duration sit beside it.

---

## Phase 3: review. Hard human gate, never auto-passes

### Why this gate exists

Persistent meeting rooms keep recording. A booked 45-minute session has been measured
capturing 231 minutes, because the room was open before the booking and stayed open
after it, and recording runs while the room is open. A capture like that can hold a
different person's private session in its tail.

Any workflow that records recurring sessions in one persistent room carries the same
shape of risk, in the head as well as the tail. Whether your recordings carry it is a
question nobody can answer without looking, and this gate is the looking. It is a
check to run, not an incident to respond to. Anything it turns up is a question until
a human answers it, and the correct verdict on an unreviewed file is `NOT-REVIEWED`,
never "probably fine".

Two questions, not one. **Gate A, the head and the tail:** does the recording contain
material from outside the advertised session? **Gate B, the surface:** does it
anywhere show a CRM screen, an inbox, an admin panel, a bookmarks bar, or a name plate
that exposes someone who did not agree to be exposed? Gate B applies to the whole
file, not just the ends.

### The trap: you cannot find the seam by listening for silence

Measured on a real 51-minute master: zero silence runs of 30 s or more at a -35 dB
floor, across the whole file. Zoom audio carries continuous comfort noise and room
tone, so an empty room that is still recording is not silent. A gate built on
`silencedetect` returns clean on a file that ran nine minutes past the session.
The seam is a video event: tiles vanish, the screen share ends, the layout collapses.
Silence detection stays in the gate as a corroborator that tells a human which minutes
of a tail contain speech. It is never the locator.

### 3.1 The sweep, the primary instrument

```bash
F="$VID_MASTERS/ws-2026-07-25-full.mp4"; D="$VID_REVIEW/$(basename "$F")"; mkdir -p "$D"
DUR=$(ffprobe -v error -show_entries format=duration -of csv=p=0 "$F")

# max keyframe gap, measured, never assumed. Demux only, no decode.
KFMAX=$(ffprobe -v error -select_streams v:0 -show_entries packet=pts_time,flags -of csv=p=0 "$F" \
  | awk -F, '$2~/K/{t=$1; if(n++){g=t-p; if(g>mx)mx=g} p=t} END{printf "%.2f", mx+0}')
[ "$KFMAX" = "0.00" ] && { echo "ABORT: keyframe probe failed on $F"; exit 1; }   # fail CLOSED
IV=30; awk -v k="$KFMAX" -v i="$IV" 'BEGIN{exit !(k>i)}' && { SKIP=""; IV=$(printf '%.0f' "$KFMAX");
  echo "NOTE: keyframe gap ${KFMAX}s, full decode, interval ${IV}s"; } || SKIP="-skip_frame nokey"

ffmpeg -nostdin -hide_banner -loglevel error $SKIP -i "$F" -an \
  -vf "fps=1/$IV,scale=200:-2,tile=6x10:margin=4:padding=2:color=#202020" \
  -q:v 4 "$D/sweep_%02d.jpg"

# the tile timeline, as data. Sheets are 1-BASED: image2 writes sweep_01.jpg first.
awk -v iv="$IV" -v dur="$DUR" 'BEGIN{ for(i=0;i*iv<=dur;i++){s=int(i/60)+1;t=i%60;
  printf "sweep_%02d.jpg,row%d,col%d,%.0f\n", s, int(t/6)+1, (t%6)+1, i*iv }}' > "$D/tile_index.csv"
```

Cost, measured on a 51-minute, 409 MB master: 1.52 s, 2 sheets, 436 KB, roughly
1.8 seconds of compute per source hour. The same sweep at full decode takes 7.46 s.

Two things about `-skip_frame nokey` go silently wrong:

- (a) The sample interval must be greater than or equal to the max keyframe gap.
  Measured on a 10 s-GOP file sampled every 5 s: 4 tiles emitted, 2 distinct.
  Duplicates read to a reviewer as "nothing changed here".
- (b) The sheets differ slightly from a full decode. The `fps` filter holds the most
  recent input frame, and with this flag input frames exist only at keyframes, so a
  tile labelled t shows content from [t minus keyframe_gap, t], never later than t.
  The error is one-directional, and it matters at cut time (3.4).

### 3.2 Corroborators (one decode, ~1.1 s per source hour)

```bash
ffmpeg -nostdin -hide_banner -nostats -i "$F" -vn \
  -af "volumedetect,silencedetect=noise=-40dB:d=10" -f null - 2>&1 \
  | grep -E 'mean_volume|max_volume|silence_' > "$D/audio.txt"
ffmpeg -nostdin -hide_banner -loglevel error -i "$F" -vn \
  -filter_complex "aformat=channel_layouts=mono,showwavespic=s=1800x240:colors=#39FF14:scale=sqrt,format=rgb24" \
  -frames:v 1 "$D/wave.png"
tail -60 "$VID_MASTERS/ws-2026-07-25-full.vtt" 2>/dev/null   # Zoom already transcribed the tail
```

Warning: `-v error` swallows every filter that reports through the log:
`volumedetect`, `silencedetect`, `blackdetect`, `metadata=print`. Measured:
`-v error` with `volumedetect` prints nothing, so a grep for `max_volume` comes back
empty and a script reads "no reading" as "nothing to hear". Use `-hide_banner
-nostats`, never `-v error`, on any of them.

### 3.3 Read the sheets

An agent can run 3.1 and 3.2 and read the JPEGs. A human owns the verdict. Windows:

- **Tail: everything past the advertised end, minus two minutes.** The window is
  derived, never chosen: a measured bleed ran 186 minutes past a 45-minute booking,
  so any round number has already been beaten. The minus two minutes is the
  load-bearing part. A window starting exactly at the advertised end shows a tail
  with no reference point, and you cannot tell a bled tail from a session that
  genuinely ran long. You have to see the seam to know it is one. If the advertised
  length is recorded nowhere, the window is the whole file. That costs 1.8 s per
  source hour. Do not invent a session length to shrink it.
- **Head: the first ten minutes.** Reasoned from the mechanism: the proven failure is
  the tail, and the same room is open before the session too. Recording starts when
  the host presses record.

A finding may not describe anything the sheet cannot resolve. At 200 px a tile
carries layout: webcam grid, screen share, application chrome, a caption band, tile
count. It does not carry text. A name plate is illegible at sheet scale, and a row in
a scrolled admin list is smaller than a name plate. Any note naming a person, a
company, an account, or a panel's contents is invalid unless a full-resolution pull
backs it and cites its timestamp:

```bash
ffmpeg -nostdin -hide_banner -loglevel error -ss 12870 -i "$F" -frames:v 1 -update 1 -q:v 2 "$D/f_12870.jpg"
```

Measured: 0.08 s, full 1280x720. Input seek (`-ss` before `-i`) is frame-accurate
when the frame is re-encoded, which a JPEG pull is.

The Gate B judgment call: a screen the audience was meant to see (a site being
reviewed live, a demo the presenter queued up) is consented spotlight. An inbox, an
account screen, a CRM contact list, an admin panel, a second monitor, a notification
toast, or a standing bookmarks bar is incidental back-office: never consented, never
intended.

Residual, and it belongs in the verdict: a 30-second sample cannot miss a screen
share or a participant who stayed a minute, because those persist. It can miss a
notification toast, a briefly revealed password field, or a single alt-tab.

### 3.4 The cut, if there is one

If the gate finds material outside the session, the fix is a trim, and the trim
follows the table in Phase 4.6. Two extra rules here:

- Prefer a re-encode trim at the seam over a stream copy. A copy trim near a seam
  keeps up to one GOP of the removed material behind the edit list (Golden Rule 4),
  which at a privacy seam is the whole problem.
- After trimming, rerun the sweep on the derived file and re-verify (Phase 5) before
  the verdict changes. A cut asserted is not a cut confirmed.

### 3.5 The verdict needs a carrier at the moment of upload

A verdict written in a chat scrolls away. Give it a carrier the publish step can
check: a row in the Phase 7 index with a state (`CLEAN <date> <initials>` or
`NOT-REVIEWED`), plus optionally a "ready" folder or hardlink lane that only reviewed
files enter. `NOT-REVIEWED` blocks publication. A blank entry is not a pass.

Two source-level fixes retire much of this gate permanently, and they belong to the
operator: turn off live captions in the recording room if name plates and caption
bands are a recurring Gate B cost, and decide the name-plate policy as policy.
Zoom burns participant names into gallery and active-speaker recordings, so attendees
of a recorded session are named in it whether or not they agreed to that when they
showed up. The gate can flag it. The gate cannot decide it.

---

## Phase 4: derive. Mechanical, once Phase 3 has passed

### 4.1 The invocation template. Every preset obeys this shape

```bash
IN="$VID_MASTERS/ws-2026-07-25-full.mp4"
BASE=$(basename "$IN" -full.mp4)            # series/date/part never change
OUT="$VID_DERIVED/$BASE/${BASE}-clean.mp4"; TMP="${OUT%.mp4}.part.mp4"
mkdir -p "$(dirname "$OUT")"; rm -f "$TMP"
[ -e "$OUT" ] && { echo "REFUSING: $OUT exists"; exit 1; }

ffmpeg -nostdin -hide_banner -y \
  -i "$IN" <preset flags> \
  -map_metadata 0 -movflags +faststart "$TMP" \
  && mv -n "$TMP" "$OUT" || { rm -f "$TMP"; exit 1; }
```

- **`-nostdin` always** (Golden Rule 3).
- **`-y` against a temp path that can never be a master, never against the target.**
  ffmpeg truncates its output the instant it opens it, so a killed run over an
  existing file leaves a 48-byte stub where a file was. The collision guard is the
  `[ -e "$OUT" ]` test on a path you control. Bare `-n` is not a guard either: it
  exits 0 having written nothing, so the previous run's file gets handed to Phase 6.
- **`rm -f "$TMP"` first**, or a stale `.part` from a crashed run gets promoted.
- **`-map_metadata 0`** is not the default. Measured: without it, the source's
  `creation_time` is silently dropped. Zoom masters carry none, so on those it is a
  no-op that stays correct if a source ever does carry one.
- **`-movflags +faststart`** on anything streamed or uploaded. Measured: atom order
  goes from `ftyp free mdat moov` to `ftyp moov free mdat`. Without it, a viewer
  waits for the whole file to arrive before playback starts.
- **One input, one output, per run.** Batch by looping, never by multi-output.

### 4.2 `clean`: a size-targeted re-encode, probed, never guessed

The reference `clean` measures: same duration to the second, same 1280x720, same
25 fps, 18% smaller. That rules out a trim (duration would change), a remux (size
moves well under 1%), an audio-only change (audio is a tenth of the file) and a
resize. What remains is a video re-encode below the source's own quality target.

There is no default CRF and no expected range. Re-encoding already-lossy video at a
low CRF makes the file bigger, and how much bigger depends on how hard the source was
squeezed. Source files in the reference library span 700 to 1150 kbps, enough spread
that two masters will not want the same number. Probe:

Two arithmetic traps, both measured. (a) The denominator is the master's own video
bytes, known exactly and for free. Never estimate from a sampled window, which
drifts. (b) The target is a whole-file number while the probe measures video only,
and `-c:a copy` passes audio through unchanged. On a real master, audio plus
container is 9.7% of the file, so an 18% whole-file cut needs a 19.9% cut on video.
Mix those up and the encode lands about a third short of the target.

```bash
#!/usr/bin/env bash
# clean-probe <master> <crf> [whole-file target %]
set -euo pipefail
SRC="$1"; CRF="$2"; TARGET="${3:-18}"
DUR=$(ffprobe -v error -show_entries format=duration -of csv=p=0 "$SRC")
TOT=$(ffprobe -v error -show_entries format=size -of csv=p=0 "$SRC")
VB=$(ffprobe -v error -select_streams v:0 -show_entries packet=size -of csv=p=0 "$SRC" | awk '{s+=$1} END{print s}')
WANT=$(awk -v v="$VB" -v t="$TOT" -v p="$TARGET" 'BEGIN{printf "%.0f", v - t*p/100}')
T=$(mktemp -d); trap 'rm -rf "$T"' EXIT; tot=0
for i in 1 2 3 4 5; do
  SS=$(awk -v d="$DUR" -v i="$i" 'BEGIN{printf "%.3f", (d-60)*(i-0.5)/5}')
  ffmpeg -nostdin -v error -y -ss "$SS" -t 60 -i "$SRC" -map 0:v:0 \
     -c:v libx264 -preset slow -crf "$CRF" -pix_fmt yuv420p "$T/o.mp4"
  OD=$(ffprobe -v error -show_entries format=duration -of csv=p=0 "$T/o.mp4")
  tot=$(awk -v a="$tot" -v b="$(stat -f %z "$T/o.mp4")" -v d="$OD" 'BEGIN{print a+b/d}')
done
awk -v c="$CRF" -v e="$tot" -v v="$VB" -v w="$WANT" -v d="$DUR" -v p="$TARGET" 'BEGIN{
  est=e/5*d
  printf "CRF %s: projected video %d bytes (%+.1f%%)\n", c, est, (est-v)/v*100
  dd=(est-w)/w*100
  if (dd>3)       printf "  %.1f%% too big, try CRF %d\n", dd, c+1
  else if (dd<-3) printf "  %.1f%% too small, try CRF %d\n", -dd, c-1
  else            printf "  within 3%%, use CRF %s\n", c }'
```

Read the projection, run the encode, and confirm the whole-file result in Phase 5
rather than trusting the projection. If the operator named a size ceiling in Phase 0,
aim `WANT` at the ceiling in bytes instead of the percentage.

Then the encode, with the probed CRF substituted:

```bash
<template> -map 0:v:0 -map "0:a:0?" -c:v libx264 -preset slow -crf "$CRF" -pix_fmt yuv420p -c:a copy
```

`-c:a copy` is deliberate: audio is a tenth of the file, so re-encoding it saves
almost nothing and costs a generation of loss. Do not add `-r`: source timing is
what it is, and re-timing a two-hour file is how it goes out of sync.

For the aggressive variant (getting under a hard cap, the ~67% convention), keep the
same probe discipline and let the audio carry part of the cut: re-encode audio to
AAC at 64 kbps alongside a higher video CRF. Verify loudness in Phase 5 either way.

### 4.3 `thumb`

```bash
# 12 candidates on one sheet, driven by the real packet count (never an assumed fps)
N=$(ffprobe -v error -select_streams v:0 -count_packets -show_entries stream=nb_read_packets -of csv=p=0 "$IN")
ffmpeg -nostdin -hide_banner -loglevel error -y -i "$IN" \
  -vf "select='not(mod(n\,$((N/12))))',scale=320:-2,tile=4x3" -frames:v 1 -update 1 contact.png
# then grab the one you picked
ffmpeg -nostdin -hide_banner -loglevel error -y -ss 00:12:40 -i "$IN" -frames:v 1 -update 1 \
  "$VID_DERIVED/$BASE/${BASE}-thumb.png"
```

`-update 1` suppresses a warning about a missing `%03d` pattern. The output is
byte-identical either way; include it so the warning never trains anyone to ignore
warnings.

Gate a thumb on brightness, not on file size. A genuinely black 1280x720 PNG is
~4.3 KB, but a dim or slide-heavy screen-share frame lands well under 100 KB and is
perfectly good:

```bash
ffmpeg -nostdin -hide_banner -nostats -i thumb.png \
  -vf "signalstats,metadata=print:key=lavfi.signalstats.YAVG" -f null - 2>&1 \
  | sed -n 's/.*YAVG=\(.*\)/\1/p' | head -1        # black reads 16; a real frame read 125.9. Reject under 20.
```

### 4.4 `horizontal`: normalise to 1080p30

```bash
<template> \
  -map 0:v:0 -map "0:a:0?" \
  -vf "scale=1920:1080:force_original_aspect_ratio=decrease:force_divisible_by=2:flags=lanczos,\
pad=1920:1080:(ow-iw)/2:(oh-ih)/2:color=black,setsar=1,format=yuv420p" \
  -r 30 -c:v libx264 -preset slow -crf 20 -profile:v high \
  -c:a aac -b:a 192k -ar 48000 -ac 2
```

Verified on five source shapes (1920x1080, 1280x720, 1366x768, 1001x563, 2560x1440);
all five landed at exactly 1920x1080. `force_original_aspect_ratio=decrease` fits
without distorting; `force_divisible_by=2` keeps the even dimensions h264 with
yuv420p requires; `setsar=1` forces square pixels so the displayed shape matches the
stored one.

Warning: **`-r 30` goes after `-i`, never before.** As an input option, `-r`
re-stamps the input's timing instead of setting an output rate. Measured on a 25 fps
source: `-r 30` before `-i` produced video of 25.000 s against audio of 30.000 s,
five seconds of desync on a thirty-second file, silently. On a 129-minute master
that is twenty-one minutes.

Warning: **drop `-level`.** Asserting `-level 4.0` makes x264 quietly drop a
reference frame to fit the DPB limit, and where the violated constraint is macroblock
rate instead, x264 prints one warning and ships a file declaring conformance to a
level it violates. x264 derives the correct level on its own.

Audio is re-encoded here, unlike `clean`: a clip source may be mono, 44.1 kHz, or
PCM. If the source carries two audio tracks (the OBS and Loom default), `-map
"0:a:0?"` takes one and drops the other silently. Mix first with
`-filter_complex "[0:a:0][0:a:1]amix=inputs=2:duration=longest[a]"` rather than
halving it.

### 4.5 `story` and `story30`: a reframe, not a resize

16:9 into 9:16 is a different shape. `scale=1080:1920` squashes the picture.
Scale-with-padding letterboxes it. Both produce a perfect-looking wrong file.

```bash
CW="round(min(iw\,ih*9/16)/2)*2"; CH="round(min(ih\,iw*16/9)/2)*2"
<template> \
  -map 0:v:0 -map "0:a:0?" \
  -vf "crop='$CW':'$CH':'$XOFF':0,scale=1080:1920:flags=lanczos,setsar=1,format=yuv420p" \
  -r 30 -c:v libx264 -preset slow -crf 20 -profile:v high \
  -c:a aac -b:a 192k -ar 48000 -ac 2
```

Verified on five source shapes, all landing 1080x1920 with DAR 9:16. `round(x/2)*2`
snaps to the nearest even number. Use `round`, not `trunc`: on a 1920x1080 source,
`trunc` gives 606 where `round` gives 608, tripling the aspect deviation for nothing.
Crop before scale, so the scaler handles 608x1080 rather than 1920x1080.

**`$XOFF` is the one number in this skill that cannot be computed.** ffmpeg defaults
to centre, and on a screen recording or a two-up Zoom frame the subject is rarely
centred. Preview three candidate crops, then pin the offset explicitly and keep it
identical between `story` and `story30`:

```bash
ffmpeg -nostdin -hide_banner -loglevel error -y -ss 00:00:10 -i "$IN" -frames:v 1 -update 1 \
 -filter_complex "[0:v]split=3[l][c][r];\
[l]crop='$CW':ih:0:0,scale=-2:480[l1];\
[c]crop='$CW':ih:'(in_w-out_w)/2':0,scale=-2:480[c1];\
[r]crop='$CW':ih:'in_w-out_w':0,scale=-2:480[r1];\
[l1][c1][r1]hstack=3,format=rgb24[v]" -map "[v]" crop_preview.png
```

Verified on 1920x1080, 1280x720 and 1366x768: 810x480, three panels, no hardcoded
widths.

When the content cannot survive a crop (a wide slide, text at both edges), put the
whole frame on a blurred enlargement of itself. Verified: 1080x1920, DAR 9:16, audio
present.

```bash
ffmpeg -nostdin -hide_banner -loglevel error -i "$IN" \
 -filter_complex "[0:v]split=2[bg][fg];\
[bg]scale=1080:1920:force_original_aspect_ratio=increase,crop=1080:1920,gblur=sigma=24,eq=brightness=-0.08[bgb];\
[fg]scale=1080:-2:flags=lanczos[fgs];\
[bgb][fgs]overlay=(W-w)/2:(H-h)/2:shortest=1,setsar=1,format=yuv420p[v]" \
 -map "[v]" -map "0:a:0?" -r 30 -c:v libx264 -preset slow -crf 20 \
 -c:a aac -b:a 192k -ar 48000 -ac 2 -map_metadata 0 -movflags +faststart "$TMP"
```

Warning: with `-filter_complex`, map only the graph's outputs plus streams that
bypass it. A stray `-map 0:v:0` alongside a graph adds the unfiltered stream and
usually wins position zero, producing a 1920x1080 file from a command whose graph
output 1080x1920, with no error.

**`story30` comes from the master, not from `story`.** The reason is staleness:
change the crop offset and any cut taken from the older `story` is silently wrong.
On a 40-second clip, re-rendering from the master costs seconds, so there is nothing
to trade.

### 4.6 Trimming, in one table

**Use `-ss <start>` before `-i`, and `-t <duration>`.**

- **Never `-to` with `-ss`.** After an input seek the output timeline starts at
  zero, so `-to` behaves as a duration while reading like an end timestamp.
  Measured: a requested 5 to 15 window produced 15.011 s (copy) and 15.000 s
  (re-encode) when 10 s was asked for.
- `-ss` before `-i` is fast and frame-accurate when the output is re-encoded
  (measured: first PTS 0.000, first frame a keyframe). After `-i` it decodes every
  frame from zero for the same result.
- A `-c copy` trim is fast and lossy in a different way: it can only cut at
  keyframes, it overshoots by up to one GOP, and the overshoot hides behind the
  edit list (Golden Rule 4). Use copy trims for rough cuts only, never at a
  privacy seam, and always run the Phase 5 carries-removed-material check on the
  result.

---

## Phase 5: verify. Mechanical, safe to automate

Every failure measured while building this skill produced a file that opened and
played, with a silent ffmpeg. Verification is a probe, not a glance.

```bash
#!/bin/bash
# verify_derive.sh <file> <WIDTHxHEIGHT> <expected-seconds> [tolerance]
set -u
F="$1"; WANT_DIM="$2"; WANT_SEC="$3"; TOL="${4:-0.1}"; FAIL=0
p(){ ffprobe -v error "$@" "$F"; }
BYTES=$(stat -f %z "$F")
DUR=$(p -show_entries format=duration -of csv=p=0)
IGN=$(ffprobe -v error -ignore_editlist 1 -show_entries format=duration -of csv=p=0 "$F")
DIM=$(p -select_streams v:0 -show_entries stream=width,height -of csv=s=x:p=0)
AIDX=$(p -select_streams a -show_entries stream=index -of csv=p=0 | tr '\n' ',')

echo "$F, $BYTES bytes | dur ${DUR:-none} (want $WANT_SEC +/-$TOL) | ${DIM:-none} (want $WANT_DIM) | audio [${AIDX%,}]"
[ -n "${DUR:-}" ] && awk -v d="$DUR" -v w="$WANT_SEC" -v t="$TOL" 'BEGIN{x=d-w;if(x<0)x=-x;exit !(x<=t)}' \
  || { echo "  FAIL duration off by more than $TOL s"; FAIL=1; }
[ "$DIM" = "$WANT_DIM" ] || { echo "  FAIL dimensions are not exactly $WANT_DIM"; FAIL=1; }
# CARRIES-REMOVED-MATERIAL check. Baselines measured: real master 0.000 s,
# re-encode 0.080 s, tail copy-trim 0.022 s, head copy-trim 5.011 s.
awk -v h="$DUR" -v i="$IGN" 'BEGIN{d=i-h;if(d<0)d=-d;exit !(d<=0.5)}' \
  || { echo "  FAIL CARRIES REMOVED MATERIAL: ${IGN}s of packets behind a ${DUR}s edit list"; FAIL=1; }
if [ -z "${AIDX%,}" ]; then echo "  FAIL NO AUDIO STREAM"; FAIL=1; else
  # one decode, both numbers. NOT -v error: these filters log at info level.
  V=$(ffmpeg -nostdin -hide_banner -nostats -i "$F" -vn -af "volumedetect,silencedetect=noise=-50dB:d=5" -f null - 2>&1)
  MEAN=$(sed -n 's/.*mean_volume: \(.*\) dB/\1/p' <<<"$V"); MAX=$(sed -n 's/.*max_volume: \(.*\) dB/\1/p' <<<"$V")
  SIL=$(sed -n 's/.*silence_duration: \([0-9.]*\).*/\1/p' <<<"$V" | awk '{s+=$1} END{printf "%.1f", s+0}')
  echo "  loudness mean ${MEAN:-?} / max ${MAX:-?} dB, ${SIL}s silent of ${DUR}s"
  [ -z "${MEAN:-}" ] && { echo "  FAIL volumedetect returned nothing"; FAIL=1; }
  awk -v m="${MEAN:--99}" 'BEGIN{exit !(m > -45)}' || { echo "  FAIL audio is mostly dead (mean ${MEAN} dB)"; FAIL=1; }
  awk -v s="$SIL" -v d="$DUR" 'BEGIN{exit !(s/d < 0.5)}' || { echo "  FAIL over half the file is silent"; FAIL=1; }
fi
[ "$FAIL" = 0 ] && echo "  PASS" || echo "  FAIL"; exit $FAIL
```

**Why mean and not max.** `max_volume` is one number over the whole file, so any real
recording satisfies it no matter what happened to the rest. Measured on a real
workshop master: mean -22.8 dB, max -0.8 dB. Digital silence reads -91.0 dB. A derive
whose audio dies partway keeps a healthy max and drops its mean, which is the failure
mode a long unattended recording has. The -45 dB mean and the 50% silence
fraction are starting thresholds: calibrate them against your own files before
treating a failure as a fact.

**Tolerances.** Dimensions are exact or fail; there is no tolerance on a resize.
Duration: +/-0.05 s for a re-encode (measured exactly 10.000, 5.000, 4.000 and
120.000 s every time) or for a full-length `clean`; up to one keyframe interval of
overshoot for a `-c copy` trim.

---

## Phase 6: publish. Human decision, mechanical confirmation

Publishing itself is the operator's call, every time: which variant goes to which
surface, at what spec, is a decision this skill will not make. What the skill owns is
the confirmation afterwards, in two separate questions.

**Did something land?** Any non-zero size at the far end, plus a duration probe of the
served asset where the platform exposes one. This is the only byte question answerable
on a platform that transcodes: community platforms and video hosts re-encode, so a
served asset is never byte-identical to the local file, and a byte-equality check
would cry wolf on every successful upload and be ignored on the one that matters.

```bash
LB=$(stat -f %z "$LOCAL"); echo "local: $LB bytes"
RB=$(curl -sIL --max-time 60 "$ASSET_URL" \
  | awk '/^HTTP\//{v=""} {if (tolower($0) ~ /^content-length:/) v=$2} END{gsub(/\r/,"",v); print (v==""?"NONE":v+0)}')
case "$RB" in
  NONE) echo "WARN destination exposes no size, record UNCONFIRMED, not uploaded" ;;
  0|"") echo "FAIL zero bytes at the far end, this is the silent no-op" ;;
  *)    echo "landed: $RB bytes" ;;
esac
```

Two things in that one-liner. `IGNORECASE` is a gawk extension and macOS runs BSD
awk, where an `IGNORECASE` matcher silently fails on a `Content-Length:` header and
reports 0 bytes, which reads as a failed upload that succeeded. And resetting `v` on
every `^HTTP/` line is what stops a redirect's own `Content-Length` being reported
as the asset's size.

**Is it byte-identical?** Only claimable where the destination stores the file
verbatim. Where neither question is answerable, write "unconfirmed" in the record.
"Uploaded" is a claim about the far end, and it holds only when the far end has been
asked.

Storage ceilings deserve one line of their own: a platform near its storage cap can
accept an upload and store nothing, with no error anywhere. That failure is invisible
from the sending side. The far-end check above is the only thing that catches it.

---

## Phase 7: record it. The last step of every run

One row per recording, not per file. That is what makes the past cheap to absorb: a
library of 223 files resolved to 35 date-keyed stems in the reference library, and
35 rows is an afternoon that moves nothing.

Keep the index somewhere backed up, beside your other registries. A spreadsheet or a
markdown table both work.

| stem | date | title | master | dur | review | derived | published |
|---|---|---|---|---|---|---|---|
| `ws-2026-07-25` | 2026-07-25 | (session title) | LOCAL 1203 MB | 2:09:00 | CLEAN 2026-08-22 ab | clean, thumb | community 2026-07-26 |
| `topic-clip` | 2026-08-14 | (clip title) | LOCAL 96 MB | 0:00:40 | NOT-REVIEWED | horizontal, story, story30 | (blank) |

`review` carries the Phase 3 verdict with a date and initials. A blank cell is not a
pass, and `NOT-REVIEWED` blocks publication. `derived` should carry the exact command
that produced each variant, not just a token: a derived file is only cheap to lose
once its recipe is written down.

Backfill is opportunistic. When a recording is touched for any reason, its row gets
written. A scheduled backfill session will not happen; a row written during real work
will.

---

## Decisions this skill leaves to you

A skill that guesses these is worse than no skill, because it will be followed.

1. **What is your `clean` for?** If it clears a size ceiling at some destination, say
   the ceiling in bytes and the Phase 4.2 probe aims at that. If it was only "a
   re-encode looked better", drop the percentage and target quality.
2. **Where do masters, derived files and the publish-ready lane live?** One variable
   each, and every `--apply` stays blocked until they are set.
3. **Captions: sidecar, burned in, or discarded?** Sidecar is free, Zoom ships the
   file. Burned-in needs an ffmpeg build that may not be the one installed.
4. **Which destination takes which variant, at what spec?** Learn it from one look at
   each surface's upload dialog. Do not let the skill presume it.
5. **Does the clip trio become routine?** In the reference library it began as one
   experiment on one topic. Treat yours the same way until the posts prove it out.
6. **What marks a reviewed file?** An index row state is the minimum. A ready lane
   (a folder or hardlink tree that only reviewed files enter) fails safer, needs no
   rename, and never mutates the identifier the index keys on.
7. **What is each recurring session's advertised length?** Record it, and Phase 3 can
   flag overruns automatically. Without it the review window is the whole file,
   which is affordable at 1.8 s of compute per source hour.
8. **Are live captions on in your recording room, and do recordings carry attendee
   name plates?** Turning live captions off at the source retires half of Gate B
   permanently. The name-plate question is policy, not review: attendees are named
   in the recording whether or not they agreed to that when they showed up.
9. **The second copy.** A cloud archive tier before anything else. Local space
   usually lasts years; a single-copy library is one disk failure from gone.

---

## Anti-patterns, each one measured breaking something

- Branching on ffmpeg's exit code. It exits 0 on empty outputs, skipped overwrites
  and dead audio. Probe the output.
- Running ffmpeg without `-nostdin` in a script. It hangs on prompts and eats
  `while read` lists.
- Deleting duplicates by the ` (N)` filename pattern. Twins differ; one of them can
  be the only copy.
- Moving files into the Trash folder by hand. A plain move clobbers same-named files
  silently. Use the Trash API.
- Dating a session by file modification time. Mtime is download time.
- Dating a Zoom export by the long `cutfile` number. That is a processing time,
  weeks later than the session.
- Locating a recording seam with silence detection. Room tone means an empty,
  still-recording room is not silent.
- Sampling a contact sheet at an interval below the max keyframe gap. Duplicate
  tiles read as "nothing changed".
- Passing `-v error` to a log-reporting filter. You get silence and read it as a
  passing result.
- Putting `-r` before `-i`. It re-stamps input timing and desyncs audio from video,
  silently, at scale.
- Using `-to` after an input `-ss`. It behaves as a duration. A 10 s ask returns 15 s.
- Squashing 16:9 into 9:16 with a plain scale. The file is the right size and the
  picture is wrong.
- Adding a stray `-map 0:v:0` beside a `-filter_complex` graph. The unfiltered
  stream wins position zero and the graph output is ignored.
- Asserting `-level` on x264 output. x264 clamps silently or ships a file that
  violates the level it declares.
- Deriving a short cut from another derived file. Change the crop or the seam once
  and every downstream cut is silently stale. Derive from the master.
- Calling an upload done because no error appeared. Ask the far end.


---
Free to use and share.

---

Maintained by Katie Van Nes of Most Referred Expert. The Lofty setups get taught live and free
every week at [LoftyMasterClass.com](https://LoftyMasterClass.com), and the recordings live in the
Most Referred Expert Community.
