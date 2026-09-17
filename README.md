# stemmer-nv

Stemming reduces the inflected forms of a word to one common form, so
that a search for `connecting` also finds `connected` and `connection`.
Snowball is a small language for writing stemming algorithms and the
family of algorithms written in it, created by Martin Porter and
published at [snowballstem.org](https://snowballstem.org/). The
algorithm for English is
[Porter2](https://snowballstem.org/algorithms/english/stemmer.html),
the 2002 revision of Porter's
["An algorithm for suffix stripping"](https://tartarus.org/martin/PorterStemmer/def.txt),
*Program* 14(3), 1980. This package brings Snowball stemming to
novo-lang. The reference implementations are the Rust crate
[`rust-stemmers`](https://docs.rs/rust-stemmers) and the Python package
[`snowballstemmer`](https://pypi.org/project/snowballstemmer/).

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What stemming is

A **stem** is the string an algorithm reduces a word to. Two words that
should be treated as the same term produce the same stem, and a search
engine stores the stem rather than the word. `connect`, `connected`,
`connecting`, `connection` and `connections` all stem to `connect`,
so a query for any of them finds documents containing any of the
others.

**A stem is not a word.** The only job of a stem is that related words
produce the same one, and an algorithm that achieves that frequently
produces a string that is not in the language. Porter2 stems `arguing`
to `argu` and `ugly` to `ugli`. A program shows the original text to a
person and uses the stem only to find it.

Stemming is **not** lemmatisation. A lemmatiser uses a dictionary and
a part-of-speech tag to find the citation form of a word, answering
`be` for `was`. A stemmer applies rules to letters, knows no
vocabulary, and answers `wa`. Stemming is faster by orders of
magnitude and is what search indexes use.

Every Snowball algorithm is built out of the same two ideas, defined on
the Snowball site's
[introduction page](https://snowballstem.org/texts/introduction.html).
**R1** is the region of the word after the first non-vowel that follows
a vowel, or the end of the word when there is no such non-vowel. **R2**
is the same rule applied again inside R1. For `beautiful`, R1 is `iful`
and R2 is `ul`. Nearly every rule in nearly every algorithm is then of
the form "remove this suffix if it lies in R1", and R1 and R2 are
therefore the reason a suffix was or was not removed.

A **short syllable** is a vowel followed by a non-vowel other than `w`,
`x` or a marked `Y` and preceded by a non-vowel, or a vowel at the
start of the word followed by a non-vowel. A word is **short** when it
ends in a short syllable and R1 is empty. `bed`, `shed` and `shred` are
short; `bead`, `embed` and `beds` are not. The rule decides whether
`hoping` stems to `hope` and `hopping` to `hop`.

Snowball publishes an algorithm for about thirty languages. This
release ports two of them, both English: Porter2, and Porter's
original 1980 algorithm. The two disagree on ordinary words —
Porter2 stems `ties` to `tie` and the 1980 algorithm stems it to `ti` —
so both are named and a program can say which it used.

## Install

```
novo pkg add stemmer-nv
```

## Example

```novo
use stemcase
use stemerror
use stemlang
use stemword

fn main() [io]
    // Check the language once, where the configuration is read, so a
    // language this release cannot stem stops the program here.
    match stemlang.require(StemEnglish)
        Err(e) => println(stemerror.message(e))
        Ok(l)  =>
            for w in ["Ponies", "hopping", "arguing", "news"]
                // Lower-case the word. `stem` refuses an upper-case
                // letter rather than answering the word unchanged.
                match stemword.stem(l, stemcase.fold_ascii(w))
                    Err(e) => println(stemerror.message(e))
                    Ok(s)  => println("${w} -> ${s}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: stemmer-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `stemerror` | The four refusals, a stable code for each, and which of them a start-up check would catch. |
| `stemlang` | The enum of every language Snowball publishes an algorithm for, and which of them this release can stem. |
| `stemcase` | The case a word must arrive in, the ASCII fold, and which languages need more than one. |
| `stemregion` | R1, R2, the vowel test, the short-syllable rule and the marking of a consonant `y`. |
| `stemporter` | The English algorithm: its eight passes as values, and its three exception tables as lists. |
| `stemword` | Stemming a word, stemming a list of words, and the explanation of how one stem was reached. |

## How to choose an entry point

**`stemword.stem` is the ordinary call.** One lower-case word in, one
stem out. An indexer uses this and nothing else.

**`stemword.stem_all` takes a tokenized document** and answers the
stems in the same order. It refuses the whole list on the first word it
cannot stem, and names that word. A caller that would rather skip a bad
token calls `stem` per token.

**`stemword.explain` answers the stem and how it was reached** — the
passes that changed the word, and whether an exception table decided
it. Use it when a person asks why a word behaved as it did. Do not use
it in an indexer: it allocates a trace for every word.

**`stemlang.require` is the start-up check.** Call it where the
configuration is read, so that an unported language stops the program
before the first document rather than inside a worker.

**`stemregion` and `stemporter` are for reading the algorithm**, not
for stemming. They are published so that a caller comparing this
package against another implementation can find where the two diverge.

## The rules a user needs

1. **A stem is not a word.** `arguing` stems to `argu`, `ugly` stems to
   `ugli`, `happy` stems to `happi`. Show the original text to a person
   and use the stem only as an index key.
2. **Stemming is a pure function of the word.** The same word always
   stems to the same string, so a caller may memoise as it likes, and
   nothing here reads a file, a locale or a clock.
3. **`stem` does not lower-case.** A word holding an upper-case ASCII
   letter is refused with `StemNeedsLowercase`. The reference
   implementations answer such a word unchanged, which writes a second
   unrelated term into an index and reports nothing.
   `stemcase.fold_ascii` is the fold for English and every other
   Latin-alphabet language on the list.
4. **Nine languages need more than an ASCII fold.**
   `stemcase.needs_unicode_fold` names them. Turkish is one of them
   although its alphabet is Latin: it lowers `I` to `ı` and `İ` to `i`,
   and every other language's rule gets both wrong.
5. **An unported language is refused, never stemmed as English.**
   `stemword.stem` answers `StemUnsupportedLanguage`. English stemming
   applied to French produces a half-working index that no test over
   English vectors can detect.
6. **`StemLang` holds every Snowball language, not every available
   one.** `stemlang.available` answers the list this release can stem;
   for 0.1.0 that is `StemEnglish` and `StemPorter`.
   `stemlang.is_available` answers for one language.
7. **The empty string and a phrase are both refused.** Snowball stems
   one word; a phrase handed to it stems to itself. Tokenizing is the
   caller's job.
8. **`StemEnglish` and `StemPorter` are different algorithms.**
   `ties` stems to `tie` under the first and `ti` under the second.
   `stemword.algorithm_id` names which one a stem came from.
9. **An index is comparable with itself only.** Two releases may stem
   the same word differently, and a query stemmed by a new algorithm
   against terms stemmed by an old one does not fail, it misses.
   Record `stemword.algorithm_id` beside the index.
10. **Eleven words are stemmed by table** rather than by rule, from the
    exception list on the Snowball English page: `skis`, `skies`,
    `dying`, `lying`, `tying`, `idly`, `gently`, `ugly`, `early`,
    `only`, `singly`. `stemporter.exception_for` answers the table.
11. **Seven words are their own stem**: `sky`, `news`, `howe`, `atlas`,
    `cosmos`, `bias`, `andes`. `stemporter.is_invariant` is the call
    that answers why one of them did not move.
12. **Eight more words stop after step 1a**: `inning`, `outing`,
    `canning`, `herring`, `earring`, `proceed`, `exceed`, `succeed`.
13. **R1 is set explicitly after `gener`, `commun` and `arsen`.**
    Without the override the whole `generate`, `general`, `generic`,
    `generous` family collapses to `gener`.

## Running on a microcontroller

This package makes no device claim and ships no device probe. Every
function in it performs no input or output and holds no state between
calls, so the modules build for a microcontroller. The memory a stem
needs is the word and one string the length of the word. The tables
compiled in for English are twenty-six words and about forty suffixes,
which is under two kilobytes.

## What is not included

- **Lemmatisation.** A lemmatiser needs a dictionary and a
  part-of-speech tag. This package applies rules to letters.
- **Tokenizing.** A word arrives already split out of the text.
  `stemword.stem` refuses an argument holding a space.
- **Lower-casing.** See rule 3.
- **Stop words.** Removing `the` and `of` is a decision about a corpus,
  not about a word.
- **Twenty-eight of the thirty Snowball languages.** They are declared
  in `StemLang` and `stemlang.available` does not list them. See rule
  6.
- **A Snowball compiler.** Snowball is a language with its own compiler
  that generates C, Java, Rust and Python. This package ports the
  generated algorithms by hand; it does not read `.sbl` files.
- **Reading a word list.** Opening a file costs `[fs]`, and this
  package declares no effects.

## Related packages

- [spellcheck-nv](https://novo-lang.org/packages/spellcheck-nv)
  corrects a misspelt word against a dictionary. Take it when the input
  may be wrong; take this package when the input is right and the
  forms differ.
- [fuzzy-nv](https://novo-lang.org/packages/fuzzy-nv) scores
  approximate matches for a picker. It measures how close two strings
  are; this package makes two strings equal.
- [bm25-nv](https://novo-lang.org/packages/bm25-nv) ranks documents
  over terms that have already been reduced to a common form. Stem
  first with this package, then rank with that one.
- [unicode-nv](https://novo-lang.org/packages/unicode-nv) has the case
  mapping this package does not carry. A caller stemming a language
  `stemcase.needs_unicode_fold` names folds the word with it first.

## Test vectors

```bash
novo test tests/stemword_tests.nv      # the voc.txt pairs, and the four refusals
novo test tests/stemporter_tests.nv    # the eight passes and the three exception tables
novo test tests/stemregion_tests.nv    # R1, R2, the short-syllable rule and the `y` marking
novo test tests/stemcover_tests.nv     # the language set, the case contract, the error taxonomy
```

The normative source is the Snowball site: the
[introduction page](https://snowballstem.org/texts/introduction.html)
for R1, R2 and the short-syllable rule, and the
[English (Porter2) page](https://snowballstem.org/algorithms/english/stemmer.html)
for the eight passes, the three exception tables and the `gener`
override. The word pairs are a selection from the `voc.txt` and
`output.txt` files the Snowball project publishes beside each
algorithm, chosen so that each pair exercises one rule. The 1980
algorithm's vectors come from Porter's own
[definition page](https://tartarus.org/martin/PorterStemmer/def.txt).

The suite asserts the step-1a plural table on `caresses`, `ponies`,
`ties`, `caress`, `cats`, `gaps`, `kiwis`, `gas` and `this`; the double
letter and short-word halves of step 1b on `hopping` and `hoping`; the
`y` rule on `happy`, `cry`, `by` and `say`; the R2 test that leaves
`argument` alone beside the `arguing` that becomes `argu`; the regions
of `beautiful`, `beauty` and `beau`; the short-word rule on `bed`,
`shed`, `shred`, `bead`, `embed` and `beds`; all eleven exceptions, all
seven invariants and the eight post-step-1a words; that `ties` stems
differently under the two English algorithms; and that an upper-case
word, an empty word, a phrase and an unported language are each
refused with their own code.

The tests compile today and fail at run, each on the
`not implemented: stemmer-nv.<module>.<fn>` panic that is its body.
That is the expected state of an interface release. They turn green one
at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `stemerror.StemError` and the other public types | the types are declared |
| `stemerror.message`, `.code`, `.is_configuration_fault` | no |
| `stemlang.all`, `.available`, `.is_available`, `.require` | no |
| `stemlang.lang_name`, `.lang_named`, `.lang_code` | no |
| `stemcase.fold_ascii`, `.is_prepared` | no |
| `stemcase.needs_unicode_fold`, `.fold_note` | no |
| `stemregion.at`, `.regions`, `.is_vowel` | no |
| `stemregion.mark_consonant_y`, `.is_short_syllable`, `.is_short_word` | no |
| `stemregion.english_r1_prefixes` | no |
| `stemporter.steps`, `.step_name`, `.step_note`, `.apply_step` | no |
| `stemporter.exception`, `.exceptions`, `.exception_for` | no |
| `stemporter.invariants`, `.is_invariant`, `.post_1a_invariants` | no |
| `stemword.stem`, `.stem_all`, `.explain` | no |
| `stemword.word`, `.text`, `.moved`, `.algorithm_id` | no |
| English (Porter2) | no |
| English (Porter 1980) | no |
| The other twenty-eight Snowball languages | not in this release; see rule 6 |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
