# Changelog

All notable changes to stemmer-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `stemlang` — one closed enum over every language Snowball publishes
  an algorithm for, and the decision the rest of the package follows
  from: the enum is the WHOLE upstream set, not the shipping set.
  Declaring an arm per ported language and adding one per release
  would make every later release a breaking change, because a caller
  that maps its configuration onto a `StemLang` matches on it
  exhaustively and a new arm stops that program compiling. So the
  languages are declared once and availability is DATA:
  `available` answers the list, `is_available` answers one, and
  `require` turns the question into a `Result` a caller runs at
  start-up beside where it reads its configuration — rather than
  meeting a `todo()` at the first word of the first document, inside
  a worker, hours later. **0.1.0 ships two arms: `StemEnglish`
  (Porter2) and `StemPorter` (the original 1980 algorithm).** The
  second is not a legacy alias: the two disagree on ordinary words —
  `ties` gives `tie` and `ti` — and a paper quoting retrieval numbers
  quotes them against one of them.
- `stemerror` — four refusals, with `code` stable across releases and
  `is_configuration_fault` separating the one a start-up check would
  catch from the three that depend on a token.
  `StemUnsupportedLanguage` is the refusal this package exists to
  make: **a stemmer that does not know a language must not fall
  through to English.** English stemming over French answers
  `parlaient` for `parlaient` and `manges` for `manges`, which is not
  nothing — it is a half-working index, wrong in a way no test over
  English vectors can see, and the single most likely thing a hurried
  implementation would do.
- `stemcase` — the case contract, published, because the reference
  implementations leave it unwritten. `stemword.stem` does NOT
  lower-case and REFUSES a word holding an upper-case letter.
  `rust-stemmers` and `snowballstemmer` both hand `Running` straight
  back, because no suffix in the step tables is spelt with a capital;
  nothing is raised, and the caller has written a second unrelated
  term into an index that already holds `run`. Folding inside `stem`
  would fix that and cost more than it saves, because lower-casing is
  language-dependent: Turkish lowers `I` to `ı` and `İ` to `i` and
  every other rule gets both wrong. So the refusal is loud, and
  `fold_ascii`, `is_prepared`, `needs_unicode_fold` and `fold_note`
  are what a caller satisfies it with.
- `stemregion` — R1, R2, the vowel test, the short-syllable rule and
  the marking of a consonant `y`, published rather than private.
  Nearly every rule of nearly every Snowball algorithm reads "remove
  this suffix if it lies in R1", so the regions ARE the reason a
  suffix was or was not removed, and a reader comparing this package
  with another implementation disagrees here first.
  `english_r1_prefixes` publishes `gener`, `commun` and `arsen` as
  data: without that override the whole `generate`, `general`,
  `generic`, `generous` family collapses to `gener`, and a reader who
  meets one family keeping more of itself than its neighbours has no
  other way to find out why. Every function takes the language and
  answers a `Result`, because the vowel set is part of the algorithm
  and a region computed under the wrong one is a plausible number.
- `stemporter` — the English algorithm's eight passes as values and
  its three exception tables as lists. The tables are DATA, not
  branches: eleven words stemmed by lookup, seven that are their own
  stem, eight more that stop after step 1a. In the generated C they
  are `among` tables a reader cannot see and in most ports an `if`
  chain; here `is_invariant("news")` answers the question a user
  actually brings to a stemmer, which is never "what does this stem
  to" but "why did THIS word not move". `apply_step` runs one pass on
  its own, which is how a reader finds where this package and another
  implementation diverge without printing from inside somebody's C.
- `stemword` — the entry point, and there is no stemmer object,
  because **stemming is a pure function of the word**. The reference
  implementations hand back a `Stemmer` that holds nothing but the
  choice of language: no table to warm, no cache, no buffer the
  second call reuses. So the language is the first argument, the
  whole package is `core` and every row is `[]`. Three consequences
  are stated where a caller meets them: the same word always stems to
  the same string, so memoising in front of this package is always
  correct and removes most of the work on a Zipf-distributed corpus;
  nothing has to be shared or torn down; and an index is comparable
  with ITSELF only, which is why `algorithm_id` is published for an
  index to record. **A stem is not a word** — `arguing` gives `argu`,
  `ugly` gives `ugli` — and the stem and its explanation are TWO
  calls, because an indexer makes millions of them and wants a string,
  not a trace.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the four suites reaches `not implemented:
  stemmer-nv.<module>.<fn>`.
- **unicode-nv is refused, and the refusal is revisitable.** The only
  algorithms 0.1.0 ships are English (Porter2) and Porter 1980, both
  defined entirely over the ASCII letters `a`..`z`: the vowel set is
  `aeiouy`, the short-syllable rule tests ASCII consonants, and both
  exception tables are ASCII words. Taking unicode-nv would put its
  245 KB of tables into every program that stems English and use none
  of it, and a dependency taken and unused is never removed, while one
  added later breaks nobody. The algorithms that will force the
  question are named rather than hidden, and they are the ones whose
  first act is not stemming: Greek, which removes diacritics before it
  looks at a suffix; Turkish, whose lower-casing is wrong under any
  other locale's rule; and Russian, Serbian, Armenian, Arabic, Hindi,
  Nepali, Tamil and Yiddish, whose alphabets must be decoded from
  UTF-8 before a vowel can be tested. Snowball's own generated
  stemmers carry per-language character bitmaps rather than a Unicode
  database, so even those need decoding and folding, not general
  category lookup. What a caller does instead today is published:
  `stemcase.needs_unicode_fold` answers true for exactly that set and
  `stemcase.fold_note` is the sentence saying what to do about each,
  so a caller folds with unicode-nv itself and hands the folded word
  here.
- **Twenty-eight declared languages are not available.** They are arms
  of `StemLang` with no algorithm behind them, which is exactly the
  `todo()`-in-production risk the enum decision creates; it is paid
  for with `is_available` and `require` rather than with a smaller
  enum. A caller that never calls `require` still cannot be surprised
  silently, because `stem` refuses with `StemUnsupportedLanguage`.
- **No effect row wanted to widen.** Every public function is `[]`.
  The one place `core` was felt at all is that a stemmer cannot read
  its own algorithm from a `.sbl` file, which is what a Snowball
  compiler would do; the algorithms are ported by hand and that is
  named in the README's "What is not included".
