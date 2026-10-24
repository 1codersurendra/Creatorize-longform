# Creatorize

**Long-form faceless YouTube production, built on REA + Scenario + Hypit.**

Creatorize turns a topic into a finished, uploadable long-form video for the
`g russ` channel (`@thegruss-i8w`). It chains three tools, each doing the job
it is actually good at:

| Tool | Job | What Creatorize does with it |
| --- | --- | --- |
| **[REA](https://github.com/morluto/rea)** | Reverse engineering | Reads the reference: the channel's page and structure via passive CDP, or a shipped app's behaviour, and returns evidence |
| **[Scenario](https://mcp.scenario.com)** | Creative generation | Generates the stills that prove a claim, and the thumbnail - 750+ models, custom model training |
| **[Hypit](https://github.com/hypit-ai/hypit)** | Video language + runtime | Holds the script, the word-anchored timeline, the captions, the B-roll and the render |

Creatorize itself is the orchestrator, and the part that knows the channel:
its section templates, its runtime band, its title grammar.

---

## Why long-form is different here

Short-form is a loop. Long-form is a **spine** - and a spine has to hold a
viewer for twenty-five minutes, which no amount of hook timing can fake.

The channel already knows how to do this. Its own published chapter lists show
two working spines, and Creatorize is built from them:

```
"Opus 5.5 + Creatorize = LAZIEST Way to Make Money With AI"   17:44
  Introduction / Philosophy and results / Routines overview / Setting up
  routines / Project creation and styles / Scripting and voiceover / Advanced
  automation / Building and editing / Quality control and refinement /
  Packaging and final output / Vertical and trailer automation / Community
  and conclusion

"Creatorize + Claude = 26 Minute Documentaries For $0!"          34:04
  Intro / What's new / What it is / The idea / The researched script / The
  voiceover / Setting up the project / The new look / The build direction /
  The crew build, live / The storyboard / The build numbers / The finished
  project / Variety / Checking a long video / Fixing things like a human /
  Full-screen b-roll / Sound and export / Titles, description, chapters / What
  it costs / Tips for long videos / FAQ / The final video / The whole workflow
  / Outro
```

Those are not templates I invented. They are the channel's own decisions about
where a long-form episode turns, and they are encoded in
[`src/pipeline/plan.mjs`](src/pipeline/plan.mjs) as `tutorial` and
`documentary`.

---

## The idea that makes long-form affordable

Hypit is **word-anchored**. The Script holds the words. WhisperX projects those
words onto the audio. Everything else - B-roll, captions, motion graphics -
attaches to a *word*, not to a second.

```
<hook>
  <HOST> @{claim!} I made the @{proof}<API | A P I> || work overnight.@{/proof}
</hook>
```

Rewrite that line and every visual that answered it moves with the words.
Nothing is re-timed by hand. This is the difference between a pipeline that
produces a hundred videos and one that produces a hundred videos you can still
edit.

---

## Install

```bash
npm install -g @hypit/hypit          # the video language and runtime
npm install -g rea-agents            # the reverse engineering MCP server
node --version                       # needs 22.15+ (REA needs 22.23.4+ or 24.11+)
```

Then:

```bash
git clone <this repo> && cd creatorize
node bin/creatorize.mjs doctor
```

`doctor` checks every binary, every credential and every path, and tells you
exactly what is missing.

---

## The pipeline

```bash
creatorize analyze                            # reverse-engineer the channel
creatorize plan "Your episode topic"          # a plan on the channel's spine
#   write the script
creatorize author --plan <plan> --script <script> --project <dir>
creatorize narrate --project <dir>            # voice / local TTS / timing track
creatorize assets --project <dir>             # Scenario B-roll and thumbnail
creatorize build --project <dir>              # check, build, export
```

### 1. `analyze` - the channel as a format spec

Reads the channel page and writes `config/channels/g-russ.json`:

```
Channel      g russ @thegruss-i8w
Videos       30 (30 titled)
Runtime      median 17:34  p25 13:35  p75 25:13  max 3:00:14
Formats      short 0  medium 11  long 18  live 1
Long-form    19 episodes, target 24:22
Style        3d-animation, flat-explainer, documentary, short-form, long-form

Recurring title shapes
    5x  <MODEL> + Creatorize = <N> ...
```

`--web` adds REA's passive browser pass: Creatorize launches a headless Chrome
with a loopback debugging port, REA attaches to it and reports the DOM,
accessibility tree, script graph and route candidates. Nothing is clicked,
navigated or mutated.

### 2. `plan` - a spine for one topic

```
Building a faceless YouTube channel with AI  (Tool tutorial / build-along)
Creatorize long-form tool walkthroughs (17-18 minutes)

   1. Introduction to Building a faceless YouTube channel with AI   51s  123w
   2. Philosophy and results                  159s  384w
   3. Manual vs AI content creation           101s  244w
   4. Setting up the project                  188s  454w
   5. Scripting and voiceover                 217s  524w
   ...
Target: 24:22  ·  3529 words  ·  10 sections
```

The runtime is not a guess. It is the channel's own long-form median, so a new
episode lands inside the band the audience already watches.

`--format documentary` switches to the 25-section documentary spine.
`--chapters <videoId>` borrows a reference episode's own chapter titles.

### 3. `narrate` - the timing proof

The most useful command in the tool, and the cheapest:

```
Section timing (hypit estimate, local, no credits)
  cold-open          14s  target    51s  drift -37s
  results            34s  target   159s  drift -125s
  ...
  total          310s  (plan target 1462s)
```

`hypit estimate` counts pronunciation units locally. No service, no credit, no
account. Knowing that your words overrun section three by forty seconds is
worth knowing before you record twenty-five minutes of narration.

Three routes: `--voice <file|dir>` for the creator's own recording, `--tts
espeak` for free local speech that WhisperX can align, and the default silent
track for a pure timing proof.

### 4. `assets` - the picture

Scenario generates each B-roll brief. A brief asks for a picture that **proves**
something, not an illustration of a noun: a claim about speed wants a fast
action on screen.

`--placeholder` renders grey slates with ffmpeg instead, so the whole chain -
timeline, captions, cuts, export - can be validated before a single credit is
spent.

### 5. `build` - check, build, export

`creatorize build` runs `hypit check` first, because a broken reference should
cost a second rather than a credit.

### Long-form on a modest machine

A twenty-five minute episode has two very different memory profiles: the
transcription service, and the rasteriser that draws every frame. On a machine
with 8 GB or more they can share one build. On a smaller one they cannot, and
the build dies with its executor killed.

Creatorize splits them:

```bash
creatorize align --project <dir>     # transcribe only, produce the word timings
hypit programs down --endpoint whisperx.local
creatorize build --project <dir> --reuse <align-build-id>
```

`align` writes a Run that targets the alignment Outputs and nothing else, so
no picture is drawn. `build --reuse` then satisfies the narration media, its
domain and its alignment from that earlier build, and renders the picture with
the transcription service stopped.

This is also the right loop for editing. The narration and its word timing are
the expensive part; the picture is cheap. Re-rendering the picture of a
twenty-five minute episode takes minutes. Re-transcribing it takes a quarter
of an hour and produces exactly the same timings.

`--reuse` only claims Outputs an earlier build actually produced. A build that
died part-way leaves gaps, and those sections are simply made again.

---

## The generated project

`creatorize author` writes a real, runnable Hypit project:

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

The Source follows the structure Hypit actually validates:

```xml
<import from="@hypit/script@1"/>
...
<media:Audio id="narration-results" src="../assets/narration/results.m4a"/>
<mediaop:Normalize id="narration-results-media" source={narration-results} .../>
<whisperx:Alignment id="align-results" narrative={story}
  segment={story.segment.results} media={narration-results-media.media}
  domain={narration-results-media.domain} language="en"/>
...
<time:Timeline id="program" clock={clock} end="close.end+2s">
  <time:Window id="cold-open" from="start" for={narration-cold-open-media.extent}/>
  <time:Window id="results" from="cold-open.end" for={narration-results-media.extent}/>
  ...
</time:Timeline>
```

One narration slice and one alignment per section, so a 25-minute episode can be
re-cut section by section.

---

## Tool notes

### REA

`rea mcp` speaks newline-delimited JSON-RPC over stdio and exposes 207 tools.
Creatorize uses the web family (`list_browser_targets`, `inspect_web_page`,
`analyze_web_bundle`) and the artifact family (`open_binary`,
`binary_overview`, `list_strings`, `procedure_pseudo_code`).

REA refuses any endpoint that is not a literal loopback CDP address and never
drives a browser for you, so Creatorize owns the browser lifecycle and hands
REA an endpoint. Analysis runs locally; results carry their own evidence and
limitations.

`creatorize recon <path>` points REA at a local artifact - the Creatorize
desktop build, an ASAR, a binary - and writes the evidence to
`evidence/recon-*.json`.

### Scenario

The MCP server lives at `https://mcp.scenario.com/mcp` and is OAuth 2.1
protected with PKCE.

The server advertises a device-code grant, but its dynamic client registration
only issues `authorization_code` clients, so a device request comes back
`invalid_grant`. The authorization code flow is the one that works, and
Creatorize implements it: it registers a client, starts a loopback callback
server, prints the approval URL, and exchanges the returned code with PKCE S256.

The default tool set is deliberately small. `scenario_tools_search` finds
assets, models and workflows; `scenario_tool_execute_*` runs them. Connecting
with `?toolsets=full` lists everything. The adapter reads the live input
schema at runtime and adapts to it, rather than guessing argument names.

### Hypit

`hypit` is the CLI. Creatorize calls `check`, `plan`, `build`, `estimate`,
`download`, `transcribe`, `media`, `status`, `logs` and `get`.

`hypit vocabulary <package>` prints the installed packages, their surfaces and
their exact attributes. Every element Creatorize emits was written against
that output rather than against a guess.

---

## Command reference

```
creatorize doctor                      check every dependency and credential
creatorize analyze [url] [--web]       reverse-engineer a channel into a format spec
creatorize chapters <videoId...>       read a video's own chapter list
creatorize plan <topic>                build an episode plan from the channel spec
creatorize author                      write the Hypit project from a plan
creatorize narrate                     produce the narration (voice / tts / timing track)
creatorize align                       transcribe the narration into word timings only
creatorize assets                      generate B-roll and thumbnail via Scenario
creatorize build                       check, build and export with Hypit
creatorize recon <path>                reverse-engineer a local app with REA
creatorize auth scenario               sign in to Scenario
creatorize init [dir]                  scaffold a project directory
creatorize formats                     list the long-form section templates
```

---

## What was actually verified

Honesty about the boundary between "built" and "works":

| Claim | How it was checked | Result |
| --- | --- | --- |
| Channel reconnaissance | `creatorize analyze` against the live channel | 30 videos, 30 titles, 30 runtimes, 5 title formulas, long-form band of 19 episodes |
| REA browser pass | `creatorize analyze --web` | 6380 DOM nodes, 1466 accessibility nodes, 11 network requests, with REA's own completeness and limitations |
| The generated Source is valid | `hypit check authors/episode.svml` | `✓ Source is valid - 176 outputs` |
| The build plan is valid | `hypit plan runs/episode.svrun` | `✓ Build plan is valid - 73 requests` |
| Transcription really runs | local WhisperX 3.8.6, CPU int8 | all ten sections transcribed and word-aligned, timings in the service log |
| The whole chain runs offline | `node test/smoke.mjs` | 19 passed, 0 failed |
| Scenario OAuth | live server, deliberately fake code | client registered, PKCE URL built, loopback redirect received, token exchange attempted |
| REA on a local app | `creatorize recon` on an Electron-style app | module graph recovered, with REA's own limitations |
| A finished render | - | **not produced here** |

The last row is the honest gap, and it has a specific cause. This sandbox caps
the container at 3 GB. The transcription service alone holds about 1.4 GB, and
the rasteriser needs the rest. The first full build transcribed all ten
sections successfully, produced 25 normalised media files, and was then killed
by the cgroup OOM killer during rasterisation. The `align` / `--reuse` split
above exists because of that finding - it is the fix, and it is implemented and
tested, but the render it enables was not completed inside this 3 GB limit.

Everything up to the render is verified against the real tools.

---

## License

MIT. REA is MIT. Hypit is Apache-2.0. Scenario is a hosted service with its own
terms.

Both REA and Hypit are for lawful research, education, interoperability,
software preservation, and analysis of software you own or are authorised to
examine.
