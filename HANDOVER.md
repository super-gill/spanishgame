# Spanish Practice App — Handover

This document hands off the Spanish learning app to a code-focused session. The app is a single-file HTML game collection built incrementally over multiple tutoring sessions.

## TL;DR

- **App**: `/mnt/user-data/outputs/spanish-app.html` (~2,970 lines, 132 KB, single file)
- **Reference book**: `/mnt/project/Spanish_1.md` (~1,640 lines, read-only project file)
- **Stack**: vanilla HTML + CSS + JS, no build step, all-in-one
- **Persistence**: Claude artifacts `window.storage` API, key `jason-spanish-app-v1`
- **Audience**: Jason, IT manager near Milton Keynes, learning **Spain Spanish** (vosotros, "th" for c/z, *patata* not *papa*, *zumo* not *jugo*)
- **9 games**, all in a single `Games` object
- **Standing rule for any new content**: no em dashes (user preference). Existing em dashes left untouched for consistency.

## File layout

The HTML file is organised top to bottom roughly like this:

| Line range (approx) | Section |
| --- | --- |
| 1–390 | `<style>` block — all CSS |
| 390–425 | `<body>` skeleton — header, nav, `#main` mount point |
| 425–520 | DATA: vocab (16 topics) |
| 520–860 | DATA: verbs (92 entries) |
| 860–950 | DATA: articleNouns (92 entries) |
| 950–1010 | DATA: sentences (69 entries) |
| 1010–1075 | DATA: placeActions + allPlaces |
| 1075–1130 | DATA: numbers + times |
| 1130–1290 | DATA: dialogues (2 entries) |
| 1290–1340 | STORAGE + makeProgress + load/save |
| 1340–1395 | Utilities: normalize, pickWeighted, weightForKey, gameAggregateStats, weakSpotsForGame, recordResult |
| 1395–1455 | Navigation: navigateTo, home rendering |
| 1455–2945 | `Games` object — 9 games |
| 2945–end | Global hint button handler + init() |

If line numbers drift after edits, search anchors are usually unique enough: `Games.verbs`, `articleNouns:`, `// === Stem O→UE`, etc.

## DATA shape

```js
const DATA = {
  vocab: {
    family: [{en, es}, ...],
    pets: [...],
    food: [...],
    city: [...],
    jobs: [...],
    countries: [...],
    days: [...],
    weather: [...],
    adjectives: [...],
    colours: [...],
    emotions: [...],
    flavours: [...],
    homes: [...],
    timeOfDay: [...],
    rooms: [...],
    houseObjects: [...]
  },
  verbs: [
    {
      inf: 'hablar',
      en: 'to speak',
      cat: 'reg-ar',  // see categories below
      conj: {
        present: {yo, tu, el, nos, vos, ellos},
        past:    {yo, tu, el, nos, vos, ellos}  // optional
      }
    },
    ...
  ],
  articleNouns: [{noun, article}, ...],  // article is 'el' or 'la'
  sentences: [{en, es, focus}, ...],     // focus is the word being tested in fillBlank
  placeActions: [{verb, en, places}, ...],
  allPlaces: ['biblioteca', 'casa', ...],
  numbers: [{num, es}, ...],
  times: [{display, es}, ...],           // display like '4:15', es like 'son las cuatro y cuarto'
  dialogues: [
    {id, name, description, userRole, turns: [{speaker, en, es}, ...]}
  ]
};
```

### Verb categories (`cat` values)

| Value | Meaning |
| --- | --- |
| `reg-ar` | Regular -AR |
| `reg-er` | Regular -ER |
| `reg-ir` | Regular -IR |
| `stem-ie` | E→IE stem-changer |
| `stem-ue` | O→UE stem-changer (jugar U→UE grouped here for simplicity) |
| `stem-i` | E→I stem-changer |
| `irreg` | Irregular |
| `reflexive` | Reflexive (pronoun me/te/se/nos/os/se baked into each conjugation: `'me ducho'`, `'te duchas'`) |

### Past tense policy

Past tense (`conj.past`) is included **only** for verbs with fully-regular preterites. The following get **present only**:

- Orthographic-change verbs (yo busqué, llegué, pagué, practiqué)
- All stem-changers
- All irregulars
- All reflexives
- `leer` (i→y in él/ellos forms)

The current syllabus hasn't covered irregular preterites yet. When Jason's human tutor covers them, add past forms verb by verb.

## Storage and progress tracking

```js
const STORAGE_KEY = 'jason-spanish-app-v1';
const GAME_KEYS = ['verbs','vocab','sentenceBuilder','fillBlank','articles',
                   'timeNumbers','placeAction','dialogue','recognition'];

// Shape:
progress = {
  verbs: { 'hablar:yo:present': {correct: 5, incorrect: 1}, ... },
  vocab: { 'manzana': {correct: 3, incorrect: 0}, ... },
  ...
};
```

