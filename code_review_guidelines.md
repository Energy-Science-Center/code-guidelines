# Code review guidelines

General rules for writing, reviewing, and preparing code for review. Project-specific additions belong
in a separate conventions file.

How to read this file: every bullet is a required rule, except bullets marked *(own)*, which are
recommendations. Apply all rules BEFORE asking for review, not after.

## Top priorities

The rules that matter most, in order. Check these first, in every change and every review; the
sections below give the details.

1. **Agree on the design before coding** (section 0). Once the MR is open, the structure is decided.
2. **No silent data loss** (section 2). A run that drops data or catches an error and continues must
   fail, never report success.
3. **One function, one goal** (section 3.1). A function that does several things is split; a function
   whose name needs "and" does too much.
4. **Descriptive names** (section 5). Every function and variable name says what it holds or does;
   no generic names (`data`, `df`, `val`), no unclear abbreviations (`h3`, `ts_typ`).
5. **Tests for every new or changed behaviour** (section 9). An MR without them is not ready.
6. **Each value defined in one place** (section 4). Constants, defaults and shared names live in one
   file or the configuration; no copies across modules.
7. **Extend, don't duplicate** (section 3.2). No parallel process next to an existing one; no copied
   logic.
8. **No dead code** (section 3.3). Unreachable branches, unused functions, debug-only outputs and
   commented-out code are deleted.
9. **Docstrings and comments match the code and stay short** (section 10.2).
10. **Type hints on every new or changed function** (section 6).

## 0. Before coding: agree on the design

Once a merge request is open, the structure is effectively decided: changing it then means redoing the
work. So the structure is agreed before the first line of code, not discussed in the review.

1. **Start from an issue** that describes the problem and the planned solution.
2. **Write a short plan:** what changes, which existing modules are touched or extended, where new code
   goes, what the data flow and outputs are, and how it will be tested.
3. **Check the plan against the structure rules** (sections 3 and 4): no parallel process next to an
   existing one, no duplicated constants or logic, single responsibility, settings from configuration.
4. **Discuss the plan with the reviewer** and agree on the general structure. Only then start coding.
5. If the design has to change during implementation, go back to the reviewer before continuing.

## 1. Branches, commits and merge requests

- **Aim for one feature per branch and per merge request**; an independent feature gets its own MR.
  This is a goal, not an absolute rule: when features cannot be separated (e.g. one depends on the
  other), keep them in one MR, say so in the MR description, and keep each feature in its own commits.
- Plan the scope so that a branch is one feature from the start where possible. A branch that has
  grown into several separable features can be split into several branches (one per feature, each
  rebased onto the main branch), but this is a repair, not the goal.
- Create the branch from the latest main branch and name it after the issue.
- An MR contains only its own commits; rebase onto the main branch before asking for review.
- Keep MRs small enough to review; the description links the issue and states what changed, why, and
  how it was tested.
- Assign a reviewer, and merge only after approval.
- Untracked or unrelated files (scratch scripts, local project files, other modules) stay out of the MR.
- Commit messages describe the change in one clear line.
- Reply to every review thread and resolve it once addressed.

## 2. Correctness: no lost or corrupted data

- The code does what the MR claims; check edge cases: empty input, missing files, missing values,
  units, time zones, leap years, index/shape alignment.
- No silent failures: no bare `except`, no catch-log-continue that drops data, no defaults that hide
  missing input. A run that loses data must fail, not report success.
- Results are reproducible: same input, same output, no hidden state.
- Guard data integrity before writing (e.g. raise on duplicate keys before a database insert).
- Functions don't mutate their inputs unless that is their documented purpose; work on a copy, because
  a DataFrame or list is passed by reference and the caller's data would change without notice.
- Functions don't change global state (library configuration, environment, working directory); scope
  such changes with a context manager (`with ...:`), because a global change silently affects all code
  that runs afterwards.

## 3. Code structure

### 3.1 One responsibility per function and module

- Single responsibility: one component owns one concern. Shared I/O goes through one component instead
  of being reimplemented in each module.
- **One function, one goal.** A function does one thing, at one level of detail. Split it when its
  name needs "and" (`load_and_validate`), when its docstring lists several steps, or when it mixes
  computing, file I/O and reporting (e.g. a 200-line function that builds paths, runs a pool,
  aggregates results and writes reports).
- Functions are short; avoid deep nesting.
- Separate computation from I/O and logging so the computation can be tested on its own.
- Keep the public interface small; helpers used only inside a module are prefixed with `_`.

