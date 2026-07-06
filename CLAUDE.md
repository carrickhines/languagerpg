# LanguageRPG

A browser-based language-arts game for two kids. The gameplay is a simple
turn-based RPG combat loop: the player and a foe trade attacks, and each attack
must be "earned" by answering a language question. Answer correctly (and fast
enough) to land your hit; fail or run out of time and the foe gets a free swing.

The game covers **spelling, grammar, and other language subjects**, with room to
grow toward **reading comprehension** later.

## Who it's for

Two players, at two different levels:

- **Younger son (age 5)** — just beginning to read and write. Content should
  stay at the earliest level: letter sounds, simple sight words, very short
  words. Big, forgiving, and encouraging. **His four tracks are built** (see
  below) — the current focus is getting him kindergarten-ready.
- **Older son (age 8, turning 9, 3rd grade)** — a confident reader working on
  correct spelling and grammar. Content can be more challenging: spelling
  harder words, grammar rules, and eventually comprehension. **His tracks are
  not built yet.**

The level/track is chosen at the start of a game, so each kid plays the version
suited to them.

## The no-transcription architecture (key design decision)

The game **never listens to the child** — no speech-to-text, no transcription
service, no API keys, no external services. Instead:

- **The game's voice is the parent's voice.** The game has a *closed
  vocabulary* (~84 short prompts total), so `record.html` — the "Voice
  Studio" — lets a parent record every prompt once (~10 min) with the
  browser's built-in `MediaRecorder`, then downloads a single generated
  `voice.js` (clips as base64 data URIs) that sits next to `index.html`. Data
  URIs mean no fetch, so it works from `file://` too. The `Voice` module in
  `index.html` plays clips through one reusable `<audio>` element (unlocked on
  the Start tap for iOS), chaining multi-clip prompts like `P_spell` + `W_cat`
  ("Spell this word!" → "cat").
- **`speechSynthesis` TTS is only a fallback** for clips not yet recorded
  (the `Speech` module). It was originally the primary voice, but robot TTS
  quality (especially outside Apple devices) proved too poor for a child to
  make out words — that's why recorded clips are primary. Record on the device
  the kids play on (mp4/AAC records everywhere and plays everywhere;
  Chrome-recorded webm may not play on iPads).
- **Emoji are the art assets.** Every picture the game needs (🐱 for "cat")
  ships with the OS — nothing to host, nothing to load.
- **All input is tapping** — letter blocks, word cards, picture cards. No
  typing, no talking. A 5-year-old can drive everything with one finger.

The clip-id scheme: `L_B` (full letter prompt), `W_cat` (word), `S_the`
(sight word), `P_*` (game phrases: spell/findword/great/double/itwas/win/lose).
The content lists (`LETTERS`/`WORDS`/`SIGHT`) are duplicated in `record.html` —
**keep them in sync with `index.html`** when adding content, and re-record the
new clips.

Reading *aloud* is the one skill this can't grade; that stays a
parent-at-bedtime activity by design. Don't add transcription services back.

## Younger son's tracks (built, in `index.html`)

Four tracks forming a learning ladder toward reading:

1. **🔠 Letter Hunt** — letter recognition + initial sounds. TTS says "Find the
   letter B! B, as in ball!", a cue emoji (⚽) shows in the card, and the child
   taps the right letter block out of 4. Content: all 26 letters with cue
   words (`LETTERS`).
2. **🧱 Word Builder** — encoding/spelling. Shows an emoji (🐱) and says the
   word; the child spells it by tapping alphabet blocks into dashed slots
   (word's letters + 2 decoys, shuffled). Tapping a filled slot puts the
   block back. Auto-checks when full. Content: ~26 three-letter words with
   clear emoji (`WORDS`).
3. **⭐ Sight Words** — TTS says a word ("Find the word... the!") and the child
   taps it among 3 word cards. Content: Dolch pre-primer list (`SIGHT`).
4. **🖼️ Read It!** — true decoding. Shows the *written* word with **no audio
   hint**; the child reads it and taps the matching emoji out of 4. The word
   is spoken *after* answering as reinforcement.

Answer tiles are styled as chunky **ABC blocks**, each tinted its own hue by
alphabet position — the spiritual sibling of the physical alphabet blocks and
the math game's Numberblocks counting aid.

Foes are five emoji monsters with escalating HP (`FOES`): Slow Slime 🐌 →
Pixel Imp 👾 → Giggly Ghost 👻 → Chompy Rex 🦖 → The Big Dragon 🐉. The player
has 5 hearts; a normal hit does 1 damage, a double hit 2.

## Core gameplay loop

1. A foe appears. Player and foe each have HP.
2. To attack, the player is shown a language question (spelling, grammar, etc.).
3. The player answers.
4. **Correct + in time** → player's attack lands, foe loses HP.
5. **Wrong or out of time** → the attack misses (or the foe counterattacks and
   the player loses HP).
6. Repeat until either the foe's HP or the player's HP hits zero.
7. Win → next foe / victory screen. Lose → game over.

Keep the RPG framing light and fun (simple foes, hits, HP bars) — the point is
to make drilling language skills feel like a game, not a worksheet.

## Difficulty modes

Three modes, distinguished primarily by **time pressure** on each problem:

- **Easy** — long, relaxed timer. Good for the 5-year-old learning the loop.
- **Normal** — moderate countdown per problem.
- **Expert** — shorter countdown; rewards quicker recall.

The content stays tied to the chosen track; the modes change how much time the
player gets, not the kind of question. (If we later want harder modes to also
widen the content difficulty, document that here when we add it.)

## Double Attack Timer

Every mode (including Easy) shows a timer bar on each problem. The early part of
the bar is a gold **fast zone**, ending at a white marker line:

- Answer correctly **while still in the fast zone** → a **DOUBLE hit** (double
  damage) with a bigger, flashier attack animation and a "DOUBLE!" callout.
- Answer correctly after the fast zone → a normal single hit.
- This applies to **all tracks** the same way.

Per-mode timing lives in the `MODES` config in `index.html` (`total` = full bar
duration, `fast` = the double-hit window, `penalty` = whether timing out lets the
foe counterattack).

**Easy stays pressure-free:** the timer is shown and the double-hit bonus is
available, but running out of time does **not** let the foe attack
(`penalty: false`). The timer is purely a reward for speed, never a punishment.
Normal and Expert keep the timeout penalty (`penalty: true`). Note: a *wrong*
answer still lets the foe counterattack in every mode.

## Win reward — iPad game time

Beating every foe (the **Victory** screen, not Game Over) grants a fun,
variable reward: **5–10 minutes of iPad game time** (`rand(5, 10)`, stored as
`state.earnedMins`). The win screen shows the earned minutes and a **"Start the
timer"** button that runs an in-app countdown (`startRewardTimer`). When it
reaches zero (`ringTimer`), the countdown flashes red, shows **"TIME'S UP!"**,
and a **looping two-tone alarm** plays (`Sound.startAlarm` / `stopAlarm`) until
the player taps **"Stop alarm"**. The countdown tracks a wall-clock end time so a
delayed tick can't let the total drift. Navigating away (Play Again / starting a
new game) always calls `clearRewardTimer`, so the alarm never bleeds into the
next screen.

