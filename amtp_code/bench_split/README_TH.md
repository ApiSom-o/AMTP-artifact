# bench_split — วัดเวลา search ของ server แยกจาก proof

โฟลเดอร์นี้แยกจากโค้ดเดิมทั้งหมด **ไม่แก้ไฟล์ใน `amtp/`, `bench/`, `baselines/` หรือ `scripts/` เลย** แค่เรียกใช้โค้ดเดิม

## ทำไม

อาจารย์ให้แยกการทดลองตามขั้น: search อย่างเดียว, proof อย่างเดียว, verify, update และแยกฝั่ง user กับฝั่ง server

ตัวจับเวลาเดิมใน `amtp/server.py` (`build_response`) ใช้ไม่ได้สำหรับเรื่องนี้ เพราะช่วง "lookup" ครอบงานเตรียม proof ไว้ด้วย (แผน AND pruning, IMT path, รูป posting list ที่จะส่ง) ตัวเลขนั้นจึงไม่ใช่เวลาค้นหาจริง

## ข้างในมีอะไร

| ไฟล์ | หน้าที่ |
|---|---|
| `search_only.py` | การค้นหาของ server อย่างเดียว ไม่สร้าง proof มี 3 แบบ (ดูด้านล่าง) และ `SearchIndex` |
| `run.py` | รัน suite เดิมของ `bench.experiments` (search1, xsearch, xdomains ฯลฯ) โดยห่อ `run_query` ให้วัด search แยกเพิ่ม · argument, flag, ข้อมูล และ query เหมือนเดิมทุกอย่าง |
| `plot_search.py` | กราฟเส้น "Server search time, without proof" เทียบกับ baseline · ใช้ฟังก์ชันวาดกราฟของ `baselines/plot_paper.py` (ไม่แก้ไฟล์นั้น) |
| `run_cloud.sh` | รันบน cloud 1 เครื่อง: pytest → search1 → xsearch → xdomains → กราฟ (~3.5 ชม.) |
| `test_search_split.py` | ทั้ง 3 แบบต้องได้ไฟล์ตรงกับ response จริง (AND, OR, query ซ้อน, keyword ที่ไม่มี) ทั้งตอนเปิดและปิด flag |

## การค้นหา 3 แบบ

| แบบ | คอลัมน์ | ขั้นตอน |
|---|---|---|
| เต็ม (plain) | `server_search`, `server_search_fetch` | lookup (Cuckoo + B+ tree) → สร้างเซตไฟล์จาก posting list ทั้งหมด → AND / OR → ดึง ciphertext |
| partition ก่อน | `server_search_pf`, `server_search_pf_fetch` | lookup → หา partition ที่ match ได้ (`boolean.focus`) → สร้างเซตเฉพาะใน partition เหล่านั้น → AND / OR → ดึง |
| **fast (มีดัชนี)** | `server_search_fast`, `server_search_fast_fetch` | ใช้ `SearchIndex` ที่สร้างครั้งเดียวต่อ snapshot (ตอน setup) → หา partition ก่อนโดยไม่ copy เซต → ดึงไฟล์เฉพาะ partition เหล่านั้น |

- **`SearchIndex`:** สำหรับ keyword ที่อยู่ตั้งแต่ `KEEP_MIN_PARTS` (= 3) partition เก็บ hex key → (เซตของ partition, ไฟล์ในแต่ละ partition) โดยอ้างถึง list เดิมของ snapshot ไม่ได้ copy ส่วน keyword ที่เล็กกว่านั้นยังค้นผ่าน Cuckoo filter + B+ tree
- **ต้นทุนของดัชนี:** ทุกรอบบันทึกเวลาสร้างและขนาดไว้ใน `search_index_<suite>.json` (D1, N = 100k: 83k keyword, 0.43 วินาที, ~89 MB)
- **proof ไม่เปลี่ยน:** Cuckoo filter, B+ tree และ IMT ยังใช้สร้าง proof เหมือนเดิม (`amtp/server.py`)
- **`_fetch`:** รวมการดึง ciphertext เพื่อให้นับงานเท่ากับ `search_core` ของ SE-COLLAB ซึ่งรวม result fetch ไว้

**จุดตรวจความถูกต้อง:** ทุก query ที่วัด ทั้ง 3 แบบต้องได้ไฟล์ชุดเดียวกับ response จริง ไม่อย่างนั้นการรันจะหยุด

## เทียบกับ baseline (นิยาม "search อย่างเดียว")

| scheme | คอลัมน์ | หมายเหตุ |
|---|---|---|
| AMTP | `server_search_fetch` / `server_search_fast_fetch` | ตามตารางด้านบน |
| SE-COLLAB | `search_core` | BPL retrieval + Boolean + result fetch (SE-COLLAB แยกไว้เองอยู่แล้ว) |
| Chen | `server_total` | server ของ Chen ทำแค่การค้น · วัดหลัง warm-up ซึ่ง chain ถูกยุบแล้ว (กรณีที่ดีที่สุดของ Chen) และ Chen ไม่ต้องดึงไฟล์ |
| Du | `server_total` | Bloom-filter tree บนแต่ละ cloud นับ cloud ที่ช้าที่สุด |

baseline ไม่ได้รันใหม่ ใช้ผลบน cloud จาก `baselines/results/new_amtp_result/gamma_with_v/`

## วิธีรัน

```
cd amtp_code
../.venv/bin/python -m pytest -q bench_split                                   # test
SMOKE=1 RUN_NAME=smoke_split bash bench_split/run_cloud.sh                      # ตรวจทุกขั้น 1 รอบต่อจุด
RUN_NAME=paper_split bash bench_split/run_cloud.sh                              # จริง (cloud) reps 15
../.venv/bin/python -m bench_split.plot_search --amtp <run>/amtp --out <dir>    # กราฟอย่างเดียว
```

ผลของรอบจริงเก็บไว้ที่ `baselines/results/search_split/` (`data/` และ `graphs/`) แยกจากผลเดิม