### 3.2 No duplication

- Extend an existing process instead of adding a parallel one next to it.
- Extract duplicated code into a function or a class. Shared state goes into arguments, a dataclass or
  a class, not module-level copies. Parallel dicts keyed by the same id want one dataclass (section
  3.4).

### 3.3 No dead code

- No unused imports or variables, no commented-out code, no unused files or `__main__` blocks.
- Remove debug-only features, flags and outputs; keep only what users need.
- Leftover structure from removed features (wrappers, result tuples, mode flags) is deleted with them.
- No unreachable code. A branch, `isinstance` fallback or `else: raise` that no caller can reach is
  dead code and must be deleted. Validate inputs once at the boundary (section 7); do not re-validate
  inside the code what the boundary or the type already guarantees. Prefer an enum or a type
  conversion over hand-written checks for values the type system can rule out.

### 3.4 Grouping data: dataclasses

A dataclass gives a group of values that belong together one name, one type and named fields.

**Use a dataclass when:**
- several values travel together through functions (passed in or returned together);
- a function returns more than two values: return a dataclass, not a tuple
  (`LocationResult(location, error, unpaired, ...)`, not a 5-tuple read by position);
- a dict has a fixed set of known keys (`{'input': ..., 'filtered': ..., 'output': ...}`): its keys
  are field names, so write `folders.output`, not `folders['output']`;
- two or more dicts share the same keys (`bus_to_grids[bus]`, `bus_metadata[bus]`): one dict of
  dataclasses (`substations[bus].grid_indices`, `substations[bus].metadata`);
- a component needs settings: one config dataclass with typed fields.

**Do not use a dataclass when:**
- the keys are data, not names (bus name to value): that stays a `dict[str, float]`;
- the object mainly has behaviour or changing internal state: write a normal class.

**How to write one:**
- Every field has a precise type (section 6). A field type that is too long to read is a sign for a
  further dataclass.
- Use `@dataclass(frozen=True)` for values that do not change after creation (results, settings,
  records). Create a changed copy with `dataclasses.replace(obj, field=new_value)`.
- Mutable defaults use `field(default_factory=list)`, never `= []`.
- A field the caller always passes has no default; defaults live in one place only (section 4).
- Derived values are a `@property` or a method on the dataclass (`config.output_dir(year)`), not an
  extra field that can disagree with the fields it is computed from.
- `__post_init__` checks only rules that involve several fields or value ranges (`end > start`,
  `workers >= 1`); it does no file I/O.
- Pass the dataclass itself. Never copy its fields one by one into another object
  (`self.year = config.year`, `self.path = config.path`, ...): that creates a second copy that can
  drift.
- Name it as a noun for what it is (`LocationResult`, `ScenarioFolders`, `Substation`), not for what
  a function does with it.

Example:

```python
# Before: parallel dicts and a tuple read by position
bus_to_grids, bus_metadata = build_mapping(ds, shapes)
ok, message, unpaired, flat, substituted = process_location(...)

# After: one dataclass per concept
@dataclass(frozen=True)
class Substation:
    bus_name: str
    grid_indices: list[tuple[int, int]]   # (lat_idx, lon_idx)
    metadata: dict[str, str | float]

substations: list[Substation] = build_mapping(ds, shapes)
result: LocationResult = process_location(...)
if result.error: ...
```

## 4. Settings, constants and paths

- Define each constant once in an importable place: defaults, mappings, and file or folder names that
  one component writes and another reads.
- **No literal numbers in the logic.** Every number with a meaning (a threshold, a year, a buffer
  size, a limit) is a named variable. Its value lives in exactly one place: the configuration file
  (TOML) if a user may change it, otherwise the one central file that holds all defaults and
  constants. Code reads it from there; no module defines its own copy. Exempt: `0`, `1`, and numbers
  whose meaning is the arithmetic itself (e.g. `/ 2` for a mean of two, `* 100` for a percentage).
- Dependencies are passed in: every setting comes from a TOML configuration file, never from editing
  code. No `input()` prompts inside logic. No hardcoded paths.
- Users choose input and output folders freely through parameters.
- Overrides update only the given keys; they never replace the whole block.
- An option with exactly two values is a boolean named for its observable effect
  (`write_columns_without_data`), not an enum or string naming an internal mechanism.

## 5. Naming

- **Descriptive names.** Every function and variable name says what it holds or does
  (`monthly_generation_file`, not `existing_path`) and makes the data source explicit when there are
  several. A reader must understand a name without reading the code that fills it.
