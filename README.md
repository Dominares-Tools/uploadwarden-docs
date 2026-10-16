# uploadwarden — Accept file uploads without the usual holes

**Paid tool: $49 once (plus sales tax where it applies), then a private GitHub repository you're invited to, with every update. Buy: https://dominares.org/tools/uploadwarden**

Commercial license v1.3 — see https://dominares.org/license · Support: support@dominares.org

An extension allowlist, a magic-number check, an SVG refusal list, generated stored names, size caps, safe download headers, a path-resolve guard and an index that survives redeploys, as a small standard-library Python library and command.

## Get it

Buy uploadwarden at https://dominares.org/tools/uploadwarden: one payment of $49, plus sales tax where it applies, and every future update. After checkout you are invited to a private GitHub repository with the source and every version published to it. The repository includes the Python library and the uploadwarden command (check, add, list, delete, headers), sample files, a WSGI example and the full test suite.

Requirements: Python 3.10 or newer, no third-party packages. There is no virus scan, sign-in or rate limiting.

This page documents version 1.0.2. Install commands and file paths in the documentation below run inside the private repository you are invited to; the source is not in this repository.

## Documentation

Checking the five files in `samples/` gives a verdict for each: a real PDF and a plain SVG logo are accepted, a page of HTML named `forged.pdf`, an SVG with an `onload` handler and an `.exe` are refused. The command exits 1 because some were refused, which is what you would wire into CI. Run from the root of your copy of the repository, it prints:

```
REFUSE  samples/forged.pdf  That file is named .pdf but its contents are not a real PDF. Re-export it and try again.
ACCEPT  samples/guide.pdf  ->  guide-5a838678.pdf  (45 B)
ACCEPT  samples/logo.svg  ->  logo-e23386f9.svg  (113 B)
REFUSE  samples/setup.exe  “setup.exe” is not a supported type. You can upload: jpg, pdf, png, svg, webp.
REFUSE  samples/trap.svg  That SVG has parts that could run code or load other files, so it cannot be used. Export it again as a plain SVG, or as a PNG.
```

### Why

Every value in an upload (the name, the declared type, the bytes, the size) comes from a browser and ends up as a filesystem path and a public URL. The usual bugs are old and boring: `../../` in a name, an HTML file uploaded as `guide.pdf` and served from your own origin (stored XSS), an SVG with a script in it, a 2 GB body, an index file that loses a row when two requests arrive together, and uploads that vanish on the next deploy because they were written next to the code.

uploadwarden is the checklist for those, as functions you call from whatever web framework you use. It imports no framework and has no runtime dependencies (Python 3.10 or newer, standard library only).

### What it does

| Defence | Where | What it does |
|---|---|---|
| Size cap | `validate_upload` | Default 25 MB, any limit you pass. Checked first, so a huge file is refused before anything else looks at it. Empty files are refused. |
| Extension allowlist | `validate_upload`, `normalize_ext` | `pdf`, `png`, `jpg` (also `.jpeg`), `webp` by default. Anything else is refused with a message that lists what is allowed. You can pass your own allowlist. |
| Type proven by the bytes | `validate_upload` | The file must start with the magic number of its extension (`%PDF-`, the PNG signature, `FF D8 FF`, and for WebP both `RIFF` and `WEBP` at byte 8). A page of HTML named `.pdf` is refused. |
| SVG check | `validate_svg` | SVG is refused unless you pass `allow_svg=True`. Even then it is refused if it carries a script, event handler, `javascript:` link, embedded HTML, entity declaration, external reference (`url()`, `image-set()`), stylesheet import, a CSS escape (a backslash inside a stylesheet or `style=` attribute), or a link that is not `#id` or an inline raster image. Character references are decoded first, so `&#106;avascript:` is caught, and an XML namespace prefix does not hide an element (`<s:script>` is refused like `<script>`). Ordinary prefixed elements from drawing tools (`sodipodi:`, `inkscape:`, `svg:`) are accepted. |
| Safe stored name | `safe_stored_name`, `slugify` | `<ascii-slug>-<8 hex of the content hash>.<ext>`. Traversal, backslashes, NUL bytes, 400-character names, look-alike letters, double extensions (`evil.php.pdf`) and Windows device names (`con`) all collapse into an inert name. The original name is kept only as a label. Uploading the same bytes twice gives the same name. |
| No silent overwrite | `save_upload` | If a different file already sits at the name an upload would get (a crafted hash collision), the new file gets a longer hash. The old file is never replaced. |
| Path-resolve guard | `resolve_upload_path` | For the serving route. Rejects empty names, separators, colons (Windows drive and stream names), NUL bytes, names starting or ending with a dot, reserved Windows device names (`CON`, `NUL`, `COM1`...), and anything that resolves outside the uploads folder, including through a symlink. |
| Serving headers | `download_headers` | `Content-Type` from the stored extension (unknown means `application/octet-stream`), `attachment` by default (`inline=True` for images), `X-Content-Type-Options: nosniff`, a long immutable cache, and for SVG a sandboxing `Content-Security-Policy` (`default-src 'none'; style-src 'unsafe-inline'; sandbox`). The filename in the header is restricted to `[A-Za-z0-9._-]`. |
| An index that survives | `save_upload`, `delete_upload`, `load_uploads` | A JSON index (`uploads_index.json`) sits beside the bytes in the root folder you give, so whatever persists one persists the other. Bytes and index are written atomically (temp file, then rename), and the index update runs under a cross-process lock. 16 processes saving at once lose no row (tested). A missing or damaged index reads as empty and heals on the next save. |
| Download link | `render_attachment`, `human_size` | An escaped `<a download>` with type and size. The link target is used only if it is site-relative or `http(s)`; `javascript:`, `//host`, `/\host` and addresses with a control character (tab, newline) are replaced by the generated one. |

