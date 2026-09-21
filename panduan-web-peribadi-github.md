# Panduan Membina Web Peribadi Akademik di GitHub

Nota perkongsian webinar anjuran Biro Pembangunan dan Budaya Ilmu, Institusi Pembangunan Felo (IPF), UTM.

Disediakan oleh: Assoc. Prof. Dr. Mohd Murtadha bin Mohamad, Fakulti Komputeran, UTM

---

## 1. Tujuan dan hasil akhir

Membina web peribadi percuma di GitHub Pages yang:

- bermula daripada CV sebagai satu sumber kandungan
- memaparkan senarai penerbitan daripada ORCID dan Scopus
- dikemas kini secara automatik setiap minggu tanpa kerja manual

Panduan ini ditulis untuk penyelidik yang bukan pengaturcara. Setiap langkah boleh diikuti dengan menyalin dan menampal.

## 2. Gambaran seni bina

```
CV (cv.json)  ─┐
ORCID API     ─┼─>  GitHub Actions (berjadual)  ─>  data/*.json  ─>  GitHub Pages  ─>  Web anda
Scopus API    ─┘         skrip Python                 (commit)         (index.html + JS)
```

Prinsip utama: **data dipisahkan daripada paparan**. Anda hanya mengemas kini data, laman akan mengikut.

## 3. Prasyarat

| Item | Keterangan |
|---|---|
| Akaun GitHub | Daftar di github.com |
| ORCID iD | Rekod ORCID dengan senarai karya yang ditetapkan sebagai awam |
| Scopus Author ID | Diperoleh daripada profil pengarang di Scopus |
| Scopus API key | Daftar di Elsevier Developer Portal (dev.elsevier.com) |
| Penyunting teks | VS Code atau editor web GitHub |

Nota: capaian dan kuota Scopus API bergantung pada akaun dan langganan institusi. Semak terma dan had semasa di portal Elsevier sebelum menggunakannya secara berkala.

## 4. Langkah 1: Sediakan repositori dan GitHub Pages

1. Cipta repositori baharu bernama `<username>.github.io` (gantikan `<username>` dengan nama pengguna GitHub anda). Tetapkan sebagai Public.
2. Cipta struktur berikut:

```
<username>.github.io/
├── index.html
├── assets/
│   ├── style.css
│   └── app.js
├── data/
│   ├── cv.json
│   ├── orcid.json
│   └── scopus.json
├── scripts/
│   ├── fetch_orcid.py
│   └── fetch_scopus.py
└── .github/
    └── workflows/
        └── update-data.yml
```

3. Buka Settings > Pages. Pada Source, pilih Deploy from a branch, kemudian pilih branch `main` dan folder `/ (root)`.
4. Tunggu beberapa minit. Web anda tersedia di `https://<username>.github.io`.

Kandungan `index.html` yang minimum:

```html
<!doctype html>
<html lang="ms">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Nama Anda | Profil Akademik</title>
  <link rel="stylesheet" href="assets/style.css">
</head>
<body>
  <header>
    <h1 id="nama"></h1>
    <p id="jawatan"></p>
  </header>
  <main>
    <section>
      <h2>Penerbitan</h2>
      <ul id="penerbitan"></ul>
    </section>
  </main>
  <script src="assets/app.js"></script>
</body>
</html>
```

## 5. Langkah 2: Jadikan CV sebagai data berstruktur

Pecahkan CV kepada seksyen dan simpan dalam `data/cv.json`. Anda boleh meminta Claude atau alat AI lain menukar teks CV kepada format ini, kemudian semak sendiri setiap medan.

```json
{
  "nama": "Nama Penuh Anda",
  "jawatan": "Jawatan, Jabatan, Fakulti, Universiti",
  "emel": "emel@contoh.my",
  "bidang_penyelidikan": ["Bidang 1", "Bidang 2"],
  "pendidikan": [
    { "ijazah": "PhD", "institusi": "Nama Universiti", "tahun": 2010 }
  ],
  "pengalaman": [
    { "peranan": "Nama peranan", "organisasi": "Nama organisasi", "mula": 2015, "tamat": null }
  ],
  "geran": [],
  "penyeliaan": [],
  "anugerah": []
}
```

Peringatan privasi: jangan letakkan nombor telefon peribadi, alamat rumah, nombor kad pengenalan atau nombor staf jika tidak perlu dipaparkan secara awam.

## 6. Langkah 3: Tarik data daripada ORCID

ORCID menyediakan Public API yang boleh dibaca tanpa kunci. Hanya rekod dengan keterlihatan awam akan muncul.

Simpan sebagai `scripts/fetch_orcid.py`:

