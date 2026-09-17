# track_e_image — কী দিয়ে কী চালাবে

সব notebook-এ `find()` আছে — Input-এর যেকোনো গভীরতায় ফাইল খুঁজে নেয়।
তাই dataset-এর ভেতরের ফোল্ডার-গঠন নিয়ে ভাবতে হবে না, শুধু **attach** করলেই হয়।

| # | notebook | Accel | Internet | Input | Output |
|---|---|---|---|---|---|
| E0 | `E0_build_img1280.ipynb` | **None** | OFF | astroclimb, essentials, (img384) | `img1280/` (~3GB) |
| E1 | `E1_hires_ocr.ipynb` | **None** | OFF | essentials, **img1280**, img384, ocr_384x2 | `ocr_hi.parquet` |
| E2 | `E2_ocr_embedding.ipynb` | **None** | OFF | essentials, **ocr_hi** | `emb_qvl_ocr.npy` |
| E3 | `E3_qwen_hires_embed.ipynb` | **GPU** | **ON** | essentials, **img1280** | `emb_qvl8_img.npy`, `emb_qvl8_txt.npy` |
| E4 | `E4_vlm_caption_hires.ipynb` | **GPU** | **ON** | essentials, **img1280** | `vlm_caption.parquet` ⚠️ wiring নেই |
| E5 | `All/plan_d_final.ipynb` | **None** | OFF | পুরনো সব + ocr_hi + emb_qvl_ocr + emb_qvl8_* | `submission.csv` |

`essentials`-এ অবশ্যই থাকতে হবে: `img_hashes.parquet` (১৪,৫৯৮ — সব notebook-এর join key),
`meta_train.parquet` (E4), `emb_qvl_txt.npy` (E3 কপি করে `emb_qvl8_txt.npy` বানায়)।

## ক্রম

```
E0 ──┬── E1 ── E2 ──┐
     │              ├── E5 (মাপা)
     └── E3 ────────┘        E4  শুধু E5 ইতিবাচক হলে
```
E0 সবার gate। এরপর E1+E2 (CPU) আর E3 (GPU) — আলাদা account-এ একসাথে।

## প্রতিটা run-এর পর
Output → **Save Version** → Output ট্যাব → **New Dataset**। `/kaggle/working`
টেকে না, তাই পরের notebook-এ ওই dataset-ই Input।

## Resume
session কাটলে: ওই run-এর Output-কে Dataset বানিয়ে **একই notebook-এ Input দাও**,
আবার চালাও। E0 `.jpg` skip করে, E1 `ocr_hi_part*.parquet` থেকে, E3 `_ck_qvl8_img.npy`
থেকে, E4 `vlm_caption.parquet` থেকে ধরে।

## থামার সংকেত (ভুল ধরা পড়লে সাথে সাথেই থামে)
- **E0** — বানানো hash set `img_hashes.parquet`-এর সাথে না মিললে assert fail।
- **E1** — ২০০-figure gate: নতুন OCR পুরনোর ৩× না হলে, বা ফাঁকা ৪%-এর বেশি হলে থামো।
- **E3/E4** — প্রথম ছবিতেই visual token ছাপে; ~১১২-তে আটকে থাকলে `max_pixels`
  কাজ করেনি → পুরো run বৃথা, ওখানেই থামো।
- **E5** — CELL 0-তে `ocr_hi.parquet` এবং space তালিকায় `qvl8_i`, `qvl8_t`, `qvl_o`
  দেখা না গেলে feature নীরবে বাদ পড়েছে।

E5 দুবার চালাও — একবার শুধু OCR, একবার OCR + `qvl8` — তাহলে কোন lever কতটা দিল আলাদা করে বোঝা যাবে।
