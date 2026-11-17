# Issue Resolution: Prevent AttributeError in RaisesGroup check callable when called with contained exception

## 1. Upstream Issue Reference
- **Repository:** `pytest-dev/pytest`
- **Issue:** [#14324](https://github.com/pytest-dev/pytest/issues/14324) - `pytest.RaisesGroup calls the check function with simple exceptions instead of the group`
- **Status:** Open upstream (confirmed bug by pytest maintainers)

## 2. Problem Description
When using `pytest.RaisesGroup(..., check=check)` where `check` expects an `ExceptionGroup` and inspects group-specific attributes (such as `exc_group.exceptions`):
If `check(exc_group)` returned `False`, `RaisesGroup.matches` attempted a secondary speculative check: it passed the single contained sub-exception (e.g., `ValueError`) to `check()` in order to provide a hint message if the user accidentally passed a sub-exception check to `RaisesGroup`.
However, because `check(actual_exceptions[0])` was invoked without exception handling:
- If `check` accessed group attributes (e.g. `len(e.exceptions)`), it crashed immediately with `AttributeError: 'ValueError' object has no attribute 'exceptions'`.
- The crash superseded pytest's intended test failure reporting (`Failed: check did not return True on the ExceptionGroup`).

## 3. Reproduction Test Case
Added in `testing/python/raises_group.py`:
```python
    # check function inspecting ExceptionGroup attributes should not crash
    # when called on a single contained exception
    def check_eg_attr(eg: ExceptionGroup) -> bool:
        return len(eg.exceptions) > 1

    check_eg_attr_repr = repr_callable(check_eg_attr)
    with (
        fails_raises_group(
            f"check {check_eg_attr_repr} did not return True on the ExceptionGroup"
        ),
        RaisesGroup(ValueError, check=check_eg_attr),
    ):
        raise ExceptionGroup("", (ValueError(),))
```

Prior to the fix:
`AttributeError: 'ValueError' object has no attribute 'exceptions'`

## 4. Solution & Code Changes
In `src/_pytest/raises.py`:
- Wrapped the speculative check on `actual_exceptions[0]` in a `try ... except Exception:` block.
- If `self._check_check(actual_exceptions[0])` raises an `Exception` (e.g., `AttributeError` or `TypeError`), it safely catches it and sets `matched_contained = False`.
- The original failure reason (`check did not return True on the ExceptionGroup`) is properly preserved and reported without crashing.

## 5. Verification
- Reproduction test passed.
- All 30 tests in `testing/python/raises_group.py` and 36 tests in `testing/python/raises.py` passed.
- Ruff linter passed cleanly with zero errors.