### Critical: prototype-pollution guard

`progress` is built with `Object.create(null)` via the `makeProgress()` helper. This was added after a real bug where the vocab game always picked "snow" because vocab included "constructor" (builder), which then triggered `progress.vocab['constructor']` returning `Object.prototype.constructor` and poisoning the weight calculation.

Always:
- Use `Object.create(null)` for any new progress-style object
- `weightForKey` guards: only treats the stored value as stats if `typeof s === 'object'` AND numeric `correct`/`incorrect` exist
- `pickWeighted` falls back to uniform random if any weight is non-finite

If you add a new game with new item keys, those guards already cover you, but stay alert if you change the storage shape.

### window.storage caveat

`saveProgress()` calls `window.storage.set(...)`. In Claude artifacts (claude.ai) this works and persists across sessions. **Loading the HTML file directly via `file://` for testing throws "Save failed" errors** — these are harmless dev-only noise. Filter them in test scripts:

```js
page.on('console', m => {
  if (m.type() === 'error' && !m.text().includes('Save failed')) {
    errors.push(m.text());
  }
});
```

## Game pattern

Every game in `Games` follows roughly the same shape:

```js
Games.someGame = {
  name: 'Display Name',
  description: 'One-liner shown on home card',
  state: { /* per-game UI state */ },

  init() {
    this.state = { /* reset */ };
    // Render UI shell into #main
    // Hook up event listeners
    this.next();
  },

  next() {
    // Pick next item via pickWeighted, populate UI
  },

  check(answer) {
    // Compare, recordResult('gameKey', itemKey, correct), show feedback
  },

  renderStatsBlock() {
    // Returns HTML for the stats panel at the bottom
    // Uses gameAggregateStats and weakSpotsForGame helpers
  },

  refreshStatsBlock() {
    const block = document.getElementById('stats-block');
    if (block) block.outerHTML = this.renderStatsBlock();
  }
};
```

Games are mounted into `#main` via `navigateTo(gameKey)`, which calls `Games[gameKey].init()`. Home is `navigateTo('home')` which renders the game grid.

### Per-game item keys (used in `progress`)

| Game | Key format |
| --- | --- |
| `verbs` | `'{inf}:{person}:{tense}'` e.g. `'hablar:yo:past'` |
| `vocab` | `'{es}'` |
| `articles` | `'{noun}'` |
| `fillBlank` | `'{es-sentence}'` |
| `sentenceBuilder` | `'{es-sentence}'` |
| `timeNumbers` | `'num:{n}'` or `'time:{display}'` |
| `placeAction` | `'{verb-entry}'` |
| `dialogue` | `'{dialogue-id}:{turn-index}'` |
| `recognition` | `'words|{es}'` or `'sentences|{es}'` |

## The 9 games

1. **verbs** (Verb Drill) — conjugation input. Filters: All / Present / Past / Regular only / Weak spots. "Regular only" uses `cat.startsWith('reg-')` so reflexives correctly excluded.
2. **vocab** (Vocab Matching) — multiple choice. Has a topic picker with `all` plus the 16 topics. Auto-discovers topics from `Object.keys(DATA.vocab)`.
3. **articles** (El or La) — binary gender choice from articleNouns.
4. **fillBlank** (Fill the Blank) — fills in the `focus` word from a sentence. English sentence behind a hint button.
5. **sentenceBuilder** (Sentence Builder) — drag/click word tiles into order.
6. **timeNumbers** (Time & Numbers) — input the Spanish for a number or time.
7. **placeAction** (Place Match) — match a verb action to one or more places. English meaning behind a hint button.
8. **dialogue** (Dialogue Replay) — type out user role's turns in a scripted dialogue. Two dialogues: `sara-fernando` (12 turns) and `carolina-miguel` (24 turns). Other speaker's English behind hint button.
9. **recognition** (Reconocer) — newest. Two modes via toggle:
   - **Words**: flashcards. Show Spanish, reveal English + grammar info + agreement/conjugation samples. Rate Knew it / Fuzzy / Didn't know.
   - **Sentences**: full Spanish sentence shown, each word a tappable chip. Tap to reveal that word's grammar info via popover. "Show full translation" reveals English. Rate Got it / Needed help.

## Recognition game specifics

`Games.recognition` is the most complex game. Key pieces:

### `FUNCTION_WORDS` dictionary

A flat lookup of ~150 common Spanish words that don't naturally fit in the vocab/verb/article structure: articles, prepositions, pronouns, possessives, conjunctions, question words, common adverbs, tener-idiom nouns (hambre, sed, miedo...), languages (español, inglés, francés...), common irregular preterites (fui, fue, tuvo, dijo, hizo...), gustar/encantar/importar forms (gusta, gustan, encanta...).

