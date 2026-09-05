# Review: non-seekable multipart uploads fail during rendering

## Finding — P1: the rewind catches too narrow an exception

**Defective production file:** `httpx/_multipart.py`, specifically
`FileField.render_data()` at lines 175–179. A readable binary file-like object
may expose `seek()` but raise `OSError` when seeking is unsupported. The code
attempts `seek(0)` and catches only `io.UnsupportedOperation`, a subclass of
`OSError`. Other `OSError` instances escape, aborting the multipart upload before
the file's first `read()`, even though length detection accepted the stream.

### Evidence in this tree

- `CHANGELOG.md:7–15` records the 0.23.2 change (#2382) to stream unknown-length
  multipart files with chunked transfer encoding instead of reading them into
  memory. This is the relevant recently introduced behavior.
- `httpx/_utils.py:371–393`, `peek_filelike_length()`, catches `OSError` from
  file-descriptor and seek/tell operations and returns `None` for an unknown
  length. This helper establishes the supported failure case; it is not itself
  the defective exception handler.
- `httpx/_multipart.py:138–151` propagates that unknown length;
  lines 225–259 select `Transfer-Encoding: chunked` when any field's length is
  unknown. Thus header construction accepts the stream, but rendering rejects
  it at lines 175–179.
- `tests/test_multipart.py:380–415` has a coverage gap: its non-seekable
  `IteratorIO` inherits `io.IOBase`'s seek behavior, which raises
  `io.UnsupportedOperation`. It exercises only the exception already caught,
  not a plain `OSError`. Lines 418–434 cover successful rewinding, not this
  failure. This is the relevant test file, not a second production defect.

### Reproduction and observed result

A minimal affected input is a readable binary stream with a failing seek:

```python
class Upload(io.BytesIO):
    def seek(self, *args):
        raise OSError("stream is not seekable")

upload = Upload(b"HelloWorld")
```

Executed the checked-in `peek_filelike_length`, `to_bytes`, and `FileField`
definitions directly using Python's AST compiler, without modifying them or
writing files. Constructed a `FileField` via `object.__new__`, assigned `file`
to the input above and `_headers` to fixed bytes, then called `get_length()`
and joined `render_data()`. Repeated with `io.UnsupportedOperation` instead of
`OSError` as a control. Output:

```text
OSError length: None
render failed: OSError: stream is not seekable
unconsumed payload: b'HelloWorld'
UnsupportedOperation length: None
rendered: b'HelloWorld'
```

Expected: both non-seekable streams remain readable and their payloads stream
with unknown length. Actual: the plain `OSError` case fails before consuming
the payload. Both synchronous and asynchronous multipart iteration use this
same rendering path (`httpx/_multipart.py:261–267`).

### Verification boundaries

Importing the full local package failed with `ModuleNotFoundError: No module
named 'sniffio'`. The execution above verifies the source-level failure, not an
end-to-end HTTP request or a passing pytest suite. No dependencies were installed
and no production or test files were changed.

After unshallowing, this work branch still has only the synthetic snapshot in
its ancestry. The changelog supports the connection to #2382, but the original
merge and later corrective commit are not available in that ancestry; their
exact historical attribution has not been independently verified. The defect
reported here is directly observable in the checked-out tree. No fix is applied.