```python
import json
import os
import requests

ORCID = os.environ["ORCID_ID"]  # contoh: 0000-0000-0000-0000
URL = f"https://pub.orcid.org/v3.0/{ORCID}/works"

r = requests.get(URL, headers={"Accept": "application/json"}, timeout=30)
r.raise_for_status()

hasil = []
for kumpulan in r.json().get("group", []):
    ringkasan = kumpulan["work-summary"][0]
    tajuk = ringkasan["title"]["title"]["value"]

    tarikh = ringkasan.get("publication-date") or {}
    tahun = (tarikh.get("year") or {}).get("value")

    jurnal = (ringkasan.get("journal-title") or {}).get("value")

    doi = None
    for e in (kumpulan.get("external-ids") or {}).get("external-id", []):
        if e.get("external-id-type") == "doi":
            doi = e.get("external-id-value")
            break

    hasil.append({
        "tajuk": tajuk,
        "tahun": int(tahun) if tahun else None,
        "jurnal": jurnal,
        "jenis": ringkasan.get("type"),
        "doi": doi,
    })

hasil.sort(key=lambda x: x["tahun"] or 0, reverse=True)

with open("data/orcid.json", "w", encoding="utf-8") as f:
    json.dump(hasil, f, ensure_ascii=False, indent=2)

print(f"{len(hasil)} karya disimpan.")
```

Ujian setempat:

```bash
pip install requests
ORCID_ID=0000-0000-0000-0000 python scripts/fetch_orcid.py
```

## 7. Langkah 4: Tarik data daripada Scopus

Simpan sebagai `scripts/fetch_scopus.py`:

```python
import json
import os
import requests

KEY = os.environ["SCOPUS_API_KEY"]
AU_ID = os.environ["SCOPUS_AUTHOR_ID"]

URL = "https://api.elsevier.com/content/search/scopus"
HEADERS = {"X-ELS-APIKey": KEY, "Accept": "application/json"}

mula, bilangan, semua = 0, 25, []

while True:
    r = requests.get(
        URL,
        headers=HEADERS,
        params={
            "query": f"AU-ID({AU_ID})",
            "start": mula,
            "count": bilangan,
            "sort": "-coverDate",
        },
        timeout=30,
    )
    r.raise_for_status()
    hasil = r.json()["search-results"]
    entri = hasil.get("entry", [])

    # Set hasil kosong dipulangkan sebagai satu entri yang mengandungi "error"
    if not entri or "error" in entri[0]:
        break

    semua.extend(entri)
    mula += bilangan
    if mula >= int(hasil["opensearch:totalResults"]):
        break

ringkas = [
    {
        "tajuk": e.get("dc:title"),
        "jurnal": e.get("prism:publicationName"),
        "tarikh": e.get("prism:coverDate"),
        "doi": e.get("prism:doi"),
        "petikan": int(e.get("citedby-count", 0)),
        "jenis": e.get("subtypeDescription"),
    }
    for e in semua
]

with open("data/scopus.json", "w", encoding="utf-8") as f:
    json.dump(ringkas, f, ensure_ascii=False, indent=2)

print(f"{len(ringkas)} rekod Scopus disimpan.")
```

Pilihan: metrik pengarang (h-index dan jumlah petikan) boleh diambil melalui Author Retrieval API, iaitu `https://api.elsevier.com/content/author/author_id/{AU_ID}?view=METRICS`. Semak dokumentasi Elsevier dan respons sebenar akaun anda, kerana medan yang dipulangkan bergantung pada tahap akses.

Untuk menggabungkan ORCID dan Scopus, padankan rekod menggunakan DOI supaya satu karya tidak dipaparkan dua kali.

## 8. Langkah 5: Automasikan dengan GitHub Actions

1. Buka Settings > Secrets and variables > Actions.
2. Tab **Secrets**: tambah `SCOPUS_API_KEY`.
3. Tab **Variables**: tambah `ORCID_ID` dan `SCOPUS_AUTHOR_ID`.
4. Simpan fail `.github/workflows/update-data.yml`:

```yaml
name: Kemas kini data penerbitan

on:
  schedule:
    - cron: "0 2 * * 1"   # setiap Isnin, 02:00 UTC
  workflow_dispatch:       # butang untuk jalankan secara manual

permissions:
  contents: write

jobs:
  kemas-kini:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - run: pip install requests

      - name: Tarik data ORCID
        run: python scripts/fetch_orcid.py
        env:
          ORCID_ID: ${{ vars.ORCID_ID }}

      - name: Tarik data Scopus
        run: python scripts/fetch_scopus.py
        env:
          SCOPUS_API_KEY: ${{ secrets.SCOPUS_API_KEY }}
          SCOPUS_AUTHOR_ID: ${{ vars.SCOPUS_AUTHOR_ID }}

      - name: Commit jika ada perubahan
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add data/
          git diff --cached --quiet || (git commit -m "Kemas kini data penerbitan" && git push)
```