Each entry shape: `{en: 'meaning', meta: 'grammatical note'}`.

### `buildPool()`

Lazy-builds and caches `_pool`, a unified array of word entries combining:
1. All vocab items, with `type` inferred from topic (see `TYPE_BY_TOPIC`)
2. articleNouns not already in vocab (with placeholder English `'(see article quiz)'`)
3. All verb infinitives, with `cat` and `sampleYo`

Each pool entry: `{es, en, type, ...rest}` where type is one of `noun`, `verb`, `adjective`, `phrase`, `place`, `word`.

### `lookupWord(rawWord)`

The sentence parser's brain. Strategy:
1. Lowercase and strip punctuation
2. Function words exact match
3. Function words match without accents (normalize)
4. Pool exact match
5. Verb conjugation lookup — also strips reflexive `me/te/se/nos/os` prefix from stored forms so `levanto` finds `me levanto`
6. Number text lookup (cinco, treinta, etc.)
7. Reflexive prefix retry: if word starts with `me /te /se /` etc., strip and retry
8. Plural -es with z→c flip (lápices → lápiz)
9. Plural -s
10. Adjective gender variants (bonita → bonito)

Returns `null` if nothing matches — chip shows "Not in dictionary" and gets `.unknown` styling.

### Coverage

98% of words in the existing 69 sentences resolve. The 2% unresolved are proper nouns (Madrid, Sevilla, Pluto) which should stay unresolved.

### Rating logic

- **Knew it** → `recordResult('recognition', key, true)`
- **Fuzzy** → records both a correct AND an incorrect (light penalty, nudges weighting)
- **Didn't know** → `recordResult('recognition', key, false)`

If Jason later asks to make Fuzzy harsher or lighter, that's the spot to change.

## Conventions and gotchas

### Content style
- **No em dashes** anywhere in new content (user preference). Existing em dashes in vocab/grammar tables are left alone for consistency.
- **Spain Spanish vocabulary**: `patata` (not papa), `zumo` (not jugo), `ordenador` (not computadora), `móvil` (not celular), `coche` (not carro), `water` for toilet (yes really).
- **Use vosotros** as second person plural informal. Conjugation tables always include the vos row.
- **Reflexive verbs use `-se`** form in infinitives and bake pronouns into conjugations: `ducharse → {yo:'me ducho', tu:'te duchas', ...}`.

### Adding a new verb
Add to `DATA.verbs`. Include the appropriate `cat` value. Include `conj.past` only if it's fully regular per the past tense policy above. Conjugation order is always `yo, tu, el, nos, vos, ellos`.

### Adding a new vocab topic
Add to `DATA.vocab`. The vocab game's topic buttons auto-generate from `Object.keys(DATA.vocab)`. If the topic is a noun category, also add a `TYPE_BY_TOPIC` entry in `Games.recognition` for the recognition game to know the word type. If items are nouns, consider also adding them (or selected items) to `articleNouns` with their gender so they show up in the El/La game.

### Adding a new sentence
Add to `DATA.sentences`. The `focus` field is what gets blanked out in Fill the Blank — pick a pedagogically interesting word. The same sentence is used by Sentence Builder (tile order) and Recognition (sentence parse).

### Adding a new dialogue
Add to `DATA.dialogues`. Required fields: `id` (kebab-case), `name`, `description`, `userRole` (the speaker name Jason plays), `turns` (array of `{speaker, en, es}`). The dialogue UI auto-shows non-user turns and prompts for user turns.

### Adding a new game
1. Add the key to `GAME_KEYS` so storage initialisation reserves a slot
2. Create `Games.newGame = { name, description, state, init, ... }` following the pattern
3. The home grid renders automatically via `Object.keys(Games).map(...)`
4. Add CSS for any new UI elements
5. If you need a hint button, use the existing `.hint-row`/`.hint-btn`/`.hint-text` pattern — there's a global event delegation handler near the bottom that handles reveal/hide automatically

## Testing

There's no formal test suite. The pattern during development was Playwright scripts in `/tmp/`. Typical smoke test:

```js
const { chromium } = require('playwright');
(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  const errors = [];
  page.on('pageerror', e => errors.push(e.message));
  page.on('console', m => {
    if (m.type() === 'error' && !m.text().includes('Save failed')) {
      errors.push(m.text());
    }
  });
  await page.goto('file:///mnt/user-data/outputs/spanish-app.html', { timeout: 5000 });
  await page.waitForTimeout(300);
  // ... drive the UI ...
  console.log('Errors:', errors);
  await browser.close();
})();
```

