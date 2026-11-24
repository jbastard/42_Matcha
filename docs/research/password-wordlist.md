# Research: common English words list for password rejection

Ticket: #4. Feeds decision ticket #18.

Question: which openly licensed lists of commonly used English words and common passwords can back R-AUTH-02, how do they compare (size, licence), and how do other projects apply them (exact match, substring, normalisation)?

Rules in scope: R-AUTH-02 (commonly used English words rejected as passwords), R-AUTH-01 (secure password), R-SEC-05 (every input validated), R-GEN-02 / R-GEN-03 (libraries allowed, frameworks without validators), R-GEN-04 (anything not explicitly allowed is forbidden).

Facts checked on 2026-11-24. Repo metadata (licence SPDX id) comes from `gh api repos/<owner>/<repo>`.

## 1. What NIST SP 800-63B says

Source: https://pages.nist.gov/800-63-4/sp800-63b.html (revision 4 per URL path; the fetched page did not show a status or publication date, unverified).

- Sec. 3.1.1.2: verifiers SHALL compare the prospective password against a blocklist of "commonly used, expected, or compromised passwords". Examples given: passwords from breach corpuses, dictionary words, context-specific words (service name, username and derivatives).
- Sec. 3.1.1.2: "The entire password SHALL be subject to comparison, not substrings or words that might be contained therein." So NIST guidance is whole-password match, not substring.
- Sec. 3.1.1.2: the blocklist SHOULD be large enough to stop passwords attackers would guess before the throttling limit; "Excessively large blocklists are of little incremental security benefit".
- Sec. 3.1.1.2: on rejection the CSP SHALL require another password and SHALL give the reason.
- Sec. 3.1.1.2: no other composition rules (no mandatory character classes) SHALL be imposed.
- Sec. 3.1.1.2: minimum length 15 for single-factor passwords, 8 when used only in multi-factor flows. Matcha login is username + password only (R-AUTH-04), so it is single-factor.
- Sec. 3.1.1.2: Unicode passwords SHOULD be NFC-normalised before hashing.

Consequence for R-AUTH-02: a 15-character minimum alone already rejects nearly every single dictionary word, but R-AUTH-02 is a subject rule and must be visibly enforced regardless of the length policy.

## 2. Candidate English word lists

