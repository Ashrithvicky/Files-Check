# Comprehensive Prompt Preparation Document

**Selected PR:** [#1808 — Add MusicBrainz ID Option to Importer](https://github.com/beetbox/beets/pull/1808)
**Repository:** [beetbox/beets](https://github.com/beetbox/beets)

---

## 3.1.1 Repository Context

Beets is a command-line music library manager written entirely in Python. It is designed for people who have large personal music collections stored as audio files on disk — the kind of users who care deeply about having correct, consistent metadata (artist names, album titles, track numbers, genres) across their entire library. Think of someone with thousands of FLAC or MP3 files scattered across folders, many with incomplete or inconsistent tags.

The core problem beets solves is the gap between messy real-world music files and a cleanly organized, properly tagged library. When a user runs `beet import /path/to/music`, beets reads the existing file tags, queries online databases like MusicBrainz to find the correct release, presents candidates to the user for confirmation, and then writes the corrected metadata back into the files. It can also rename and reorganize files on disk according to configurable path templates.

The architecture is built around a multi-stage import pipeline that processes albums through discrete steps: reading files, looking up candidates, asking the user for confirmation, applying metadata, and moving files. This pipeline uses Python generators acting as coroutines, which allows stages to be composed together flexibly.

Beyond the core import workflow, beets has a plugin system with over 70 built-in plugins. These plugins handle everything from fetching lyrics and album art to running a web server that exposes the library over HTTP. Plugins hook into the system through an event-based publish-subscribe pattern — they register listeners for events like `import_task_start` or `item_imported` and can modify behavior at those points.

The project uses Poetry for dependency management, SQLite for its database layer (wrapped by a custom ORM called `dbcore`), and `confuse` for YAML-based configuration. It targets music enthusiasts, archivists, and anyone managing large audio collections who wants automated, accurate tagging.

---

## 3.1.2 Pull Request Description

PR #1808 introduces a command-line option (`-S` / `--search-id`) that lets users pass one or more MusicBrainz release or recording IDs directly to the `beet import` command. Before this change, every import had to go through a blind search: beets would read whatever metadata was already in the files, send a text-based query to MusicBrainz ("artist name + album name"), and get back a ranked list of candidate releases. The user then picked the right one from that list.

This approach had two concrete problems. First, for albums with common names or missing tags, the search would return dozens of irrelevant candidates, forcing the user to scroll through pages of wrong answers. Second, for very large albums (box sets with 50+ tracks), the candidate matching process was extremely slow because beets had to compute edit distances between every track in the album and every track in each candidate release.

The new behavior is straightforward: if the user already knows the exact MusicBrainz release ID (which they can find on the MusicBrainz website), they pass it with `-S <ID>`. Beets then skips the broad text search entirely and fetches metadata only for that specific release. The matching is near-instant because there is only one candidate to evaluate.

The change touches several layers. At the CLI level, a new `--search-id` argument is added with `action='append'` so multiple IDs can be specified. At the pipeline level, these IDs are read from the session config and passed down to the `lookup_candidates` stage. At the autotag level, the `tag_album()` and `tag_item()` functions had their signatures changed from accepting a single `search_id` string to accepting a `search_ids` list. The interactive "enter Id" prompt was also updated to accept multiple space-separated IDs, so users can enter IDs during the import session itself, not just from the command line.

---

## 3.1.3 Acceptance Criteria

✓ **AC-1: CLI Argument Registration.** When a user runs `beet import -S <MUSICBRAINZ_ID> /path/to/album`, the `<MUSICBRAINZ_ID>` value should be captured and stored in the import session's configuration under the key `search_ids` as a list.

✓ **AC-2: Multiple IDs via CLI.** When a user specifies multiple `-S` flags (e.g., `beet import -S ID1 -S ID2 /path/`), all provided IDs should be collected into a single list and passed to the candidate lookup, so the system returns candidates for every specified ID.

✓ **AC-3: Restricted Candidate Lookup.** When `search_ids` is a non-empty list, the `tag_album()` function should query metadata backends only for those specific IDs (using `metadata_plugins.albums_for_ids()`), and must NOT perform the default text-based keyword search against MusicBrainz.

✓ **AC-4: Singleton Support.** When importing singletons (single tracks via `beet import -s -S <RECORDING_ID>`), the `tag_item()` function should use the provided IDs to look up specific recordings, restrict candidates to those IDs, and not fall through to the broader keyword search.

✓ **AC-5: Interactive "enter Id" Prompt.** When the user selects the "enter Id" (`i`) option during an interactive import session and types one or more space-separated IDs, the system should split the input on whitespace and pass all resulting IDs to `tag_album()` or `tag_item()` via the `search_ids` parameter.

✓ **AC-6: Backward Compatibility.** When no `-S` flag is provided and `search_ids` is empty, the import pipeline should behave identically to the pre-change behavior — performing the standard metadata-based text search and ID consensus lookup.

✓ **AC-7: Distance-Based Sorting.** When multiple IDs are provided (either via CLI or the interactive prompt), all resulting candidates should be sorted by their computed distance score (ascending), and the recommendation should be calculated from this sorted list, just as it is for text-search candidates.

✓ **AC-8: Metadata Backend Agnostic.** The ID-based lookup should be routed through the existing `metadata_plugins.albums_for_ids()` and `metadata_plugins.tracks_for_ids()` hook functions, so that any installed metadata source plugin (MusicBrainz, Discogs, Spotify) that implements `album_for_id()` or `track_for_id()` will automatically work with the feature without code changes.

---

## 3.1.4 Edge Cases

### Edge Case 1: Invalid or Non-Existent MusicBrainz ID

**Scenario:** The user provides an ID that does not correspond to any release or recording in MusicBrainz (e.g., a typo, a random UUID, or an ID from a different service).

**Expected Handling:** The `albums_for_ids()` / `tracks_for_ids()` hook should return an empty result for that ID. If all supplied IDs are invalid, `tag_album()` should return an empty candidates list with a `Recommendation.none`, and `tag_item()` should return `Proposal([], Recommendation.none)`. The importer should then display "No matching release found" and present the user with fallback options (Skip, Use as-is, Enter search, etc.) — the same behavior as when a normal text search yields zero results. The system must not crash or raise an unhandled exception.

### Edge Case 2: Mixed Valid and Invalid IDs in a Multi-ID Query

**Scenario:** The user passes multiple IDs via `-S ID1 -S ID2 -S ID3`, where `ID1` and `ID3` are valid MusicBrainz release IDs but `ID2` is invalid or returns nothing.

**Expected Handling:** The system should gracefully skip `ID2` (it simply won't produce a candidate) and return candidates only for `ID1` and `ID3`. The two valid candidates should be sorted by distance and a recommendation should be computed normally. No error should be raised for the missing ID — the loop in `tag_album()` that iterates over `metadata_plugins.albums_for_ids(search_ids)` naturally handles this because the hook generator simply yields nothing for unknown IDs.

### Edge Case 3: ID Belongs to a Different Entity Type

**Scenario:** The user is importing a full album but accidentally provides a MusicBrainz *recording* ID (a track-level ID) instead of a *release* ID (album-level ID), or vice versa.

**Expected Handling:** When `tag_album()` calls `metadata_plugins.albums_for_ids()` with a recording ID, MusicBrainz will not find a matching release, so no candidate is returned. The user sees "No matching release found" and can retry. Similarly, when `tag_item()` is called with a release ID instead of a recording ID, `metadata_plugins.tracks_for_ids()` returns nothing. The system should not crash, and the user should be given the standard fallback prompt to enter a correct ID, do a manual search, or skip.

### Edge Case 4: Empty String or Whitespace-Only Input at the Interactive Prompt

**Scenario:** During an interactive import, the user selects "enter Id" and presses Enter without typing anything, or types only spaces.

**Expected Handling:** The `manual_id()` function calls `ui.input_().strip()` and then `search_id.split()`. Splitting an empty or whitespace-only string produces an empty list `[]`. When `tag_album()` receives an empty `search_ids` list, the `if search_ids:` guard evaluates to `False`, so the function falls through to the standard text-based search. This is the correct and expected behavior — the user effectively cancels their ID entry and gets the normal search results.

### Edge Case 5: Duplicate IDs Provided

**Scenario:** The user accidentally passes the same ID twice (e.g., `-S abc123 -S abc123`).

**Expected Handling:** The `_add_candidate()` function in `match.py` already includes a deduplication check: `if info.album_id and info.identifier in results: return`. So even if the same release is fetched twice, it will only appear once in the candidates dictionary. No duplicate entries should appear in the candidate list presented to the user.

---

## 3.1.5 Initial Prompt

You are tasked with implementing a feature for the **beets** music library manager — a Python command-line tool that imports music files, auto-tags them using MusicBrainz metadata, and organizes them into a library backed by SQLite.

### Feature Overview

Add a `--search-id` (short form `-S`) command-line option to the `beet import` command that allows users to specify one or more MusicBrainz release/recording IDs. When these IDs are provided, the importer should skip the default text-based search and instead fetch metadata only for those specific releases or recordings, dramatically speeding up imports for users who already know the correct MusicBrainz ID.

### What You Need to Implement

**1. CLI Layer (`beets/ui/commands/import_/__init__.py`):**
Add a new option to the `import_cmd` parser:
- Flag: `-S` / `--search-id`
- Destination: `search_ids`
- Action: `append` (so the user can specify it multiple times: `-S ID1 -S ID2`)
- Metavar: `ID`
- Help text: "restrict matching to a specific metadata backend ID"

**2. Pipeline Stage (`beets/importer/stages.py` → `lookup_candidates`):**
Modify the `lookup_candidates` stage to read `search_ids` from the session config and pass it to the task's `lookup_candidates()` method. The IDs should be obtained via `session.config["search_ids"].as_str_seq()`.

**3. Task Model (`beets/importer/tasks.py`):**
Modify `ImportTask.lookup_candidates()` and `SingletonImportTask.lookup_candidates()` to accept a `search_ids: list[str]` parameter and pass it through to `tag_album()` / `tag_item()` respectively.

**4. Autotag Matching (`beets/autotag/match.py`):**
- Change `tag_album()` signature: replace any existing single `search_id` parameter with `search_ids: list[str] = []`.
- When `search_ids` is non-empty, query `metadata_plugins.albums_for_ids(search_ids)` to get candidates. Do NOT fall through to the keyword search.
- When `search_ids` is empty, preserve existing behavior (text-based search).
- Apply the same pattern to `tag_item()`: accept `search_ids: list[str] | None = None`, and when provided, use `metadata_plugins.tracks_for_ids()` to fetch candidates. If the match is not strong, do NOT fall through to keyword search — return whatever was found (or an empty Proposal).

**5. Interactive Prompt (`beets/ui/commands/import_/session.py` → `manual_id`):**
Modify the `manual_id()` function so that the user-entered ID string is split on whitespace (`search_id.split()`) and passed to `tag_album()` / `tag_item()` via the `search_ids` parameter.

### Acceptance Criteria to Satisfy
- AC-1 through AC-8 as defined above — especially backward compatibility (AC-6), multi-ID support (AC-2, AC-7), and metadata backend agnosticism (AC-8).

### Edge Cases to Handle
- Invalid/non-existent IDs → return empty candidates, not a crash
- Mixed valid/invalid IDs → return candidates for valid ones only
- Wrong entity type (recording ID for album import) → no match, user gets fallback prompt
- Empty input at "enter Id" prompt → fall through to standard search
- Duplicate IDs → deduplicate via the existing `_add_candidate` guard

### Testing Requirements
- Add an `ImportMusicBrainzIdTest` class in the test suite
- Mock `musicbrainzngs.get_release_by_id` and `musicbraingngs.get_recording_by_id` — do NOT make real HTTP calls to MusicBrainz in tests
- Test single ID import for both albums and singletons
- Test multiple ID import with candidates sorted by distance
- Test that an invalid ID returns zero candidates without crashing
- Test the interactive `manual_id` prompt with space-separated IDs

### Important Constraints
- Do not break existing behavior when `-S` is not used
- The `search_ids` parameter must flow through the existing pipeline architecture (CLI → session config → stage → task → autotag)
- Use `metadata_plugins.albums_for_ids()` and `metadata_plugins.tracks_for_ids()` as the entry points for ID-based lookups — these are the hook functions that all metadata source plugins implement
- Candidates must always be sorted by distance before computing recommendations
