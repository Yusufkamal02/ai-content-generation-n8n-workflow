# Aura Social Campaign — Content Generation API (GCP)

Workflow n8n: **Aura_Social_Campaign_Automation** (`XgzIotlw1Hn8lwAr`).
Deploy target: **Cloud Run**. Media output: **Google Cloud Storage** (callback kirim URL, bukan base64).
Model AI: **Google AI Studio API key** (Gemini + Veo).

Alur produk: `Campaign (manual) → Ide → Caption → Prompt → Content Generated`.
Campaign/Ide/Caption/Prompt ditangani **Backend + Frontend**. Workflow ini: **prompt → konten jadi (image / carousel / video) → upload GCS → callback**.

---

## 1. Endpoint

```
POST https://<CLOUD_RUN_URL>/webhook/aura-generate
Content-Type: application/json
```

### Request body

| field | wajib | keterangan |
|---|---|---|
| `callbackUrl` | ✅ | URL Backend penerima hasil (POST JSON). |
| `contentType` | ✅ | `image` \| `carousel` \| `video`. Default `image`. |
| `prompt` | ✅ (image/video) | Prompt final dari Backend. |
| `prompts` | — | Array prompt per-slide untuk carousel. |
| `carouselCount` | — | Jumlah slide (2–10). Default 4 / `prompts.length`. |
| `aspectRatio` | — | **image**: `1:1` `9:16` `16:9` `3:4` `4:3` `4:5` (default `1:1`). **video**: `9:16` `16:9` (default `9:16`). |
| `videoDuration` | — | **video**: `8` `16` `32` `48` detik. Default `8`. (1/2/4/6 klip Veo @8 dtk → digabung ffmpeg.) |
| `jobId` | — | ID job Backend. Kalau kosong pakai execution id. Dipakai sebagai path GCS: `<prefix>/<jobId>/...`. |
| `campaign`, `idea`, `caption` | — | Diteruskan apa adanya ke payload callback. |

### Response sync (< 1 dtk)

```json
{ "jobId": "...", "status": "accepted", "contentType": "image" }
```

Backend **tidak menunggu** di koneksi ini.

---

## 2. Callback (async) — POST ke `callbackUrl`

```json
{
  "jobId": "...",
  "status": "success" | "failed",
  "contentType": "image | carousel | video",
  "campaign": "...", "idea": "...", "caption": "...",
  "aspectRatio": "4:5",
  "videoDuration": 16,                     // null kalau bukan video

  "content": {
    // image / video:
    "kind": "image" | "video",
    "url":   "https://storage.googleapis.com/<bucket>/generated/<jobId>/image.png",
    "gsUri": "gs://<bucket>/generated/<jobId>/image.png",
    "mimeType": "image/png",
    "bytes": 1405493

    // carousel:
    // "kind": "carousel", "count": 3,
    // "slides": [ { "url": "...", "gsUri": "...", "mimeType": "image/png", "bytes": ... }, ... ]
  },

  "tokens": {
    "input": 129, "output": 1073, "total": 1202,
    "estimated": false,                    // true = image/carousel (token diestimasi)
    "byStep": [
      { "step": "plan", "input": 129, "output": 1073 },
      { "step": "veo",  "input": 0, "output": 0, "videoSeconds": 16 }
    ]
  },

  "restoreTokens": false,                  // true kalau status=failed → Backend refund kuota token user
  "restoreAmount": 0,                      // = tokens.total saat gagal

  "error": null,
  "generatedAt": "2026-09-10T13:41:00.000Z"
}
```

### Restore Token
- Setiap job hitung token LLM: **input** (`promptTokenCount`) + **output** (`candidatesTokenCount` + thinking tokens).
- **video**: token asli dari langkah *plan*. Generate Veo dihitung per-detik (`videoSeconds`), bukan token.
- **image / carousel**: node Gemini image n8n tak lapor usage → **estimasi** (`estimated: true`): input ≈ `panjang_prompt / 4`, output `1290` per gambar.
- Job **gagal** → `restoreTokens: true`, `restoreAmount = tokens.total` → Backend kembalikan ke kuota user.
- Job **sukses** → Backend potong `tokens.total` dari kuota user.
- Node ber-`onError: continueRegularOutput` → workflow selalu selesai & kirim callback. Tidak pernah hang tanpa kabar.

---

## 3. Setup di n8n (sebelum activate)

### a. Node `Config` (klik, edit 3 nilai)
| field | isi |
|---|---|
| `gcsBucket` | nama GCS bucket, mis. `aura-content-media` |
| `gcsPrefix` | prefix object, mis. `generated` |
| `filesDir` | `/tmp/aura` (jangan diubah untuk Cloud Run) |

### b. Kredensial
| node | kredensial |
|---|---|
| `Gemini - Image`, `Gemini - Carousel Image`, `Gemini - Plan Video Scenes`, `Gemini - Video Scene (Veo)` | **Google Gemini (PaLM) API** (API key AI Studio) — sudah terpasang |
| `GCS - Upload` | **Google Service Account** — buat SA dengan role **Storage Object Creator** (atau Admin) di bucket, download JSON key, masukkan ke n8n |

