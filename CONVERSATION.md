# บันทึกการสนทนา — Manga Reader TH (15 ต.ค. 2026)

## โปรเจค
- โฟลเดอร์: `/home/chuedekleb/manga-reader/` (ไฟล์เดียว `index.html`)
- GitHub: https://github.com/makzilla08/manga-reader
- ไลฟ์: https://makzilla08.github.io/manga-reader/ (Pages: legacy, branch main /root)
- เซิร์ฟเวอร์เทส local: `python3 -m http.server 8101 --directory ~/manga-reader` → http://127.0.0.1:8101/

## แอปคืออะไร
เว็บอ่านการ์ตูน 1 ไฟล์ (HTML+JS เพียว) บล็อกโฆษณา:
- ค้นหา/ยอดนิยม: AniList GraphQL (https://graphql.anilist.co, CORS เปิด, ไม่โดนบล็อก)
- หมวด: 🌏 รวม / 🇯🇵 JP / 🇰🇷 KR / 🇨🇳 CN (กรองด้วย countryOfOrigin) / 🇹🇭 เว็บไทย
- อ่านภาพ: MangaDex API ผ่าน Jina Reader proxy (`https://r.jina.ai/<api-url>` ตัด markdown header ทิ้งเอา JSON)
- ภาพมี fallback `images.weserv.nl` กัน hotlink (IMGERR/imgTag)
- fallback ปุ่มเว็บไทย Oremanga/NiceOppai/NekoPost + วาง URL ตอนดึงภาพ (loadThUrl2)

## ปัญหาที่เจอ + ทางแก้ (สำคัญ!)
1. MangaDex API **ไม่ส่ง CORS header** → browser ยิงตรงล้ม 100%
2. proxy สาธารณะตายหมด: corsproxy.io (ต้อง API key), allorigins/codetabs (522), cors.lol (429), consumet (ปิดบริการ), test.cors.workers.dev (429)
3. Google Translate proxy ได้ JSON แต่**ไม่ส่ง CORS** → ใช้ไม่ได้
4. **Jina Reader ส่ง `access-control-allow-origin` จริง** → ใช้เป็นตัวหลัก (โควต้า ~20 req/min)
5. Jina response มี markdown header → `j()` ตัดตั้งแต่ `{` ตัวแรกแล้ว JSON.parse
6. บางตอนไม่มีข้อมูลภาพ → guard ก่อน .map ทุกจุด

## ประวัติเวอร์ชัน
- v6: แตะปกอ่านทันที | v7: 7 proxy + weserv | v8: google translate proxy
- v9: jina proxy | v10: guard undefined maps
- v11 (15 ต.ค. 2026): กรองตอนติดลิขสิทธิ์ (externalUrl/pages=0) + read() ข้าม 5 ตอนอัตโนมัติ + ข้อความบอกติด LC → ใช้เว็บไทย — push แล้ว (78c1eb7)
- v12 (16 ต.ค. 2026): **ตัดซอร์สนอกหมด (AniList/MangaDex ออก)** → เว็บไทยล้วน: Oremanga (หลัก) + NekoPost + NiceOppai เลือกได้ — ดึงผ่าน Jina markdown: ค้น `?s=`, ยอดนิยมหน้าแรก, สารบัญจากลิงก์ `/<slug>/<slug>-N/`, ภาพตอนจาก `![Image](img.ต้นทาง)` — กรองโฆษณาพนัน (cdend/banner/gif) — ป้ายชาติจาก `/genre/manga|manhwa|manhua` — เทสจริง: หน้าแรก 41 ปก, ค้น solo 9 เรื่อง, เปิดตอนได้ 45 ภาพ — push แล้ว (78915bf)
  - หมายเหตุ: dark-manga + upmanga โดน Cloudflare bot-check (curl/Jina ผ่านไม่ได้) เลยไม่ได้ใช้; mangalc ต่อไม่ติด
  - กับดัก: regex literal `/!\[Image[^]]*\]/` รันใน browser ไม่ตรง (ทั้งที่ Python ตรง) → เขียน parseCards ใหม่ด้วย string ops (split/indexOf) ล้วน
  - backup: `index_v11_backup.html`

## เรื่องค้าง (อัปเดต 16 ต.ค.)
- ป้ายชาติหน้าแรกบางเรื่องไม่แม่น (การ์ด trending ไม่มี genre) — ควรดึง genre จากหน้ารายละเอียด หรือใช้หน้า manga-list
- NekoPost/NiceOppai ยังไม่ได้เทสโครง (มีแค่ dropdown) — เทสแล้วค่อยยืนยันว่าใช้ได้
- v12.2 (16 ต.ค. 2026): เหลือ Oremanga แหล่งเดียว (ตัด NekoPost/NiceOppai ออกจาก dropdown) — push แล้ว (7068c42)
  - สำรวจ dark-manga: curl ต่อติด (non-www) แต่ Jina โดน Cloudflare กันบอท → ใช้ในแอปไม่ได้ (browser ยิงตรงก็ติด CORS) — พักไว้
  - สำรวจ Manga-LC (ตัวจริง `manga-lc.net`, มี JP/KR/CN, โครง `/manga/<slug>/` + ตอน `/manga/<slug>/.../ตอนที่-N/`): ค้น/สารบัญผ่าน Jina ได้ แต่**ภาพตอนเป็น lazy `data-src` (mangaclaw.com)** Jina มองไม่เห็น → อ่านในแอปไม่ได้ ต้องมี proxy raw-HTML — พักไว้ก่อน
- v13 (ค้าง 16 ต.ค. 2026 — user เลือก "vercel + cloudflare worker"):
  - สร้าง `~/manga-proxy/` (worker.js + wrangler.jsonc) allowlist 7 โดเมน (oremanga/img/manga-lc/mangaclaw/dark) + CORS `*` — deploy แล้ว: `https://manga-proxy.hls-worker.workers.dev/?url=...`
  - เทสผ่าน worker: manga-lc ตอน (เจอ `data-src` mangaclaw ครบ), dark-manga ต้องใช้ `www` (non-www 404) — `?s=` search ของ dark โดน challenge บ้างเป็นครั้งคราว
  - โครง dark-manga (Ts theme mangareader): การ์ด `.bsx` (30) / `.listupd` (9), สารบัญ `.eplister` + `chapternum/ตอนที่ N`, genre `/genres/<slug>/`, ภาพตอนฝังใน `ts_reader.run({post_id, prevUrl, nextUrl, sources:[{images:[...]}]})`
  - **งานค้าง**: wire worker เข้า index.html (rawHtml via worker + parser dark/lc + dropdown แหล่ง 3 เจ้า) — ยังไม่ได้เริ่ม

## เรื่องค้าง
- user เน็ตต่อ MangaDx ไม่ได้โดยตรง ต้องพึ่ง Jina
- user เจอแคชเก่าบ่อย → ให้เปิดแท็บไม่ระบุตัวตน / Ctrl+Shift+R, เช็กป้ายเวอร์ชันที่หัวเว็บ
- error ล่าสุดที่ user รายงาน: "เปิดอ่านตรงไม่ได้" (ข้อความเก่า = ยังโหลดไฟล์เก่า)
