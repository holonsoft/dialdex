# Import formats

Reverse-engineered from real backup files (not committed, they contain private URLs).
All example values below are invented. Analysed: 2026-11-17.

The importers live in the core layer and map each format to a neutral `ImportBundle` (see PLAN.md 4.5).
This document lists only the facts that shape the mapper code.

---

## 1. Group Speed Dial (Firefox / Chrome)

- File: `.txt`, one line of UTF-8 JSON. Real sample: ~14 MB, 46 groups, 746 dials, 702 thumbnails.
- Format version: `dataVersion: 33`. **An unknown `dataVersion` makes the import fail with a clear message.**
  No best-effort guessing.

### Shape

```jsonc
{
  "dataVersion": 33,
  "groups": [
    {
      "id": 5, "name": "News", "rows": 5, "cols": 6,
      "background": { "type": 0, "value": "rgb(0, 0, 66)", "onlyName": false }, // optional
      "color": "white",                                                        // optional
      "iconId": 0, "archived": false, "options": {},
      "dials": [
        { "name": "Example News", "created": 1700000000, "type": 0, "url": "https://news.example.com/" },
        {},                                                                     // empty slot
        { "name": "Docs", "type": 0, "url": "https://docs.example.org/",
          "visual": { "noTitle": false, "noIcon": false, "mask": false },
          "thumbnail": { "type": 0, "value": 0, "position": 1, "background": "" } }
      ]
    }
  ],
  "metadata": { "tags": [ { "id": 1, "name": "Work", "icon": 36, "color": "#0095ff" } ], "folders": [] },
  "options": { "THEME": "sharp", "DIAL_PADDING": 12, "...": "about 70 keys" },
  "___thumbnails": [
    { "url": "https://news.example.com/", "group": 5, "icon": "data:image/png;base64,...", "img": "data:image/webp;base64,...", "version": 1700000000000 }
  ]
}
```

### Quirks the mapper must handle

| # | Quirk | Mapping rule |
|---|---|---|
| G1 | `groups[].dials` is **always exactly `rows * cols` long**. `{}` is an empty slot; the array index is the grid position. ~48% of slots are empty. | Skip `{}`. Keep the index as `slotIndex`, keep `rows`/`cols` as group layout. |
| G2 | `created` is in **seconds**. All other timestamps in the file are milliseconds. | `createdAt = created * 1000`. |
| G3 | `created` is missing on ~12% of dials. | Fall back to import time. |
| G4 | `name` is missing on some dials. | Fall back to hostname. |
| G5 | The same URL can appear in several groups (~1%). | Allowed. Duplicate handling is a preview-step decision, not a mapper decision. |
| G6 | Thumbnails live in a separate table `___thumbnails`, joined on `(url, group)`. Every thumbnail matched a dial in the sample. | `icon` becomes the favicon, `img` (webp, optional) the thumbnail. Store as blobs, in batches. |
| G7 | `metadata.tags` defines tags (name, color, numeric icon id), but **no dial references a tag** in the sample. | Import tag definitions; no dial assignment. |
| G8 | `metadata.folders` is empty in the sample. **Folder format unknown.** | Do not invent a mapping. Log and ignore until a sample exists. |
| G9 | `dials[].type` was always `0` (URL dial). | Only `type: 0` supported; other types are reported as skipped. |
| G10 | `background`/`color` exist on some groups only. | Optional group appearance. |
| G11 | `options`, `productivity_mode`, `sync`, `last_user_id`, `_*` keys. | Ignored. Settings import is "later" (see PLAN.md 3.7). |

---

## 2. FVD Speed Dial ("Speed Dial for Chrome")

- File: `.txt`, JSON. **Real sample is tiny (8 dials, 2 groups). Mapper confidence is low.** Expect surprises with
  larger exports.

### Shape

```jsonc
{
  "db": {
    "groups": [ { "id": 4, "global_id": "AbC123", "name": "Tools", "position": 1, "sync": 1, "rowid": 4 } ],
    "dials": [
      {
        "id": 380, "rowid": 380, "global_id": "xYz789",
        "url": "https://tool.example.com/", "display_url": "https://tool.example.com/",
        "title": "Tool", "group_id": 4, "group_globalId": "AbC123", "position": 1,
        "thumb": "data:image/png;base64,...", "thumb_url": "", "thumb_source_type": "screen",
        "thumb_width": 400, "thumb_height": 300, "clicks": 3, "deny": 0
      }
    ],
    "deny": []
  },
  "prefs": { "sd.display_mode": "fancy", "...": "about 120 keys" }
}
```

### Quirks the mapper must handle

| # | Quirk | Mapping rule |
|---|---|---|
| F1 | The group with `global_id: "recommend"` contains **sponsored links injected by the vendor**: `url` is an affiliate redirect, the real site is in `display_url`. | Skip by default. Opt-in import uses `display_url`. |
| F2 | `clicks` is the only real usage data in either format. | Seed `openCount` for frecency. |
| F3 | `position` is 1-based, and has gaps. | Sort by position, renumber. |
| F4 | `thumb` is an inline data URL or `""`. `thumb_url` points to a remote image. | Use `thumb` only. Never fetch `thumb_url` during import (privacy). |
| F5 | `title` can be empty. | Fall back to hostname. |
| F6 | `group_globalId` is missing on some dials; `group_id` is always there. | Join on `group_id`. |
| F7 | `prefs.GROUPS_TRASH_KEY` / `DIALS_TRASH_KEY` may contain deleted items (empty in sample). | Ignore. |
| F8 | No creation timestamp per dial. | Import time. |

---

## 3. Test fixtures

- Synthetic fixtures under `src/core/import/__fixtures__/`, **one per quirk** (G1 to G11, F1 to F8), invented data only.
- An optional local test reads real files from `samples/` (git-ignored) when `DIALDEX_SAMPLES` is set. It is skipped in CI.