### c. Bucket GCS
- Buat bucket (region sama dengan Cloud Run).
- Akses object: pilih salah satu —
  - **Public read** (uniform bucket-level access + `allUsers: objectViewer`) → `content.url` langsung bisa dipakai Frontend.
  - **Private** → Frontend/Backend generate Signed URL sendiri dari `content.gsUri`.
- Lifecycle rule opsional: auto-delete object > N hari.

---

## 4. Deploy Cloud Run

### Dockerfile (n8n + ffmpeg)
```dockerfile
FROM n8nio/n8n:latest
USER root
RUN apk add --no-cache ffmpeg     # image n8n berbasis Alpine
USER node
```
(kalau pakai image Debian: `apt-get update && apt-get install -y ffmpeg`)

### Env vars Cloud Run
```
N8N_HOST=<cloud-run-url-host>
N8N_PROTOCOL=https
WEBHOOK_URL=https://<cloud-run-url>/
N8N_PORT=5678
N8N_RESTRICT_FILE_ACCESS_TO=/tmp/aura
NODES_EXCLUDE=["n8n-nodes-base.localFileTrigger"]
N8N_ENCRYPTION_KEY=<samakan dgn instance lama biar kredensial kebaca, atau set baru & re-input kredensial>
DB_TYPE=postgresdb   # WAJIB untuk Cloud Run (SQLite hilang tiap restart) — pakai Cloud SQL
DB_POSTGRESDB_HOST=... DB_POSTGRESDB_DATABASE=... DB_POSTGRESDB_USER=... DB_POSTGRESDB_PASSWORD=...
GENERIC_TIMEZONE=Asia/Jakarta
```

### Flag deploy (WAJIB — workflow lanjut kerja SETELAH balas HTTP)
```
gcloud run deploy aura-n8n \
  --image <IMAGE> \
  --region <REGION> \
  --min-instances 1 \
  --no-cpu-throttling \        # CPU always allocated — kalau tidak, proses async mati setelah respons
  --timeout 3600 \
  --memory 2Gi --cpu 2 \
  --concurrency 4 \
  --service-account <SA-dengan-akses-bucket> \
  --allow-unauthenticated     # webhook publik; amankan dengan header secret di Normalize kalau perlu
```

> **Penting**: tanpa `--no-cpu-throttling` + `--min-instances 1`, Cloud Run mematikan CPU setelah `Respond - Accepted` balas → generate image/video tidak jalan. Alternatif lebih kokoh: n8n **queue mode** (worker terpisah) atau deploy di **Compute Engine VM**.

---

## 5. Contoh cURL

```bash
EP=https://<CLOUD_RUN_URL>/webhook/aura-generate
CB=https://backend.aura/api/aura/callback

# Image 4:5
curl -X POST $EP -H 'content-type: application/json' -d "{\"callbackUrl\":\"$CB\",\"contentType\":\"image\",\"aspectRatio\":\"4:5\",\"prompt\":\"siswa SMA merakit smart city kit, neon biru\"}"

# Carousel 3 slide 1:1
curl -X POST $EP -H 'content-type: application/json' -d "{\"callbackUrl\":\"$CB\",\"contentType\":\"carousel\",\"carouselCount\":3,\"aspectRatio\":\"1:1\",\"prompt\":\"infografik fitur Aura Smart City Kit, flat modern biru\"}"

# Video 16 dtk 16:9
curl -X POST $EP -H 'content-type: application/json' -d "{\"callbackUrl\":\"$CB\",\"contentType\":\"video\",\"aspectRatio\":\"16:9\",\"videoDuration\":16,\"prompt\":\"kota biasa berubah jadi kota pintar futuristik, sinematik\"}"
```

---

## 6. Integrasi Frontend

- **Content type**: dropdown `image` / `carousel` / `video`.
- **Aspect Ratio** per content type — image: `1:1, 9:16, 16:9, 3:4, 4:3, 4:5` · video: `9:16, 16:9`.
- **Durasi video**: dropdown `8, 16, 32, 48` detik (khusus video).
- Tampilkan `tokens.input / output / total` dari callback (badge "estimasi" bila `tokens.estimated`).
- Media: pakai `content.url` (image/video) atau `content.slides[].url` (carousel).

---

## Status uji (sebelum penyesuaian GCP, model & pipeline sudah terverifikasi)
| tipe | hasil |
|---|---|
| image | ✅ generate + token accounting + base64 delivery |
| carousel | ✅ 3 slide generate |
| video 8/16 dtk | ✅ Veo + ffmpeg concat + real token dari plan call |

Yang **belum diuji** di versi GCP ini: node `GCS - Upload` (butuh bucket + SA), path `/tmp/aura` (butuh `N8N_RESTRICT_FILE_ACCESS_TO`). Uji pertama sebaiknya langsung di Cloud Run staging.
