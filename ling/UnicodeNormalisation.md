# Unicode normalisation for Sámi text

*How to fix decomposed Sámi letters before linguistic analysis.*

## Why normalisation matters

The same written letter can have more than one Unicode representation. For example, `á` can be stored as a single code point (`U+00E1`, LATIN SMALL LETTER A WITH ACUTE) or as `a` (`U+0061`) followed by COMBINING ACUTE ACCENT (`U+0301`). Likewise, North Sámi `č` can be stored as `U+010D` or as `c` (`U+0063`) followed by COMBINING CARON (`U+030C`).

These representations look alike, but their underlying code point sequences differ. Consequently, raw string equality, dictionary lookups and morphological analysis can behave differently unless the input and linguistic resources use a consistent normalisation form.

| Displayed letter | Precomposed representation | Decomposed representation |
| --- | --- | --- |
| `á` | `U+00E1` | `U+0061 U+0301` |
| `č` | `U+010D` | `U+0063 U+030C` |

## The four Unicode normalisation forms

Unicode defines four standard normalisation forms:

| Form | Meaning | What it does |
| --- | --- | --- |
| **NFD** | Canonical decomposition | Decomposes canonically equivalent sequences and puts combining marks in canonical order. |
| **NFC** | Canonical decomposition followed by composition | Produces a canonical composed representation where Unicode composition rules permit it. |
| **NFKD** | Compatibility decomposition | Also decomposes compatibility-equivalent characters, potentially removing meaningful distinctions. |
| **NFKC** | Compatibility decomposition followed by canonical composition | Applies compatibility mappings, then canonical composition. |

For the specific task of converting decomposed Sámi letters to their canonically equivalent composed form, **NFC is generally the appropriate choice**. For example, it converts `c` + COMBINING CARON into `č` without otherwise applying compatibility mappings.

**NFKC is not merely a stronger form of NFC.** It can change additional characters: for instance, `①` remains `①` under NFC but becomes `1` under NFKC. That may be desirable for a deliberately broad comparison key, but not when preserving the distinctions in original corpus text matters.

A composed representation also does **not** mean that every user-perceived character becomes a single Unicode code point. Some combining sequences do not have a precomposed equivalent.

## Recommended command: convert input text to NFC

The North Sámi analysers described in this documentation expect composed letter representations. When the input contains decomposed Sámi letters, normalise the UTF-8 data before passing it to an analyser:

```sh
uconv -f UTF-8 -t UTF-8 -x Any-NFC < infile.txt > outfile.txt
```

This reads `infile.txt` and writes the canonically normalised result to `outfile.txt`; the input file is left unchanged. The `uconv` command is provided by ICU, although the package name and installation path depend on your operating system.

Check that the command is available with:

```sh
uconv --help
```

For details, consult `man uconv` or the ICU documentation linked below. If a corpus-processing pipeline deliberately uses **NFKC**, do not change that pipeline without checking its expected treatment of compatibility-equivalent characters. The older version of this page used:

```sh
uconv -f UTF-8 -t UTF-8 -x Any-NFKC < infile.txt > outfile.txt
```

That command still has a valid purpose when compatibility normalisation is explicitly required; it should not be presented as the only way to compose Sámi letters.

## Verify the result

Python's standard library provides a convenient independent check of the expected Unicode behaviour:

```python
import unicodedata as ud

assert ud.normalize("NFC", "a\u0301") == "\u00E1"  # a + acute -> á
assert ud.normalize("NFC", "c\u030C") == "\u010D"  # c + caron -> č
assert ud.normalize("NFD", "\u010D") == "c\u030C"

# NFC preserves compatibility distinctions that NFKC may remove.
assert ud.normalize("NFC", "\u2460") == "\u2460"  # ①
assert ud.normalize("NFKC", "\u2460") == "1"

print("Unicode normalisation checks passed")
```

For an actual corpus, verify that the output can still be decoded as UTF-8 and test representative tokens containing Sámi diacritics. Compare the results of the morphological analyser before and after normalisation rather than assuming that a visually identical token has the same underlying representation.

## Corpus-processing considerations

- **Normalise consistently:** apply the chosen form at a documented input boundary. Matching data and the linguistic resources being queried must use compatible conventions.
- **Keep the original data when needed:** retain unmodified source files for reproducibility and provenance, especially when processing archived corpora.
- **Separate normalisation from other transformations:** case folding, transliteration, punctuation changes, and removal of control characters are separate policy decisions. Neither NFC nor NFKC is a spelling corrector.
- **Do not silently discard malformed input:** character decoding errors should be logged or handled according to an explicit corpus policy.
- **Test the actual downstream result:** successful `uconv` execution does not by itself establish that every corpus token will be accepted by the analyser.

The previously linked [example normalisation script](https://github.com/redpony/cdec/blob/master/corpus/utf8-normalize.sh) uses NFKC and also performs additional cleaning, including control-character handling. It is an example of a broader corpus-cleaning pipeline, **not** an interchangeable substitute for the NFC command above.

## Normalisation and approximate matching

Canonical normalisation should usually happen **before** character-by-character comparisons when canonically equivalent representations are intended to match. For example, comparing the code point sequences `c` + COMBINING CARON and `č` without normalisation can report a difference even though both represent the same written letter. Normalising both to NFC first avoids that discrepancy.

This is particularly relevant when edit-distance methods are used in spelling correction or corpus matching. Normalisation, however, does not make genuinely different Sámi letters equivalent and does not replace language-specific error models. Further background on the distinction between Unicode normalisation and text matching:

https://www.levenshtein.net/text-normalization-and-transliteration

## References

- Unicode Standard Annex #15, *Unicode Normalization Forms*: https://www.unicode.org/reports/tr15/
- ICU, *General Transforms* (`Any-NFC`, `Any-NFKC`): https://unicode-org.github.io/icu/userguide/transforms/general/
- `uconv` command-line documentation: https://man7.org/linux/man-pages/man1/uconv.1.html
