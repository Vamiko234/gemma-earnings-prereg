# gemma-earnings-study - public pre-registration

This repository exists for one purpose: to place the pre-registration of an empirical
finance study **in public, before the results exist**, so that the date can be verified by
someone other than the author.

**The study.** Does an open-weight language model (Gemma 4 26B A4B QAT, thinking enabled),
reading only the scrubbed text of an 8-K Ex-99.1 earnings release, predict post-earnings
abnormal returns better than SUE (a seasonal random walk)? The model's training cutoff is
January 2025, so events from February 2025 onward are the clean test and earlier events are
an explicitly contaminated comparison.

**Status when this was published: no confirmatory result has been scored.** What is
registered here is the plan, not an outcome.

## What each file is

| File | What it is |
|---|---|
| `prereg.md` | The pre-registration. Hypotheses, sample construction, the scoring rule, the analysis, and the freeze record (section 13). |
| `deviations.md` | Every departure from the pre-registration, numbered D-001 onward, each with what changed, why, and whether a re-run was required. Written as the study runs, including the mistakes. |
| `scrubber_freeze.json` | The frozen anonymisation configuration: stop-lists, their hashes, the audit history, and the hash of every frozen source file. |
| `excluded_event_ids.csv` | The 41 events excluded from every confirmatory analysis: 20 pilot events, 20 pre-cutoff probe events, and 1 pipeline test. Fixed before confirmatory scoring begins. |
| `FREEZE_HASHES.txt` | Every hash in one place, with the command to reproduce it. |

## Which commit this corresponds to

Everything here is mirrored from the private working repository
[github.com/Vamiko234/gemma-earnings-study](https://github.com/Vamiko234/gemma-earnings-study) at commit:

```
f70a0cc909252da69d8f6013698856321c5079dd
```

Mirrored on 2026-09-26.

The working repository is private because it contains the model prompts, the extracted
release text, and the raw model outputs. Nothing in this repository depends on that: the
files here are complete documents, and the hashes below bind the private code to them.

## How to verify the hashes

**The files in this repository.** Every published file is listed in `FREEZE_HASHES.txt` with
its SHA-256. On any machine:

```
sha256sum prereg.md deviations.md scrubber_freeze.json excluded_event_ids.csv
```

Compare against `FREEZE_HASHES.txt`. If a line matches, that file is byte-identical to what
was published at the commit named above.

**The frozen source files.** These are not published, only their hashes are. The convention
is the SHA-256 of the file **as committed**, that is with LF line endings:

```
git show <commit>:<path> | sha256sum
```

A hash taken from a checked-out file on Windows will not match, because the working
repository uses `core.autocrlf=true`, which stores LF and checks out CRLF. This is the
reason the convention is stated explicitly rather than assumed.

Publishing the hashes without the code is a deliberate trade. It fixes the content of the
analysis code now, while the results do not yet exist, and defers disclosure until
publication. It does mean a reader today must take on trust that the hashes correspond to
working code - that part becomes checkable when the private repository is opened.

## Independent timestamps

A private repository is off the author's machine but is not independently verifiable: only
the author can show it. This repository is archived by two services that are independent of
both the author and GitHub, and whose archive dates are their own records:

- **Internet Archive (Wayback Machine)** - archives the rendered pages
- **Software Heritage** - archives the repository and its full commit history

Archive links and dates are recorded in `prereg.md` section 13.3.

The Open Science Framework is not used. OSF requires account holders to be 18 and the author
is 15. A parent-hosted OSF registration may be added later; if it is, it will be recorded as
an additional timestamp in section 13.3, never as a replacement for these.

## Licence and contents

Documentation only. No earnings-release text, no model prompts, no model outputs, no
credentials, and no third-party data are included. Company and product names appearing in
`deviations.md` are referenced descriptively, as examples of what the anonymiser can and
cannot remove.