| Source | Size | Licence | Notes |
|---|---|---|---|
| zxcvbn-ts `@zxcvbn-ts/language-en` `commonWords.json` | 55,830 entries, 558 kB JSON | package MIT (`packages/languages/en/package.json`, v4.1.2) | Frequency-ranked. Generated from OpenSubtitles 2024 frequency data (`data-scripts/lists.ts`: `https://object.pouta.csc.fi/OPUS-OpenSubtitles/v2024/freq/en.freq.gz`), `minOccurrences: 1000` (`data-scripts/_generators/CommonWordsGenerator.ts`). Contains fragments like `'s`, `'t`. Licence of the upstream OpenSubtitles frequency data: not verified. |
| zxcvbn-ts `@zxcvbn-ts/language-en` `wikipedia.json` | 29,782 entries, 335 kB | MIT (same package) | Generation source not found in `lists.ts`; a `data-scripts/wikiExtractor` exists. Upstream licence: not verified (Wikipedia text is CC BY-SA). |
| dropbox/zxcvbn (original) | `us_tv_and_film` 30,000, `english_wikipedia` 30,000 (cutoffs in `data-scripts/build_frequency_lists.py`) | repo MIT | Bundle "about 400kB gzipped or 820kB uncompressed, most of which is dictionaries" (README). Last push 2024-08-19; the TypeScript port above is the maintained one (last push 2026-11-14). |
| SCOWL / ESDB (`en-wl/wordlist`, http://wordlist.aspell.net) | Graded by size level (10 to 95); not counted here | Permissive custom notice (`Copyright` file): use, copy, modify, distribute, sell allowed with notice. Lists larger than size 80 also carry the UKACD notice. | Spelling dictionary, not frequency-ranked by usage in the strict sense; small size levels (10-35) approximate "common" words. Exact word counts per level: not verified. |
| first20hours/google-10000-english | 10,000 (plus `20k.txt`) | No SPDX licence (`NOASSERTION`). `LICENSE.md`: data derived from the Google Web Trillion Word Corpus (LDC); "I do not recommend using this data for commercial purposes without licensing it from the Linguistic Data Consortium." | Not openly licensed. Avoid. |
| rspeer/wordfreq | Python library, large | Code Apache-2.0, data CC BY-SA 4.0 (README "License") | Author states the data should not be exported to CSV or another attribution-less format (README FAQ). Share-alike plus Python-only: poor fit. |
| dwyl/english-words | about 479k words (count from repo README, not re-verified) | Unlicense | Exhaustive dictionary, not "commonly used". Includes obscure words; fine as an exact-match list but oversized for the intent of R-AUTH-02. |

## 3. Candidate common-password lists

| Source | Size | Licence | Notes |
|---|---|---|---|
| SecLists `Passwords/Common-Credentials/xato-net-10-million-passwords-*.txt` | variants of 10, 100, ..., 1,000,000 lines; full file 5,189,454 unique (folder README) | repo MIT (`danielmiessler/SecLists`) | Ranked by frequency from Mark Burnett's 10M corpus. Same corpus feeds both zxcvbn and zxcvbn-ts password lists. |
| SecLists `100k-most-used-passwords-NCSC.txt` | 836 kB, about 100k lines | repo MIT; original NCSC/HIBP release | Original NCSC download link reported gone (search results, not verified); SecLists is the mirror. |
| SecLists `Pwdb_top-*`, `probable-v2_top-*`, `10k-most-common.txt` | 1k to 10M | repo MIT | Other aggregations; `probable-v2` from berzerk0/Probable-Wordlists (2018). |
| zxcvbn-ts `@zxcvbn-ts/language-common` `passwords.json` | 49,233 entries, 487 kB | MIT (v4.2.0) | Generated from the Xato file; `PasswordGenerator.ts` drops non-ASCII entries and entries zxcvbn already guesses faster than their rank. |
| Have I Been Pwned Pwned Passwords | hundreds of millions of SHA-1 hashes (exact count not stated on page) | "no licencing or attribution requirements on the Pwned Passwords API" (https://haveibeenpwned.com/API/v3, License section) | k-anonymity range API `GET https://api.pwnedpasswords.com/range/{5 hex SHA-1 prefix}`, no API key, optional `Add-Padding: true`. Network dependency at registration time; offline downloader exists (https://haveibeenpwned.com/Passwords). |

## 4. How other projects apply the lists

- Django `CommonPasswordValidator` (`django/contrib/auth/password_validation.py`, https://github.com/django/django): ships 20,000 common passwords (Royce Williams gist, licence of the gist not verified), lowercased and deduplicated. Check is `password.lower().strip() in self.passwords`: whole-password exact match after lowercase and trim. Separate `UserAttributeSimilarityValidator` compares against user attributes and their components.
- zxcvbn (`src/matching.coffee`, `dictionary_match`): lowercases the password, then looks up every substring `password_lower[i..j]` in each ranked dictionary. Matches feed a guess-count estimate and a 0-4 score; a dictionary word inside a password lowers the score instead of causing a hard reject. `user_inputs` adds per-user words (README).
- zxcvbn-ts (`docs/guide/matcher/README.md`): dictionary matcher variants are plain, reverse, l33t (including extended l33t such as `|_|` to `u`) and diceware; source imports `toLowerCase` (`packages/libraries/main/src/matcher/dictionary/matching.ts`). An optional async `@zxcvbn-ts/matcher-pwned` queries HIBP.
- NIST (section 1): whole-password match, no substring rejection.
- Keycloak password blacklist policy: could not read the relevant docs section, not verified.

## 5. Matching strategies compared

| Strategy | Effect | Risk |
|---|---|---|
| Exact match after NFC + lowercase + trim | Matches NIST and Django. Rejects `sunshine`, `Password`. | Accepts `Sunshine1!`. An evaluator testing R-AUTH-02 with a word plus a digit would see it accepted. |
| Exact match after also stripping leading/trailing digits and symbols, and l33t-folding | Rejects `Sunshine1!`, `p4ssw0rd`. Still whole-password, not substring. | Small custom code; must keep the list lowercased and ASCII-folded the same way. |
| Substring match (any dictionary word inside) | Rejects most passphrases (`correcthorsebatterystaple`), contradicts NIST Sec. 3.1.1.2. | With a 50k word list almost every password containing letters fails. Not recommended. |
| Strength score (zxcvbn-ts, reject below a threshold) | Covers words, l33t, reversal, keyboard walks, dates; gives feedback text. | Adds a dependency of several hundred kB; scoring is less obviously "this is an English word" to an evaluator. Whether an evaluator counts it as a "validator" under R-GEN-02 is unclear (R-GEN-02 limits frameworks, R-GEN-03 allows libraries); not verified against defence practice. |

## 6. Recommendation for decision ticket #18

1. Lists: combine a frequency-ranked English common-words list and a common-passwords list, both MIT-packaged:
   - words: `@zxcvbn-ts/language-en` `commonWords.json` (55,830 entries), or a SCOWL small size level if the team prefers a list with a clearer upstream licence;
   - passwords: SecLists `xato-net-10-million-passwords-100000.txt` or `100k-most-used-passwords-NCSC.txt` (MIT repo), or `@zxcvbn-ts/language-common` `passwords.json`.
   Keep the combined file in the repo (R-GEN-12) with the licence notices. Avoid google-10000-english (LDC terms) and wordfreq (CC BY-SA, no export).
2. Matching: whole-password exact match on a normalised form (NFC, lowercase, trim, strip leading/trailing digits and symbols, fold common l33t substitutions). This meets NIST's "entire password" rule while catching the obvious `Word123!` probe an evaluator would try for R-AUTH-02. Do not do substring matching.
3. Add context words: reject the username, first name, last name and email local part (NIST context-specific words; Django `UserAttributeSimilarityValidator`).
4. Return an explicit rejection reason to the user (NIST Sec. 3.1.1.2 SHALL).
5. Run the check server-side (R-SEC-05); a client-side copy is optional UX only.
6. Optional: zxcvbn-ts as an additional strength meter, decided separately given the R-GEN-02 ambiguity. HIBP adds a network dependency at registration; skip unless the team wants it.

## Unverified items

- NIST SP 800-63B rev 4 status and publication date (not shown on the fetched page).
- Upstream licences of OpenSubtitles frequency data and of the Wikipedia-derived list behind zxcvbn-ts English data.
- Royce Williams gist licence (Django list source).
- SCOWL word counts per size level; dwyl/english-words exact count.
- Availability of the original NCSC 100k download.
- Keycloak blacklist policy behaviour.
