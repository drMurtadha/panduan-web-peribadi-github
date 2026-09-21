# Panduan Membina Web Peribadi Akademik di GitHub

Bahan webinar anjuran Biro Pembangunan dan Budaya Ilmu, Institusi Pembangunan Felo (IPF), UTM. Disediakan oleh Assoc. Prof. Dr. Mohd Murtadha bin Mohamad, Fakulti Komputeran, UTM.

**[Lawati laman interaktif](https://drmurtadha.github.io/panduan-web-peribadi-github/)** · [Baca fail Markdown](panduan-web-peribadi-github.md)

Panduan ini menerangkan cara membina laman profil akademik percuma dengan GitHub Pages, menyusun CV sebagai data JSON, menarik karya daripada ORCID dan Scopus, serta menjadualkan kemas kini melalui GitHub Actions. Ia ditulis untuk penyelidik yang bukan pengaturcara.

## Kemas kini laman

Kandungan panduan berada dalam `panduan-web-peribadi-github.md`; reka bentuk laman berada dalam `src/pages/index.astro`. Setiap perubahan yang dihantar ke cabang `main` dibina dan diterbitkan secara automatik oleh `.github/workflows/deploy.yml`.

Untuk melihatnya pada komputer sendiri:

```bash
npm ci
npm run dev
```

Sediakan akaun GitHub, ORCID iD, Scopus Author ID dan Scopus API key untuk mencuba contoh dalam panduan. Semak data peribadi sebelum menerbitkan repositori awam.

> Contoh JavaScript dalam panduan memaparkan data ORCID. Untuk memaparkan data Scopus bersama ORCID, tambah langkah penggabungan dan padankan karya menggunakan DOI.