Use `data-game="recognition"` selectors and `#main` element IDs. Re-query elements after each re-render to avoid stale handles.

For data structure checks without a browser:

```js
const fs = require('fs');
const html = fs.readFileSync('spanish-app.html', 'utf8');
const dataStart = html.indexOf('const DATA = ') + 'const DATA = '.length;
let depth = 0, i = dataStart, end = -1;
for (; i < html.length; i++) {
  if (html[i] === '{') depth++;
  else if (html[i] === '}') { depth--; if (depth === 0) { end = i + 1; break; } }
}
const DATA = eval('(' + html.substring(dataStart, end) + ')');
// ... inspect DATA ...
```

## Current state (as of June 2026)

### Counts
- 92 verbs (30 reg-ar, 7 reg-er, 6 reg-ir, 11 stem-ie, 9 stem-ue, 3 stem-i, 10 irreg, 16 reflexive)
- 16 vocab topics covering ~280 items
- 92 article nouns
- 69 sentences
- 22 place-action pairs across 30 places
- 47 numbers, 24 times
- 2 dialogues

### Recently added (latest session)
- "En La Casa" content: rooms (9 items) + houseObjects (24 items) as two new vocab topics
- Tener idioms (hambre, sed, sueño, miedo, etc.) — content in sentences plus a reference table in the book
- 8 new verbs: cerrar, pensar, encontrar, acostarse, quedarse, llevarse, sentirse, encontrarse
- 19 new sentences using the above
- New game: **Reconocer** (Recognition) with Words and Sentences modes

### Known to-dos / open candidates
- **Past tense expansion**: as Jason's human tutor covers irregular/stem-changing/reflexive preterites, add `conj.past` to the relevant verbs. Highest priority candidates flagged in conversations: *almorzar*, *vestirse*, *dormir*.
- **Recognition filters**: no filters currently. Words mode mixes nouns/verbs/adjectives. Adding `Nouns only / Verbs only / Weak spots only` filters in the mode toggle area would mirror Verb Drill's pattern.
- **Adquirir / inquirir** (I→IE, rare stem-change pattern): mentioned in source materials but not added because rare. Could add if Jason hits them in reading.

### Verbs flagged for future full conjugation tables in the book
*Almorzar*, *Vestirse*, *Dormir* — already in the app but only listed in summary tables in the book. If Jason asks for a deeper dive, give them dedicated tables like *Querer*, *Poder*, etc.

## Workflow notes

- **ZIP-file syncs**: Jason periodically drops a course-material ZIP. Workflow: extract with `find ... -print | while read -r f` (filenames have spaces), diff against the book, add genuinely new content, flag ambiguous cases. Don't add things speculatively.
- **Book updates require copying out of project**: `/mnt/project/Spanish_1.md` is read-only. To edit, copy to `/home/claude/Spanish_1.md`, edit there, then `cp` to `/mnt/user-data/outputs/Spanish_1.md` for delivery.
- **App updates are direct**: `/mnt/user-data/outputs/spanish-app.html` is writable. Use `str_replace` for targeted edits, verify with the data-structure check after, then a Playwright smoke test for UI changes.
- **Jason prefers explicit scoping**: when offering work, list options and let him pick rather than making unilateral calls on what to add. "Do everything (big batch)" is a valid answer he sometimes gives — that's a go-ahead.

## Quick command reference

```bash
# Count verbs by category
grep -c "cat:'reg-ar'" /mnt/user-data/outputs/spanish-app.html

# Find a game's init
grep -n "^Games\." /mnt/user-data/outputs/spanish-app.html

# Verify data parses
node -e "
const fs = require('fs');
const html = fs.readFileSync('/mnt/user-data/outputs/spanish-app.html', 'utf8');
const s = html.indexOf('const DATA = ') + 13;
let d = 0, i = s, e = -1;
for (; i < html.length; i++) {
  if (html[i] === '{') d++;
  else if (html[i] === '}') { d--; if (d === 0) { e = i + 1; break; } }
}
const DATA = eval('(' + html.substring(s, e) + ')');
console.log('Verbs:', DATA.verbs.length, 'Vocab topics:', Object.keys(DATA.vocab).length);
"
```

## Important context about Jason

- Mid-40s IT manager. Brother is Ross, dog is Pluto.
- Uses Windows dictation (Win+H) and Spanish keyboard layout for verbal practice.
- Curriculum is paced by his human tutor; the book and app are durable artifacts of that work, not the curriculum itself.
- Errors he's been drilling: `de + conjugated verb` (he self-corrected this cleanly), gustar agreement with article + plural, stem changes in reflexives (despertarse e→ie), Spanish vowels not reducing like English.
- He likes side-by-side reasoning and clear options. Doesn't like Claude making big decisions unilaterally without listing them.
