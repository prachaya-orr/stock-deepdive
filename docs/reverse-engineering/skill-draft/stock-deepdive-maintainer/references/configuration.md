# Configuration and interface reference

## CLI flags (from argparse definitions)

| Script | Positional | Flags (default) |
|---|---|---|
| `fetch_us.py` | `ticker` | `--out` (`data/<T>`) · `--contact` (`$SEC_CONTACT`, required) · `--annuals 2` · `--quarters 4` · `--events 6` · `--no-exhibits` · `--no-facts` |
| `fetch_th.py` | `symbol` | `--out` (`data/<S>`) · `--periods 6` · `--no-text` · `--no-561` |
| `extract.py` | `folder` | `--no-pdf` |
| `build_paper.py` | — | `--parts` (req) · `--out` (req) · `--title` (req) · `--subtitle` · `--tag` (repeatable) · `--sources` |
| `check_numbers.py` | `report facts` | `--strict` · `--out` |

## Environment
- `SEC_CONTACT="Name email"` — required by EDGAR; put in User-Agent `stock-deepdive/1.0 (<contact>)`.
- Optional Python packages: `pymupdf` (import `fitz`) preferred, else `pypdf`.

## Exit codes
- 0 ok · 1 input/lookup failure · 2 = `fetch_us` missing contact, `build_paper` structure problems, `check_numbers` missing input.
- `check_numbers` returns 1 for orphans only with `--strict`.

## Formats consumed downstream
- Folder names: `FS_<YYYYMM>_<zipname[:20]>`, `R561_<…>`; US files `<FORM>_<DATE>_<ACCN>[_<EXTYPE>].<ext>`.
- Text markers: `===== ชีต: … =====`, `----- ตารางฝัง #n (file) -----` / `----- จบตารางฝัง -----`, `[[หน้า N]]`, `⟨ภาพสแกน — ข้อความไม่ครบ⟩`.
- `MANIFEST.md` table columns: `ไฟล์ต้นทาง | ไฟล์ที่อ่านได้ | ตัวอักษร | หมายเหตุ`.
- claims JSONL keys: `หัวข้อ`, `ข้อความ`, `ทิศ` (`บวก|ลบ|กลาง`), `ตัวเลขที่ใช้`.
- Computed-number marker: `[คำนวณ]` (exempts the whole line from the screen).

## CSS class vocabulary (must match `build_paper.py` CSS)
`tag` (+`red amber green blue`) · `box` (+`key warn quote good note`) · `tw` table wrapper · `num` · `pos neg mut` · `ladder rung head meta` · front page: `verdict big num`, `decode`, `toc`.
