# Practice work

Scratch material from working through the CI/CD module. Not part of the two
assessed tasks - see [`cicd/`](../cicd) for those.

- [`matrix/`](./matrix) - a small Python package and unit test used while
  experimenting with build matrices and test discovery in GitHub Actions.

## Pipeline lessons from this practice

The experimental workflow failed three times before it ran green. Keeping the
notes because the lessons apply to any CI pipeline, not just this throwaway code.

### `Could not find a version that satisfies the requirement unittest`

`pip install -r requirements.txt` failed with `No matching distribution found
for unittest`.

**Cause:** `unittest` is part of the **Python standard library**, not a PyPI
package, so there is nothing for pip to download and it fails hard.

**Fix:** removed it from the requirements file.

**Lesson:** requirements files list third-party dependencies only. Anything that
ships with the language does not belong there.

### `ModuleNotFoundError: No module named 'matrix'`

The test step failed importing the code under test.

**Cause:** the workflow ran `cd matrix` before invoking the tests, which made
`matrix/` the working directory. The repository root was therefore not on
`sys.path`, so `from matrix.hello import hello` could not resolve. There was
also no `__init__.py`, so the folder was not importable as a package.

**Fix:** added `__init__.py`, and ran discovery from the repository root instead
of changing into the folder:

```bash
python -m unittest discover -s matrix -t .
```

`-s` says where the tests live; `-t` sets the top-level directory placed on
`sys.path`.

**Lesson:** `cd` inside a CI step silently changes Python import resolution. Run
from the repo root and point the runner at the subdirectory.

### Build matrix resolved `3.10` as `3.1`

The matrix requested Python `3.10` but the runner set up `3.1`.

**Cause:** YAML parses an unquoted `3.10` as a **number**, and a trailing zero on
a float is meaningless, so it becomes `3.1`.

**Fix:** quote the versions so they stay strings:

```yaml
python-version: [ "3.9", "3.10", "3.11" ]
```

**Lesson:** classic YAML type coercion. Quote version numbers, and watch for the
same trap with unquoted `yes`, `no`, `on`, and `off`.
