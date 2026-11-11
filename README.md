# mkdocs-fix

**Unofficial patch for CVE-2026-117425: directory traversal in the MkDocs `mkdocs serve` dev server on Windows.**

This repo contains a single file: MkDocs **1.6.1**'s `mkdocs/livereload/__init__.py` with the path check in `_serve_request()` fixed. Nothing else is changed, so it works as a drop-in replacement until an official MkDocs release fixes the issue.

| | |
|---|---|
| **CVE** | CVE-2026-117425 |
| **GitHub Advisory** | GHSA-88gf-v88m-p95v |
| **Affected** | MkDocs 1.2.3 – 1.6.1, `mkdocs serve` only, Windows hosts only |
| **Not affected** | Linux / macOS, sites built with `mkdocs build`, MkDocs 2.0 pre-releases |
| **Official fix** | None yet |
| **This patch** | For MkDocs **1.6.1 only** |
| **Write-up** | [w4nn4d13.in/posts/cve-2026-117425](https://w4nn4d13.in/posts/cve-2026-117425/) |

---

## The problem

`mkdocs serve` maps the URL path to a file inside its build folder. To stop `../` traversal it cleans the path with `posixpath.normpath()`:

```python
# MkDocs 1.6.1, mkdocs/livereload/__init__.py, lines 291-292
rel_file_path = posixpath.normpath("/" + rel_file_path).lstrip("/")
file_path = os.path.join(self.root, rel_file_path)
```

`posixpath` only treats `/` as a separator. On Windows, `os.path.join()` also treats `\` and drive letters (`C:`) as path syntax, so some request paths make it drop the site folder entirely. The server then opens and returns files from anywhere on the disk that the process can read.

By default `mkdocs serve` listens on `127.0.0.1`, so an attacker needs to be on the same machine. With `--dev-addr 0.0.0.0`, anyone on the network can reach it.

## The fix

The patch builds the path with the host OS's own rules, resolves it, and refuses anything that ends up outside the site folder:

```diff
-            # Prevent directory traversal - normalize the path.
-            rel_file_path = posixpath.normpath("/" + rel_file_path).lstrip("/")
-            file_path = os.path.join(self.root, rel_file_path)
+            # Prevent directory traversal - use os.path.normpath (not posixpath)
+            # so that backslashes are treated as path separators on Windows.
+            file_path = os.path.realpath(os.path.join(self.root, rel_file_path))
+            root_real = os.path.realpath(self.root) + os.sep
+            if not file_path.startswith(root_real):
+                return None  # Path traversal attempt rejected
```

Out-of-root requests now get a **404**. Normal pages, assets and in-folder links still load.

> **Note:** a hardened version of this check (`normpath` → `commonpath` + `normcase` → `realpath` → check again) is described in the write-up. It also handles drive-root folders and letter-case differences on Windows, and rejects bad paths before touching the filesystem.

## How to apply (Windows, MkDocs 1.6.1)

```powershell
# 0. Make sure you are on 1.6.1
py -3 -m mkdocs --version

# 1. Find the installed livereload folder
py -3 -c "import mkdocs.livereload, os; print(os.path.dirname(mkdocs.livereload.__file__))"

# 2. Back up the original
copy <that-folder>\__init__.py <that-folder>\__init__.py.bak

# 3. Replace it with the patched file
curl.exe -L -o <that-folder>\__init__.py https://raw.githubusercontent.com/0x5t4l1n/mkdocs-fix/main/mkdocs/livereload/__init__.py
```

**To undo it:** copy `__init__.py.bak` back over `__init__.py`, or run `pip install --force-reinstall mkdocs==1.6.1`.

## How to check it worked

Start `mkdocs serve` on a test project. Then request a file that's outside the site folder, using the method from the write-up. On the patched server you should get **404**, and your normal docs pages should still load.

## Tested

| Test | Result |
|---|---|
| Normal pages, assets, in-folder symlinks | ✅ still served |
| Backslash, `..\`, drive-letter and other-drive paths (Windows path rules, simulated) | ✅ blocked (404) |
| Symlink pointing outside the site folder | ✅ blocked (404) |
| Live run on a Windows host | ⏳ not yet re-tested with this patch |

## ⚠️ Disclaimer

- This is an **unofficial, community patch**. It is **not** made, reviewed, or endorsed by the MkDocs maintainers.
- It is provided **as-is, without warranty of any kind**. Use it at your own risk, on your own development machines.
- It applies to **MkDocs 1.6.1 only**. Don't copy it over any other version.
- Read the diff before you use it.
- Upgrading or reinstalling MkDocs overwrites the patched file. Once an **official fixed release** is out, run `pip install --upgrade mkdocs` and stop using this patch.
- `mkdocs serve` is a development server. Never use it to host a production site; deploy the output of `mkdocs build` instead.
- Shared for **defensive and educational purposes**. Only test systems you own or are authorized to test.

## Credits & license

- Vulnerability found and reported by **[@0x5t4l1n](https://github.com/0x5t4l1n)** (w4nn4d13).
- MkDocs © 2014-present Tom Christie, BSD 2-Clause license. This patched file is a derivative of MkDocs and is distributed under the same license. See [`LICENSE`](LICENSE).

## Links

- CVE record: https://www.cve.org/CVERecord?id=CVE-2026-117425
- MkDocs advisory: https://github.com/mkdocs/mkdocs/security/advisories/GHSA-88gf-v88m-p95v
- Original code (1.6.1): https://github.com/mkdocs/mkdocs/blob/1.6.1/mkdocs/livereload/__init__.py#L291-L292
- MkDocs: https://github.com/mkdocs/mkdocs
