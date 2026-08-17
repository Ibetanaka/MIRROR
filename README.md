# ANS + Agent Scanner — Infrastruktur untuk LitVM Testnet

Infrastruktur **Agent Name Service (ANS)** dan **skill scanner** untuk agent AI on-chain,
dibangun untuk **LitVM testnet** ("LiteForge") — Layer-2 EVM di atas Litecoin (Arbitrum Orbit stack).

| | |
|---|---|
| Chain ID | `4441` |
| Gas token | `zkLTC` |
| RPC | `https://liteforge.rpc.caldera.xyz/infra-partner-http` |
| Explorer | `https://liteforge.caldera.xyz` |

## Struktur proyek

```
litvm-ans/
├── contracts/
│   ├── ANSRegistry.sol   # pendaftaran nama -> alamat agent
│   └── AgentScorer.sol   # scanner: kirim & baca skor speed/decision/execution
├── scripts/deploy.js      # deploy kedua kontrak ke LitVM testnet
├── hardhat.config.js
├── package.json
├── .env.example
└── frontend/index.html    # dashboard: registry, scanner, leaderboard (1 file, tanpa build step)
```

## 1. Apa yang dibangun

**`ANSRegistry.sol`**
- `register(name, agentAddress, metadataURI)` — klaim nama unik untuk sebuah agent.
- `resolve(name)` — nama → alamat agent, owner, metadata, status aktif.
- `nameOfAgent(address)` — reverse lookup, alamat → nama.
- `updateAgentAddress`, `transferName`, `deactivate` — pengelolaan nama oleh pemiliknya.

**`AgentScorer.sol`**
- Peran **scanner**: alamat yang diotorisasi admin (`addScanner`) mengirim hasil pemindaian task lewat `submitScan(agent, speed, decision, execution, success, taskId)`.
- Tiga skala penilaian, masing-masing 0–100:
  - **speed** — kecepatan agent menyelesaikan task relatif terhadap target waktu.
  - **decision** — kualitas keputusan yang diambil selama eksekusi.
  - **execution** — apakah hasil akhir benar-benar tercapai sesuai target.
- Skor komposit dihitung on-chain sebagai rata-rata berbobot (default 1/3, 1/3, 1/3 — bisa diubah admin lewat `setWeights`).
- `getLeaderboardPage(offset, limit)` menyediakan data untuk leaderboard tanpa perlu indexer eksternal.
- Histori lengkap per agent tersimpan (`getHistoryAt`) untuk audit.

Siapa yang berperan sebagai "scanner" terserah Anda: bisa berupa skrip off-chain yang memonitor eksekusi agent (mis. membaca log task, latensi respons API, hasil transaksi on-chain agent), lalu menuliskan skornya lewat `submitScan`.

## 2. Deploy ke LitVM testnet

```bash
npm install
cp .env.example .env
# isi .env dengan PRIVATE_KEY deployer (harus punya saldo zkLTC — klaim di faucet LitVM testnet)

npm run compile
npm run deploy:litvm
```

Script akan mencetak dua alamat kontrak. Tempelkan ke `frontend/index.html` pada bagian:

```js
const CONFIG = {
  ...
  registryAddress: "0x...",  // alamat ANSRegistry
  scorerAddress:   "0x...",  // alamat AgentScorer
};
```

`scripts/deploy.js` sudah otomatis melakukan dua hal ini setelah deploy:

```js
const CREATOR_ADDRESS = "0x2768ef0331cfde4cab0ffbf989c8f9d622c64c10";
// 1) await scorer.addScanner(CREATOR_ADDRESS);   -> boleh mengirim submitScan
// 2) await registry.setAdmin(CREATOR_ADDRESS);   -> jadi admin ANSRegistry
//    await scorer.setAdmin(CREATOR_ADDRESS);     -> jadi admin AgentScorer
```

Setelah deploy, **hanya `CREATOR_ADDRESS`** yang punya kendali admin: mengatur `registrationFee`,
mengubah bobot skor (`setWeights`), menambah/mencabut scanner lain (`addScanner`/`removeScanner`),
dan menarik dana kontrak (`withdraw`). Wallet deployer (dari `.env`) tidak lagi punya hak admin setelah
transaksi ini selesai — pastikan `CREATOR_ADDRESS` benar sebelum menjalankan `npm run deploy:litvm`,
karena transfer admin ini tidak bisa dibatalkan kecuali lewat wallet `CREATOR_ADDRESS` itu sendiri.

Untuk menambahkan alamat scanner lain (mis. wallet skrip monitoring tambahan) setelah deploy, panggil
lewat wallet `CREATOR_ADDRESS`:

```js
// lewat console ethers.js atau script terpisah, ditandatangani oleh CREATOR_ADDRESS
await scorer.addScanner("0xAlamatScannerLain");
```

## 3. Menjalankan aplikasi (frontend)

`frontend/index.html` adalah aplikasi mandiri — tidak butuh build step. Cukup buka filenya di browser
(atau serve dengan `npx serve frontend`), lalu:

1. Klik **Sambungkan Wallet** — aplikasi otomatis meminta MetaMask/Rabby untuk menambahkan/beralih ke LitVM testnet.
2. **Registry** — daftarkan nama untuk agent Anda, atau cari agent lain berdasarkan nama.
3. **Scanner** — kirim hasil pemindaian task (hanya berhasil jika wallet Anda sudah diotorisasi lewat `addScanner`).
4. **Leaderboard** — daftar agent terurut berdasarkan skor komposit, dengan rincian tiga skala penilaian.

Sebelum kontrak di-deploy dan `CONFIG` diisi, aplikasi otomatis berjalan dalam **mode pratinjau** dengan data
contoh, supaya alur UI tetap bisa dicoba sebelum infrastruktur on-chain-nya siap.

## 4. Alur skoring yang disarankan

1. Agent didaftarkan lewat `ANSRegistry.register`.
2. Setiap kali agent menyelesaikan sebuah task (skill tertentu), proses monitoring off-chain Anda mencatat:
   - waktu respons/penyelesaian → dikonversi jadi skor **speed**,
   - jejak keputusan yang diambil (rute, strategi, prioritas) → dinilai jadi skor **decision**,
   - hasil akhir tercapai atau tidak → skor **execution** + flag `success`.
3. Skrip scanner memanggil `submitScan` (atau `submitScanBatch` untuk banyak hasil sekaligus, lebih hemat gas).
4. Frontend membaca `getAgentScore` / `getLeaderboardPage` untuk menampilkan reputasi agent secara real-time.

## Catatan keamanan

- `registrationFee` default 0 zkLTC — naikkan lewat `setRegistrationFee` bila ingin mencegah spam nama di testnet ramai.
- Peran scanner terpusat pada admin kontrak; untuk produksi, pertimbangkan multi-sig atau staking/slashing agar
  penilaian tidak bisa dimanipulasi satu pihak.
- Ini kode untuk **testnet**. Sebelum dipakai di mainnet LitVM, lakukan audit dan pertimbangkan rate-limiting
  pada `submitScan` untuk mencegah spam skor.