- No unclear abbreviations (`h3`, `t_col_df`, `val_typ`); write the words out (`historic_blocks`,
  `typical_values`). Common short forms (`id`, `max`, `n_`) are fine.
- Names say *what*, not *how*: `reconstruct_hourly_profiles`, not `vectorized_match_and_reconstruct`.
- Never use a generic word alone as a name: `data`, `df`, `mappings`, `source`, `body`, `payload`,
  `first`, `table`, `ts`, `result`, `val`. Add what it holds: `node_table`, `hourly_profiles`.
- A name that no longer matches what the code does is a bug and must be renamed in the same change
  (e.g. `save_csv` that now writes parquet). "Backward compatibility" is no reason to keep a wrong
  name on an internal function; only a published library API needs that.
- Function prefixes match behaviour: `_write_*` only writes, `_get_*` returns a value.
- One name per concept: the same object is not called three different things across functions.
- Use the right term for things: a copy of originals is a `backup`, not an `archive` or `tmp`;
  `payload` is the data as opposed to its metadata, never the metadata.
- Avoid words that have a domain meaning when you mean something else (in a power-system codebase,
  "grid" means the electrical grid; do not use it for a data grid or table).

## 6. Type hints and code style

- Follow the language's standard style guide (PEP 8 for Python) where project rules are silent.
- Full and precise type annotations on every function the MR adds or changes, including return types
  and generic contents: `-> dict[str, list[str]]`, never bare `tuple`/`dict`/`list`/`set`. Missing
  hints on untouched functions are not a blocking review point. A type too long to read is a sign the
  data wants its own structure.
- Write type hints unquoted (`list[str]`, `str | None`), as Python 3.10+ allows; quoted hints break
  `typing.get_type_hints`. Quote only a forward reference that cannot be resolved otherwise.

## 7. Input, output and logging

- Code runs from the repository root, through the project's single entry point (the framework's
  app), never as a standalone script with its own setup. A standalone helper tool that is not part of
  a run uses `python -m package.module` from the root. Modules contain no `sys.path` changes or
  `ImportError` fallbacks.
- New functionality is added as a module (plug-in) that follows the framework's plug-in interface and
  is registered where the framework creates its modules; it does not bypass the framework.
- All outputs go to one defined output folder. Generated folders go into `.gitignore`; consider whether
  they can grow unbounded.
- Validate inputs at the boundary and fail early with a clear message.
- Use `logging`, never `print()`. ASCII-only log messages. Log an error once, where it is handled, not
  at every level it passes through. Usage instructions belong in the README, not in log output.

## 8. Performance

- Don't do work that may be unused inside a loop (read a file only when it is needed).
- Use simple, idiomatic, vectorised library calls instead of row-by-row loops over large data.
- Use the library's direct API. Do not build intermediate objects (e.g. extra DataFrames) that the
  target function does not need.
- Avoid repeated full passes over the same data on disk when one pass can do the work.
- Optimise only where it matters; readability first.

## 9. Tests

- Write automated tests first (TDD), right before the code they cover. Manual test scripts do not
  belong in the codebase.
- An MR without unit tests for its new or changed behaviour is not ready for review. Tests document
  what the code is supposed to do; reviewers read them to understand the intent.
- Tests are required even when no CI pipeline runs them.
- Check that every test can fail: break the code it guards on purpose (e.g. change a condition or a
  value) and confirm the test fails, then restore the code. A test that still passes guards nothing.
- Framework: pytest. Test layout mirrors the production code (`tests/<package path>/test_<module>.py`).
- One `class Test<Component>` per component under test; one behaviour per test.
- Every test has three marked parts: `# Arrange`, `# Act`, `# Assert`. When the call itself raises,
  Act and Assert merge into one `# Act & Assert` around `pytest.raises`.
- Several cases of the same test: `@pytest.mark.parametrize`.
- Expected results are hard-coded or read from a fixed source; never computed with the production logic.
- **Error tests assert the type or the whole message.** When the message is the behaviour under
  test, match all of it (`match=re.escape(expected_message)`); otherwise assert only the exception
  type. A partial substring hides which part matters, breaks on rewording, and can be matched by an
  unrelated error.
- **A test reads on its own.** A reader understands it without opening the code under test or the
  fixtures; a domain fact or a deliberate quirk of the setup (a fixture missing a column on purpose)
  is stated in a comment.
- **Failure-test inputs are obviously synthetic** (`any_unknown_name`, not a plausible real name),
  so the test keeps its meaning whatever the production data later contains.
