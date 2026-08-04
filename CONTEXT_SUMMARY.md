# Context Summary — Kalori Tracker

**Tipe**: Claude.ai React artifact (bundled single HTML), bukan project Capacitor/Android — dibangun pakai `web-artifacts-builder` (React + TS + Vite + Tailwind + shadcn/ui, dibundle jadi 1 file HTML via Parcel).
**Tema**: Botanical Garden (fern green / marigold / terracotta / cream) + Figtree font, flat design mobile touch-first.
**Storage**: `window.storage` (Claude artifact persistent storage API) — personal, per-akun, tidak shared.

---

## 1. Struktur Folder

```
kalori-tracker/
├── index.html                  # entry HTML, load src/main.tsx
├── src/
│   ├── main.tsx                 # ReactDOM root, render <App/>
│   ├── App.tsx                  # SATU-SATUNYA file logic+UI aplikasi (lihat §3)
│   ├── index.css                # CSS variables tema (warna, radius) + font import
│   ├── data/
│   │   └── foods.ts             # database makanan built-in (const array, lihat §2)
│   ├── lib/
│   │   ├── storage.ts           # wrapper loadJSON/saveJSON ke window.storage
│   │   └── utils.ts             # util `cn()` bawaan shadcn (classnames merge)
│   └── components/ui/           # 40+ komponen shadcn/ui (dipakai: button, input,
│                                 # card, separator — sisanya terpasang tapi tidak dipakai)
└── bundle.html                  # OUTPUT akhir: hasil build, inilah yang di-share ke user
```

Tidak ada backend/server — 100% client-side, jalan di dalam sandbox artifact claude.ai.

---

## 2. Database

**Tidak pakai SQL/SQLite.** Ada 2 "tabel" data, disimpan sebagai JSON:

### 2a. Database makanan bawaan — `src/data/foods.ts`
- Constant array `FOODS: Food[]`, **130 item** makanan Indonesia (nasi, protein hewani/nabati, sayur, buah, susu, minyak, masakan warteg, gorengan/jajanan, minuman, bumbu).
- Hardcoded di source code, bukan file terpisah — jadi kalau mau nambah/edit massal, edit langsung file ini lalu build ulang.
- Skema per item (per 100 gram):
  ```ts
  interface Food {
    id: string;       // slug unik, mis. "dada-ayam"
    name: string;
    kcal: number;
    protein: number;  // gram
    carbs: number;    // gram
    fat: number;      // gram
    custom?: boolean; // true kalau ditambahkan user lewat form custom food
  }
  ```

### 2b. Data user (persisten via `window.storage`)
Disimpan sebagai key-value, personal (non-shared):

| Key | Isi | Tipe |
|---|---|---|
| `custom-foods` | Makanan custom yang ditambah user | `Food[]` |
| `target` | Target kalori & macro harian | `{kcal, protein, carbs, fat}` |
| `log:YYYY-MM-DD` | Log makanan yang dimakan pada tanggal tsb | `LogEntry[]` |

```ts
interface LogEntry {
  id: string;        // crypto.randomUUID()
  name: string;
  grams: number;      // porsi yang dicatat
  kcal, protein, carbs, fat: number;  // sudah dikali faktor grams/100, bukan per-100g lagi
}
```

Karena log dikunci per tanggal (`log:2026-08-04`), pindah hari otomatis dapat log kosong — ini yang jadi jawaban "reset harian" sebelumnya: bukan reset aktif, tapi partisi data per hari.

---

## 3. Arsitektur Komponen (`src/App.tsx`)

Satu file, 1 komponen utama + 5 komponen anak (semua di file yang sama, tidak dipecah — sesuai gaya ponytail: fewest files):

```
App (root)
├── DateNav          — navigasi tanggal (← Hari ini/Kemarin/tanggal →)
├── SummaryCard       — total kalori + 3x <Bar/> (protein/karbo/lemak)
│   └── Bar           — progress bar 1 macro (value/goal, warna beda per macro)
├── (search & log card)
│   └── CustomFoodForm — form tambah makanan custom (toggle open/close)
├── (log list card)
└── TargetDialog      — panel popover buat edit target harian
```

### Helper functions (murni, tanpa side-effect)
| Fungsi | Input | Output | Fungsi |
|---|---|---|---|
| `todayStr(offsetDays?)` | jumlah hari offset (default 0) | `"YYYY-MM-DD"` | tanggal hari ini ± offset |
| `fmtDateLabel(dateStr)` | `"YYYY-MM-DD"` | `"Hari ini"` / `"Kemarin"` / `"Sen, 3 Agu"` | label tanggal buat UI |
| `round1(n)` | number | number | bulatkan 1 desimal |

### Fungsi ber-efek (baca/tulis storage) — semua di dalam `App()`
| Fungsi | Input | Efek / Output |
|---|---|---|
| `persistLog(next)` | `LogEntry[]` baru | `setLog` + `saveJSON("log:<date>", next)` |
| `addEntry(food)` | `Food` yg dipilih dari hasil search + `gramsDraft[food.id]` | hitung kalori/macro proporsional gram, push ke log, panggil `persistLog` |
| `removeEntry(id)` | id log entry | filter log, panggil `persistLog` |
| `addCustomFood(food)` | `Food` baru dari form | prepend ke `customFoods`, `saveJSON("custom-foods", ...)` |
| `saveTarget(t)` | `Target` baru | `setTarget` + `saveJSON("target", t)` |

### Data flow (read)
1. Mount → `useEffect` load `custom-foods` + `target` dari storage (sekali).
2. `date` berubah → `useEffect` load `log:<date>` dari storage.
3. `allFoods` = `customFoods` + `FOODS` (custom foods diprioritaskan muncul duluan di hasil search).
4. `results` = filter `allFoods` by substring match ke `query`, maks 8 hasil.
5. `totals` = reduce semua `LogEntry` hari itu → total kcal/protein/carbs/fat.

### `src/lib/storage.ts`
```ts
loadJSON<T>(key, fallback): Promise<T>   // get + JSON.parse, fallback kalau gagal/kosong
saveJSON(key, value): Promise<void>      // JSON.stringify + set, silent-fail kalau storage error
```
Thin wrapper — tidak ada retry/caching/queue, sengaja (ponytail: `window.storage` sudah cukup, tidak butuh abstraksi tambahan).

---

## 4. Yang belum ada (sudah di-skip sesuai diskusi awal)
- Reset manual log harian (baru sebatas "hari baru = log kosong", bukan tombol "hapus semua")
- Barcode scan, sync antar device, grafik trend mingguan
- Dark mode (dihapus dari template shadcn, tidak dipakai)
- Search fuzzy/typo-tolerant (masih exact substring match)