5. Buka tab Actions, pilih workflow ini, kemudian klik **Run workflow** untuk ujian pertama.

Nota: GitHub boleh melumpuhkan workflow berjadual pada repositori awam yang tiada aktiviti selama 60 hari. Jika ini berlaku, aktifkan semula di tab Actions.

## 9. Langkah 6: Paparkan data pada laman

Simpan sebagai `assets/app.js`:

```js
async function muat(laluan) {
  const r = await fetch(laluan);
  if (!r.ok) throw new Error(`Gagal memuat ${laluan}`);
  return r.json();
}

function itemPenerbitan(p) {
  const li = document.createElement("li");
  const tajuk = document.createElement("strong");
  tajuk.textContent = p.tajuk;
  li.append(tajuk, ` (${p.tahun ?? "t.t."}) `);

  if (p.jurnal) li.append(document.createTextNode(`${p.jurnal} `));

  if (p.doi) {
    const a = document.createElement("a");
    a.href = "https://doi.org/" + p.doi;
    a.textContent = "DOI";
    li.append(a);
  }
  return li;
}

async function mula() {
  const cv = await muat("data/cv.json");
  document.getElementById("nama").textContent = cv.nama;
  document.getElementById("jawatan").textContent = cv.jawatan;

  const senarai = await muat("data/orcid.json");
  document.getElementById("penerbitan").replaceChildren(
    ...senarai.map(itemPenerbitan)
  );
}

mula().catch(console.error);
```

Kod di atas menggunakan `textContent` dan bukan `innerHTML` supaya kandungan daripada API luar tidak boleh menyuntik skrip ke dalam laman anda.

## 10. Keselamatan dan etika

- **API key** hanya disimpan dalam GitHub Secrets. Jangan taip terus dalam kod, jangan commit fail `.env`.
- Jika kunci terdedah secara tidak sengaja, batalkan dan jana kunci baharu di portal Elsevier.
- Repositori awam boleh dilihat sesiapa. Semak setiap fail sebelum commit.
- Semak terma penggunaan Elsevier API sebelum memaparkan data Scopus secara awam, termasuk had penyimpanan dan pemaparan.
- Data ORCID ialah data yang anda sendiri tetapkan sebagai awam. Kemas kini rekod ORCID anda dahulu, kerana kualiti web bergantung pada kualiti rekod sumber.
- Semak ketepatan senarai karya secara berkala. Data automatik tidak menggantikan penilaian penyelidik.

## 11. Penyelenggaraan

| Kekerapan | Tugas |
|---|---|
| Setiap minggu | Automatik: workflow menarik data terkini |
| Setiap bulan | Semak tab Actions untuk memastikan workflow tidak gagal |
| Setiap semester | Kemas kini `cv.json`, semak pautan, buang maklumat lapuk |
| Setiap tahun | Semak terma API dan pusing semula API key |

## 12. Masalah lazim

| Masalah | Kemungkinan punca dan tindakan |
|---|---|
| Web tidak muncul selepas Pages diaktifkan | Tunggu beberapa minit, semak nama repositori mesti `<username>.github.io` |
| `orcid.json` kosong | Karya ORCID belum ditetapkan sebagai awam |
| Ralat 401 atau 403 daripada Scopus | API key salah, atau akaun tiada akses. Semak di portal Elsevier |
| Ralat 429 | Had kuota dicapai. Kurangkan kekerapan jadual |
| Workflow gagal push | Pastikan `permissions: contents: write` ada dalam fail workflow |
| Data tidak berubah pada laman | Muat semula tanpa cache (Ctrl+F5), semak sama ada commit baharu wujud |

## 13. Rujukan rasmi

- GitHub Pages: https://docs.github.com/en/pages
- GitHub Actions: https://docs.github.com/en/actions
- GitHub Secrets: https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions
- ORCID Public API: https://info.orcid.org/documentation/features/public-api/
- Elsevier Developer Portal: https://dev.elsevier.com

Pautan dokumentasi boleh berubah. Jika satu pautan tidak lagi berfungsi, cari halaman terkini melalui laman utama tapak berkenaan.

## 14. Lesen dan penggunaan

Panduan ini boleh digunakan dan diubah suai untuk tujuan pembelajaran. Sila nyatakan sumber jika dikongsi semula.