Everything takes a `root` folder instead of reaching for global settings:

```python
from uploadwarden import save_upload, resolve_upload_path, download_headers, UploadRejected

try:
    rec = save_upload("/var/data/site", body_bytes, client_filename, actor="ana")
except UploadRejected as e:
    ...  # e's text is written to be shown to the person who uploaded the file
else:
    rec["url"]          # "/uploads/annual-guide-5a838678.pdf"  (url_prefix= to change it)

# in the GET route, with the name taken straight from the URL:
path = resolve_upload_path("/var/data/site", name_from_url)   # raises UploadRejected if hostile
headers = download_headers(name_from_url)                    # send these with path.read_bytes()
```

`examples/wsgi_app.py` is a complete upload-and-download endpoint on the standard library (about 40 lines). Run `python3 examples/wsgi_app.py`, then `curl --data-binary @samples/guide.pdf 'http://127.0.0.1:8000/upload?name=guide.pdf'`. The tests exercise it, including traversal attempts.

### Options

Library functions take these keyword arguments where they apply: `allow_svg` (default `False`), `max_bytes` (default 25 MB), `types` (your own allowlist: `{"ext": {"mime": ..., "magic": (b"...",), "label": ...}}`), `url_prefix` (default `/uploads`), `actor` (who uploaded it, recorded in the index, cut to 80 characters).

On disk, under the root you pass: `uploads/<stored name>` and `uploads_index.json` (plus a `uploads_index.json.lock` file). Put the root on a volume that survives deploys.

#### Command line

```
uploadwarden [--root DIR] [--json] [--max-mb N] [--allow-svg] <command>

  check FILE...        would these be accepted? prints ACCEPT or REFUSE; exit 1 if any refused
  add FILE             store a file (--actor NAME, --url-prefix /media)
  list                 what is stored
  delete STORED_NAME   remove a stored file and its index row
  headers STORED_NAME  the response headers it should be served with (--inline)
```

`--root` defaults to `uploads-store`. Exit codes: 0 success, 1 refused or nothing to delete, 2 file unreadable.

### Limits

Read these before you rely on it.

- **It checks the first bytes, not the whole file.** A file that starts with `%PDF-` is accepted even if the rest is not a valid PDF, and a PDF can contain active content. There is no virus scan and no re-encoding of images. If you need either, run them after `validate_upload`.
- **The SVG check is a refusal list, not a sanitiser.** A file that trips it is rejected, never cleaned up. It is one of three layers: the sandboxing header from `download_headers`, and the fact that an `<img>` tag never runs an SVG's script. If you serve SVG, send those headers.
- **It does not authenticate anyone, check CSRF tokens or rate limit.** It validates and stores a file; who may upload is your application's decision.
- **It works on bytes already in memory.** Refuse oversized requests in your framework or proxy before reading the body (the WSGI example does so from `Content-Length`). The size cap here is a second line, not the first.
- **The cache header says `public`.** Names are content-addressed, so a URL never changes, which is what makes `immutable` correct. It also means the bytes are cacheable by shared caches. Do not use these headers for files that must stay private; serve those through your own authorised route.
- **The index is one JSON file rewritten on each save.** Every save reads and rewrites the whole file, so it suits a site's document library, not a store with millions of files.
- **The lock is advisory.** It serializes processes that call uploadwarden on the same root. On a filesystem with no lock support it does nothing rather than failing a save. The tests cover POSIX; the Windows code path is not exercised, and the Windows name rules in `resolve_upload_path` are checked as text patterns only.
- **Names are ASCII.** A name written entirely in non-Latin letters is stored as `upload-<hash>.<ext>`; the original is kept in the index as `original_name`.

---

Docs mirror of the private repository Dominares-Tools/uploadwarden; updated with each release.
