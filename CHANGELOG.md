# Changelog

## 1.0.2

README only: it opens with the sample verdicts and leaves the install steps to the page you bought it from. No code change.

## 1.0.1

Security fixes found in review. The README sample output now lists svg in the supported types. Upgrade if you accept SVG or use `render_attachment`.

- `validate_svg` now sees through an XML namespace prefix: `<s:script>`, `<h:iframe>`,
  `<s:foreignObject>` and a prefixed `<set attributeName="href">` are refused. Ordinary prefixed
  elements from drawing tools (`sodipodi:`, `inkscape:`, `svg:`) are still accepted.
- `validate_svg` refuses a backslash inside a stylesheet or a `style=` attribute (CSS escapes
  such as `\75 rl(` hid an external reference) and refuses `image-set()`.
- `render_attachment` replaces a link target of the form `/\host` or one containing a control
  character such as a tab or newline (browsers read both as `//host`).
- `resolve_upload_path` also rejects a colon, a trailing dot and reserved Windows device names.
- The size refusal for a limit under 1 MB reads in bytes or KB (`11 B ... 10 B`), and the
  "not a supported type" message lists `svg` when `allow_svg` is on.
- The public-API test no longer skips on Python 3.10 except the two tests that read `pyproject.toml`.

## 1.0.0

First release.

- `validate_upload`: size cap, extension allowlist, and the file's magic number checked against
  its extension. WebP also requires the `WEBP` marker at byte 8.
- `validate_svg`: an SVG is accepted only with `allow_svg=True`, and only if it is plainly a
  drawing (no script, event handlers, script URLs, embedded HTML, entities, external references).
- `safe_stored_name`, `slugify`: a generated, content-addressed, ASCII-only stored name.
- `resolve_upload_path`: a path-resolve guard that rejects separators, dot-names, NUL bytes and
  symlinks that leave the uploads folder.
- `download_headers`: attachment by default, `nosniff`, explicit type, sandboxing CSP for SVG.
- `save_upload`, `delete_upload`, `load_uploads`: atomic writes, an index that lives beside the
  bytes, a cross-process lock around the index update, and no overwrite when two different
  files would share a stored name.
- `render_attachment`, `human_size`: an escaped download link for a stored file.
- A `uploadwarden` command: `check`, `add`, `list`, `delete`, `headers`.