**Known iOS limitation — by design, not a bug:** the in-app timer/alarm only
works while the page is **open and in the foreground with the screen on**. iOS
Safari (and all iOS browsers) freeze JavaScript timers and Web Audio in
backgrounded tabs, and a web page **cannot steal focus or come to the
foreground** — there is no API for it (and no Vibration API on iOS). So the
moment the kid switches to another game, our countdown pauses and the alarm
won't fire on time. That's why the reward screen nudges the player to **keep the
screen open, or ask Siri to set a timer** — the iPad's own Clock/Siri timer is
the only thing that reliably interrupts another app. Don't try to "fix" this
with notifications/PWA push: it needs an installed PWA + permission and still
can't schedule a future alarm without live JS, which breaks the single-file
`file://` design.

## Sound

All sound effects are **synthesized at runtime with the Web Audio API** — there
are no audio files to ship, so the game stays a single self-contained HTML that
works from `file://`. The `Sound` module near the top of the script in
`index.html` builds short tones/noise bursts for: attack hit, the SUPER double
hit, taking damage, wrong/too-slow, foe KO, victory, game over, UI clicks, and
the looping "time's up" alarm for the iPad-game-time reward.

- A 🔊 **mute toggle** sits in the top-right corner; the preference is saved to
  `localStorage`.
- The audio context is created/unlocked on the first user gesture (Start), to
  satisfy browser autoplay rules.

## Technical approach

- **Pure HTML game.** Plain HTML + CSS + JavaScript, no build step and no server.
  It should run by opening an `.html` file directly in a browser (`file://`) or
  by serving the folder statically.
- **No external dependencies / frameworks** unless there's a clear reason —
  prefer vanilla JS so it stays easy to open, read, and tweak.
- Keep it a single self-contained game that's easy for a parent to launch on a
  laptop or tablet.

## Visual design — shared system with the math game

The UI deliberately matches the family's **Math RPG** game so the two feel
like siblings (a local copy of that game may sit at `index-ref.html`,
gitignored, purely as a styling reference). The shared system: a rounded
`#app` frame on a dark page, twilight starfield + pink horizon-glow arena,
glassy dark panels (`--panel`) with backdrop blur, gold gradient buttons with
a 3D "lip" shadow, rounded display font, HP drawn as countable pips (green
hearts for the hero, pink orbs for the foe), a hero-vs-foe emoji sprite arena
with lunge/slash/screen-flash attack animations, and the timer bar with a
white flag marker plus "⚡ DOUBLE! ⚡" label for the fast zone. When touching
the UI, keep these two games visually consistent (they don't need to be
identical).

## Design priorities

- **Kid-friendly UI.** Big text, big buttons/entry, clear feedback, bright and
  simple visuals. Must be usable by a 5-year-old.
- **Immediate, encouraging feedback.** Celebrate correct answers; make wrong
  answers gentle, not punishing.
- **Fast to start.** Minimal menus: pick track, pick mode (easy/normal/expert),
  play.
- **Readable code.** This is a hackable family project — favor clarity over
  cleverness so it's easy to adjust content, timers, and visuals.

## Project structure

```
index.html      # the whole game: menu, combat, all four tracks, sound, voice
record.html     # Voice Studio — parent records prompts, generates voice.js
voice.js        # generated by record.html (parent's voice clips as data URIs);
                #   optional — game falls back to TTS without it. Commit it.
.github/workflows/pages.yml   # GitHub Actions workflow that publishes the
                              #   repo root to GitHub Pages on push to main
```

Game logic stays inline in `index.html` (CSS + JS) on purpose — the only extra
files are the recorder page and its generated clip bundle.

## Running

Open `index.html` in a web browser. No build or install step.

## Hosting / deployment

The host is **GitHub Pages** (handy for loading the game on an iPad without
copying files around). It mirrors the single-file approach: the GitHub Actions
workflow `.github/workflows/pages.yml` uploads the repo root as the Pages
artifact and deploys it, so `index.html` at the repo root *is* the site, and
**pushing to `main` deploys** with no build step. (The repo's Pages settings
must have Source set to "GitHub Actions".) Keep the game a single
dependency-free HTML file so it works both locally and when served from Pages.

- Repo: https://github.com/carrickhines/languagerpg
- Live URL: https://carrickhines.github.io/languagerpg/
