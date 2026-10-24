---
name: creatorize
description: Produce long-form faceless YouTube episodes for the g russ channel. Use when asked to plan, script, source, generate assets for, or render a long-form video, documentary, explainer or tutorial. Chains REA (reverse engineering), Scenario (asset generation) and Hypit (authoring and render).
---

# Creatorize

A long-form faceless YouTube production line. Three tools, one division of labour:

| Tool | Job | How it is used |
| --- | --- | --- |
| **REA** | Read the reference | Reverse-engineers the channel's pages, its structure, or a shipped app's behaviour, and returns evidence |
| **Scenario** | Make the material | Generates the stills that prove a claim, and the thumbnail |
| **Hypit** | Author and render | Holds the script, the word-anchored timeline, the captions, the B-roll and the film |

Creatorize itself is the orchestrator and the part that knows the channel: its
section templates, its runtime band, its title grammar.

## The one idea that makes long-form affordable

Hypit is **word-anchored**. The Script holds the words; WhisperX projects those
words onto the audio; every other thing - B-roll, captions, motion graphics -
attaches to a *word*, not to a second.

That is why a 25-minute episode is not 25 minutes of work. You can rewrite a
line in section seven and the picture that answered it moves with the words.
Nothing is re-timed by hand.

Keep this in mind for every decision: **if a visual is attached to a timestamp
rather than to a word, it is wrong.** Attach it to a Moment (`@{claim!}`) or a
Selection (`@{proof}...@{/proof}`) in the Script instead.

## The pipeline

```bash
creatorize doctor                      # every dependency and credential
creatorize analyze                     # reverse-engineer the channel into a format spec
creatorize plan "<topic>"              # an episode plan on the channel's own spine
#   write the script, then:
creatorize author --plan <plan> --script <script> --project <dir>
creatorize narrate --project <dir>     # voice, local TTS, or a silent timing track
creatorize assets --project <dir>      # Scenario B-roll and thumbnail
creatorize build --project <dir>       # check, build, export
```

On a machine with less than about 8 GB, split the last step:

```bash
creatorize align --project <dir>                    # transcribe only
hypit programs down --endpoint whisperx.local       # free the memory
creatorize build --project <dir> --reuse <build-id> # render the picture
```

That split is also the faster loop when you are only changing the picture. The
narration and its word timing are the expensive part; the picture is cheap.

### 1. Analyze

`creatorize analyze` reads the channel page and writes
`config/channels/g-russ.json`: the video catalogue with runtimes, the title
formulas, the topic clusters, the long-form band, and the dominant style.

Add `--web` to also run REA's passive browser pass over the channel and a video
page. REA attaches to a headless Chrome you own; it never clicks, navigates or
mutates anything. It returns the DOM and accessibility structure, the SPA's
route and endpoint graph, and the scripts the page loads.

`creatorize chapters <videoId>` reads a video's own chapter list. A creator who
titles their sections has already decided the spine - that is the most useful
single signal in the whole format.

### 2. Plan

`creatorize plan` picks a template and fills it for one topic.

Two templates exist, and both are **reconstructed from the channel's own
published chapter lists**, not invented:

- `tutorial` - the tool walkthrough spine (10 sections, ~17-24 min)
- `documentary` - the faceless documentary spine (25 sections, ~34 min)

Each section gets a purpose, a target runtime, a target word count, retention
beats, and B-roll briefs. The plan is a JSON file you edit.

The plan's runtime is not a guess: it comes from the channel's own long-form
median, so a new episode lands inside the band the audience already watches.

### 3. Write the script

A script file is JSON:

```json
{ "sections": { "cold-open": "The words for that section.", "results": "..." } }
```

The keys are the section ids from the plan. Use `creatorize author` to apply it.

Write for the ear, not the page. A long-form viewer is listening, not reading.

### 4. Author

`creatorize author` writes a real Hypit project:

```
<project>/
  package.json            hypit.project: true
  authors/episode.svml    the Source
  recipes/longform.svs    the look
  runs/episode.svrun      the Run
  assets/narration/       one audio file per section
  assets/broll/           the picture
  creatorize.json         the plan and the spec
```

The Source follows the structure Hypit actually validates: a Script whose
Segments are the sections, one narration slice and one WhisperX alignment per
section, a Timeline of one Window per section, a Projection that puts the words
on the timeline, an audio Track for the narration, a visual Track for the B-roll,
a karaoke Caption above it, and a Film that composes the three.

`creatorize author` also installs the project's Hypit packages, because a
Source that imports a package which is not installed fails with a resolution
error that reads like a bug in the Source.

### 5. Narrate

Three routes, in descending order of fidelity:

- `--voice <file|dir>` - the creator's own recording. A directory of per-section
  files is used as-is; a single file is accepted but not split.
- `--tts espeak` - local espeak-ng. Robotic, free, and real words, so
  WhisperX can align it and the captions come out correct. It exists to
  validate an episode end to end without a credit.
- (default) - a silent track of the correct length per section.

The default route is a **timing proof**. It prints every section's estimated
duration against its target, and the drift. On a long-form episode this is the
single most useful number you will see:

```
  cold-open          14s  target    51s  drift -37s
  results            34s  target   159s  drift -125s
  ...
  total          310s  (plan target 1462s)
```

`hypit estimate` counts pronunciation units locally. No service, no credit.

### 6. Assets

`creatorize assets` generates the B-roll through Scenario. Each brief becomes a
picture that *proves* something, not an illustration of a noun, and a claim
about speed wants a fast action on screen.

`--placeholder` renders grey slates with ffmpeg instead. Use it to validate the
whole chain - timeline, captions, cuts, export - before spending a credit.

### 7. Build

`creatorize build` runs `hypit check` first, so a broken reference costs a
second rather than a credit. Then it builds and exports the Output.

`creatorize align` produces only the word timings, and `build --reuse <id>`
renders the picture from them. Use the pair when memory is tight, and prefer
it when you are iterating on the picture: the timings never change, so there
is no reason to transcribe again.

## Working rules

- **Check before you build.** `hypit check` is free and catches almost
  everything.
- **Keep the words authoritative.** The Script is the only place the wording
  lives. Do not restate a line in a prompt or a caption file.
- **One section, one job.** A section that only adds information will lose the
  viewer in the middle. Give every section a promise, a proof or a change.
- **Show the flaws.** A pipeline that produces ninety percent and needs a human
  for the last ten percent is worth building. One that calls ninety percent
  done is a way to publish mistakes faster.
- **Never claim a render that did not happen.** `creatorize build` reports what
  the build actually produced.

## Credentials

| What | Needed for | How |
| --- | --- | --- |
| `SCENARIO_API_KEY` | Scenario B-roll and thumbnails | `creatorize auth scenario` (PKCE against a loopback callback) or paste a key from app.scenario.com |
| A Hypit speech provider | Generated narration | `hypit auth login` |
| WhisperX | Caption alignment | local service, or HypiHub |

None of them are needed to plan, author, time, or render with placeholder
material. The whole chain runs offline.
