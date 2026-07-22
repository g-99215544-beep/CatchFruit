# Reka Bentuk: Mode 2 Pemain (Player vs Player)

Tarikh: 2026-07-22
Fail terlibat: `index.html` (satu fail sahaja)

## Ringkasan

Tambah mode ketiga "👥 2 Pemain" pada game Tangkap Buah. Dua pemain berkongsi satu
skrin kamera. Player 1 tangkap buah dengan tangan **terbuka** (✋), Player 2 dengan
tangan **genggam** (✊). Selepas masa tetap (30/45/60 saat), markah tertinggi menang.

## Keputusan yang disahkan

- **Pembezaan pemain:** ikut gerak isyarat sahaja (bukan kawasan skrin). Buah yang
  disentuh tangan terbuka → P1; disentuh tangan genggam → P2. Skrin dikongsi penuh.
- **Bom:** ada (~25%, sama seperti mode Normal). Tangan terbuka kena bom → penalti P1;
  tangan genggam kena bom → penalti P2.
- **Skrin keputusan:** papar pemenang + markah kedua-dua pemain + butang Main Lagi.
  Tiada simpan ke papan ranking.
- **Tempoh:** 30 / 45 / 60 saat (guna semula kawalan sedia ada). Kelajuan boleh pilih.

## Perubahan state

Ganti medan skor tunggal dengan medan per-pemain:

- Tambah nilai mode: `state.mode` boleh jadi `'2player'`.
- Tambah: `p1Score, p2Score, p1Combo, p2Combo, p1MaxCombo, p2MaxCombo` (semua mula 0).
- Medan `score/combo/maxCombo` sedia ada kekal digunakan oleh mode Normal & Survival.

## Logik tangkap (`update()`)

Mode single (Normal/Survival) kekal: hanya tangan terbuka tangkap.

Mode 2P:
- **Semua** tangan (terbuka & genggam) boleh menuntut satu buah terdekat yang bertindih.
- Gunakan semula set `caughtFruits` supaya satu buah tak dikira dua kali dalam satu frame.
- Buah biasa: `hand.open` → guna `handleCatch2P('p1', f)`; jika tidak → `handleCatch2P('p2', f)`.
- Bom: `hand.open` → penalti P1 (tolak 30, kombo P1 = 0); jika tidak → penalti P2.
- Buah tersasar jatuh ke bawah: **tidak** memutuskan kombo sesiapa (tak boleh ditentukan
  pemilik). Kombo pemain hanya reset bila pemain itu kena bom.
- Pengganda markah kombo per-pemain kekal seperti sekarang: `1 + floor(combo/5)*0.5`.

## HUD (mode 2P sahaja)

- Susun semula: kiri = markah P1 (label `P1 ✋`, warna hijau), tengah = MASA,
  kanan = markah P2 (label `P2 ✊`, warna oren). Kombo dipapar kecil.
- Chip KOMBO tunggal sedia ada disembunyikan dalam 2P (kombo dipapar per-pemain).
- Bulatan tangan pada kanvas: mode 2P → terbuka = cincin hijau, genggam = cincin oren
  (kedua-dua aktif). Mode single kekal: terbuka = hijau, genggam = kelabu putus-putus.
- Teks poin timbul diwarna ikut pemain (P1 emas, P2 oren).

## UI Skrin Mula

- Tambah butang mode ketiga `data-mode="2player"`: **👥 2 Pemain**.
- Bila 2P dipilih: papar kawalan tempoh (30/45/60) + kelajuan (guna semula
  `normal-options`), plus nota ringkas peraturan 2P (P1 buka tangan, P2 genggam).
- Logik butang mode sedia ada dikemas kini untuk kendali tiga pilihan.

## Kalibrasi jarak

Guna semula tanpa perubahan. Tangan terbuka P1 akan mencetuskan auto-kalibrasi lebar
tapak tangan seperti sedia ada. Pengawal jarak (`updateDistanceGuard`) guna tapak tangan
terlebar antara semua tangan — kekal berfungsi.

## Skrin keputusan (baru)

Skrin ringkas `screen-result-2p`:
- Tajuk: `🏆 Player 1 Menang!` / `🏆 Player 2 Menang!` / `🤝 Seri!` ikut perbandingan skor.
- Papar kedua-dua markah (P1 dan P2) dengan warna masing-masing.
- Butang **🔁 Main Lagi** → kembali ke skrin mula.
- Tiada input nama, tiada simpan ranking.

`endGame()` dikemas kini: jika mode 2P, tunjuk `screen-result-2p`; jika tidak, aliran
sedia ada (skrin gameover + simpan ranking) kekal.

## Skop TIDAK disentuh

Mode Normal, mode Survival, papan ranking (localStorage), avatar kepala (FaceLandmarker),
kesan flash/shake/bunyi survival — semua kekal tidak berubah.