- **Call the function under test directly**, not through a test helper that wraps it and hides what
  each test exercises.
- F.I.R.S.T.: Fast, Independent (any order), Repeatable (own data, no external state: `tmp_path`,
  in-memory databases, `monkeypatch`), Self-validating (pass/fail only), Timely (written right before
  the code).
- Test public behaviour, including edge cases; test a private helper directly only when it holds
  decision logic with many cases.

## 10. Documentation

### 10.1 README and other documents

- The README matches the code: inputs, outputs, configuration parameters, how to run.
- Docs referenced in code must exist.

### 10.2 Docstrings and comments

Based on PEP 257 (the docstring counterpart of PEP 8), the Google Python Style Guide, and Clean Code.
Points marked *(own)* are stricter than these standards: suggestions, not hard rules.

- **First line: a one-sentence summary** of what the function does or returns, in command form,
  ending with a period ("Return the mean annual yield (Wh/Wp) of the installed orientations.").
  For longer docstrings: summary line, blank line, details. *(PEP 257)*
- **At most 5 lines of text**, plus a parameter block only where the inputs need defining (a
  parameter whose name and type already say what it is gets no entry). A reader must understand the
  function from the first lines, not after 15 lines of docstring.
- **Enough to call the function without reading its code.** Describe what it does, not how it is
  implemented; implementation details go in comments inside the code. *(Google)*
- **The summary adds information beyond the name.** Do not restate the function name ("Process the
  data." for `process_data`); state the non-obvious half: what is returned, written, or assumed.
- **Too short is also a problem:** "Process a single scenario." should at least say what is
  written where or what is returned.
- **Define terms** the first time they appear, or once centrally (e.g. a type alias with a one-line
  explanation) when they are used across modules. Metaphors ("recipe") need a definition. Terms
  defined only in another repository or document must be defined in this repository or avoided.
- **Describe current behaviour only.** No history ("the previous code ...", "was changed because
  ..."); version control holds history. *(Clean Code)*
- **Explain *why* only when it is not obvious**, and keep it short.
- **Docstrings and comments must match the code.** A comment that explains a decision the code no
  longer makes is worse than no comment; update or delete it in the same change. Always a valid
  review comment. *(Clean Code)*
- **Inline comments: 1-2 lines, 3 at most.** Start with the rule or reason, not with a case study or
  an example from the data. Stories about how a bug was found belong in the commit message.
- **Longer derivations, measurements and design reasoning** (e.g. measured values comparing two
  methods) belong in the README or a methods section, not in a docstring read every time the
  function is opened. *(own)*
- Pick one section format for the project (Google `Args:/Returns:/Raises:` or NumPy style) and use
  it consistently; this is a team decision.

## 11. How to review a merge request

- **Review the latest version.** Fetch first and read the files from the remote branch, not from a
  possibly outdated local checkout. State which commit was reviewed, so line numbers match the MR diff.
- **Check before claiming.** Verify every point in the code before raising it: search whether a
  setting or function is used elsewhere, who reads a shared name, whether a value is passed in anyway.
  Let the result set the severity.
- **Rate severity honestly.** Mark each point as *worth flagging* (correctness, silently lost data,
  shared constants that can drift), *medium* (design, duplication, type hints), or *nit* (style,
  wording). Say plainly when a point has no practical effect today.
- **Write clear review comments.** One comment per point, on the line it concerns: what the problem
  is, what can go wrong, and a concrete fix. Phrase it as a suggestion or a question ("Could we
  ...?"). Group nits into one cleanup comment.
- **Track status after new commits.** Re-check every earlier point against the new version and mark it
  *done*, *partly done* or *open*; review only the changed files again.

### 11.1 Additional rules for AI-assisted reviews

When an AI reviews a merge request for a person, who then posts the comments:

- **Explain each point to the person first**, in simple words: what the problem is and what can go
  wrong; then the fix, with a short code example where it helps.
- **Give a ready-to-paste review comment for every point**, with file and line.
- **Mark points that earlier general comments already cover**, so they are not posted twice.
- **Output format:** per file, a list of points grouped by severity, each with file:line, the problem,
  the fix, and the suggested comment. After new commits: a status table (point | status | note).

## 12. Addressing review comments with an AI assistant

- Handle one review comment at a time. Ask the user about one comment per question, so that a short
  answer ("yes") can only mean one thing.
- Show each fix (the diff and the commit message) to the user before committing it.
