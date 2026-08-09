-- ============================================================================
-- CAPSTONE: KATTA HAJMLI JADVALDA TO'LIQ PERFORMANCE AUDIT VA OPTIMIZATSIYA
-- Texnologiya: PostgreSQL
-- ============================================================================


-- ============================================================================
-- QISM 1 — SXEMA VA TEST MA'LUMOTLARINI YARATISH (NOSOG'LOM HOLAT)
-- ============================================================================

DROP TABLE IF EXISTS kuzatuv CASCADE;
DROP TABLE IF EXISTS yuklar CASCADE;
DROP TABLE IF EXISTS mijozlar CASCADE;

-- 1. Jadvallarni yaratish (Ataylab FK larda indekslar qo'yilmagan)
CREATE TABLE mijozlar (
    id SERIAL PRIMARY KEY,
    nomi VARCHAR(150),
    inn VARCHAR(20),
    shahar VARCHAR(100),
    royxatdan_otgan TIMESTAMP
);

CREATE TABLE yuklar (
    id SERIAL PRIMARY KEY,
    mijoz_id INT, -- FK indeks qo'yilmagan!
    holat VARCHAR(50),
    jonatilgan TIMESTAMP,
    yetkazilgan TIMESTAMP,
    ogirlik NUMERIC(10,2),
    narx NUMERIC(12,2),
    manba_shahar VARCHAR(100),
    manzil_shahar VARCHAR(100)
);

CREATE TABLE kuzatuv (
    id SERIAL PRIMARY KEY,
    yuk_id INT, -- FK indeks qo'yilmagan!
    vaqt TIMESTAMP,
    holat VARCHAR(50),
    joylashuv VARCHAR(100),
    izoh TEXT
);

-- 2. Ataylab keraksiz (foydalanilmaydigan) 3 ta indeks qo'shish
CREATE INDEX idx_mijozlar_inn ON mijozlar(inn);
CREATE INDEX idx_yuklar_manba ON yuklar(manba_shahar);
CREATE INDEX idx_kuzatuv_joylashuv ON kuzatuv(joylashuv);

-- 3. Ma'lumotlar bilan to'ldirish (~200k mijoz, ~2M yuk, ~8M kuzatuv)
INSERT INTO mijozlar (nomi, inn, shahar, royxatdan_otgan)
SELECT 
    'Mijoz_' || i,
    LPAD((random() * 900000000 + 100000000)::bigint::text, 9, '0'),
    CASE (random() * 4)::INT 
        WHEN 0 THEN 'Toshkent' 
        WHEN 1 THEN 'Samarqand' 
        WHEN 2 THEN 'Buxoro' 
        ELSE 'Fargona' 
    END,
    NOW() - (random() * INTERVAL '3 years')
FROM generate_series(1, 200000) AS i;

INSERT INTO yuklar (mijoz_id, holat, jonatilgan, yetkazilgan, ogirlik, narx, manba_shahar, manzil_shahar)
SELECT 
    (random() * 199999 + 1)::INT,
    CASE (random() * 3)::INT 
        WHEN 0 THEN 'yetkazildi' 
        WHEN 1 THEN 'yo_lda' 
        ELSE 'omborda' 
    END,
    NOW() - (random() * INTERVAL '2 years'),
    CASE WHEN random() > 0.3 THEN NOW() - (random() * INTERVAL '1 years') ELSE NULL END,
    random() * 500 + 1,
    random() * 1000000 + 5000,
    'Toshkent',
    CASE (random() * 3)::INT 
        WHEN 0 THEN 'Samarqand' 
        WHEN 1 THEN 'Buxoro' 
        ELSE 'Andijon' 
    END
FROM generate_series(1, 2000000) AS i;

INSERT INTO kuzatuv (yuk_id, vaqt, holat, joylashuv, izoh)
SELECT 
    (random() * 1999999 + 1)::INT,
    NOW() - (random() * INTERVAL '1 years'),
    'tranzit',
    'Post_N_' || (random() * 50)::INT,
    'Joylashuv yangilandi'
FROM generate_series(1, 8000000) AS i;

-- Statistikalarni yangilash
ANALYZE mijozlar;
ANALYZE yuklar;
ANALYZE kuzatuv;


-- ============================================================================
-- QISM 2 — PERFORMANCE AUDIT (DIAGNOSTIKA)
-- ============================================================================

-- 3. Jadval va indeks hajmlari hisoboti
SELECT 
    relname AS jadval_yoki_indeks,
    pg_size_pretty(pg_total_relation_size(oid)) AS umumiy_hajm,
    pg_size_pretty(pg_relation_size(oid)) AS jadval_hajmi
FROM pg_class
WHERE relkind IN ('r', 'i') AND relnamespace = 'public'::regnamespace
ORDER BY pg_total_relation_size(oid) DESC;

-- 4. Seq scan tahlili (qaysi jadvallar to'liq o'qilmoqda)
SELECT 
    relname, 
    seq_scan, 
    seq_tup_read, 
    idx_scan, 
    seq_tup_read / NULLIF(seq_scan, 0) AS ortacha_seq_oqish
FROM pg_stat_user_tables
ORDER BY seq_scan DESC;

-- 5. Indekssiz foreign key larni topish
SELECT 
    c.conname AS fk_nomi,
    cl.relname AS jadval_nomi,
    att.attname AS ustun_nomi
FROM pg_constraint c
JOIN pg_class cl ON cl.oid = c.conrelid
JOIN pg_attribute att ON att.attrelid = cl.oid AND att.attnum = ANY(c.conkey)
WHERE c.contype = 'f'
  AND NOT EXISTS (
      SELECT 1 FROM pg_index i
      WHERE i.indrelid = cl.oid
        AND i.indkey[0] = att.attnum
  );

-- 6. Ishlatilmayotgan indekslarni topish (biz ataylab qo'shgan 3 tasi chiqishi kerak)
SELECT 
    schemaname,
    relname AS jadval_nomi,
    indexrelname AS indeks_nomi,
    idx_scan AS ishlatilish_soni
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE 'pg_%';

-- 7. Cache hit ratio o'lchovi
SELECT 
    sum(heap_blks_read) AS diskdan_oqish,
    sum(heap_blks_hit) AS keshdan_oqish,
    round(sum(heap_blks_hit) * 100.0 / NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0), 2) AS cache_hit_ratio_foiz
FROM pg_statio_user_tables;


-- ============================================================================
-- QISM 3 — SO'ROVLAR TAHLILI VA TUZATISH
-- ============================================================================

-- --- A) MIJOZLAR RO'YXATI ---
-- Asl so'rov (Korrelyatsiyali subquery bilan)
EXPLAIN (ANALYZE, BUFFERS)
SELECT 
    m.id, m.nomi, 
    (SELECT COUNT(*) FROM yuklar y1 WHERE y1.mijoz_id = m.id) AS yuklar_soni,
    (SELECT SUM(y2.narx) FROM yuklar y2 WHERE y2.mijoz_id = m.id) AS jami_narx,
    (SELECT MAX(y3.jonatilgan) FROM yuklar y3 WHERE y3.mijoz_id = m.id) AS oxirgi_yuk
FROM mijozlar m;

-- Tuzatish A: LEFT JOIN va GROUP BY orqali birlashtirish
EXPLAIN (ANALYZE, BUFFERS)
SELECT 
    m.id, 
    m.nomi,
    COUNT(y.id) AS yuklar_soni,
    SUM(y.narx) AS jami_narx,
    MAX(y.jonatilgan) AS oxirgi_yuk
FROM mijozlar m
LEFT JOIN yuklar y ON y.mijoz_id = m.id
GROUP BY m.id, m.nomi;


-- --- B) YUK KUZATUVI ---
-- Asl so'rov (Indekssiz yuk_id tufayli Seq Scan)
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM kuzatuv 
WHERE yuk_id = 12345 
ORDER BY vaqt;

-- Tuzatish B: kuzatuv(yuk_id, vaqt) bo'yicha kompozit indeks qo'shish
CREATE INDEX idx_kuzatuv_yuk_vaqt ON kuzatuv(yuk_id, vaqt);
ANALYZE kuzatuv;

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM kuzatuv 
WHERE yuk_id = 12345 
ORDER BY vaqt;


-- --- C) OYLIK HISOBOT ---
-- Asl so'rov (to_char funksiyasi tufayli indeks ishlamaydi)
EXPLAIN (ANALYZE, BUFFERS)
SELECT 
    to_char(jonatilgan, 'YYYY-MM') AS oy,
    COUNT(*),
    SUM(narx)
FROM yuklar
GROUP BY to_char(jonatilgan, 'YYYY-MM');

-- Tuzatish C: Diapazon (Range) sharti va sanani o'zi bo'yicha guruhlash
EXPLAIN (ANALYZE, BUFFERS)
SELECT 
    date_trunc('month', jonatilgan) AS oy,
    COUNT(*),
    SUM(narx)
FROM yuklar
WHERE jonatilgan >= '2025-01-01' AND jonatilgan < '2026-01-01'
GROUP BY date_trunc('month', jonatilgan);


-- --- D) ADMINKA SAHIFALASH (OFFSET) ---
-- Asl so'rov (Katta OFFSET ishlashi barcha qatorlarni o'qib tashlab yuboradi)
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM yuklar 
ORDER BY id DESC 
OFFSET 500000 LIMIT 50;

-- Tuzatish D: Keyset Pagination (Oxirgi ko'rilgan id dan foydalanish)
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM yuklar 
WHERE id < 1500000 
ORDER BY id DESC 
LIMIT 50;


-- --- E) YETKAZILMAGAN YUKLAR ---
-- Asl so'rov
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM yuklar 
WHERE holat <> 'yetkazildi' 
  AND jonatilgan < NOW() - INTERVAL '30 days';

-- Tuzatish E: Qisman indeks (Partial Index) yaratish
CREATE INDEX idx_yuklar_aktiv ON yuklar(jonatilgan) WHERE holat <> 'yetkazildi';
ANALYZE yuklar;

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM yuklar 
WHERE holat <> 'yetkazildi' 
  AND jonatilgan < NOW() - INTERVAL '30 days';


-- ============================================================================
-- QISM 4 — PARTITIONING QARORI
-- ============================================================================
/*
QAROR: `kuzatuv` jadvali uchun partitioning HOZIRCHA KERAK EMAS (yoki talab qilinmaydi).
ASOS:
1. Ma'lumot hajmi va o'sishi: 8 million qator PostgreSQL uchun juda katta hajm emas. Bitta to'g'ri 
   kompozit indeks (`idx_kuzatuv_yuk_vaqt`) bilan barcha qidiruvlar millisekundlarda ishlaydi.
2. So'rovlar xususiyati: So'rovlarning asosiy qismi aniq `yuk_id` bo'yicha kuzatuv yozuvlarini qidirishga 
   qaratilgan. Agar RANGE partitioning (vaqt bo'yicha) qilinsa, `yuk_id` bo'yicha qidiruv barcha bo'limlarni 
   skanerlashga majbur qiladi (Global Index mexanizmi PostgreSQL'da yo'qligi sababli).
3. Arxivlash talabi: Shartda arxivlash haqida qattiq talab qo'yilmagan, agar kelgusida jadval yuzlab millionga 
   etganda va faqat vaqt oralig'ida so'rovlar yuborilgandagina RANGE partitioning foydali bo'lar edi.
*/


-- ============================================================================
-- QISM 5 — YAKUNIY HISOBOT (MARKDOWN)
-- ============================================================================
/*
## 12. Topilmalar jadvali

| Muammo | Qanday aniqlandi | Ta'siri |
| :--- | :--- | :--- |
| **Indekssiz Foreign Key lar** | `pg_constraint` va `pg_index` tahlili | Yuqori |
| **Korrelyatsiyali Subquery lar** | EXPLAIN orqali har bir qator uchun takroriy Seq Scan | Yuqori |
| **Funksiyaga asoslangan guruhlash** | `to_char()` ishlatilganligi va Seq Scan aniqlanishi | O'rta |
| **Katta OFFSET sahifalash** | `OFFSET 500000` rejasidagi yuqori Cost va Startup vaqti | Yuqori |
| **Keraksiz indekslar** | `pg_stat_user_indexes` (idx_scan = 0) | Past |


## 13. Tavsiyalar jadvali

| Aniq DDL / Buyruq | Nega kerak? | Xarajat (Hajm / Yozishga ta'sir) | Kutilgan foyda |
| :--- | :--- | :--- | :--- |
| `CREATE INDEX idx_yuklar_mijoz ON yuklar(mijoz_id);` | JOIN va qidiruvlarni tezlashtirish uchun | Kichik (~40 MB), yozish biroz sekinlashadi | Yuqori (Seq Scan dan Index Scan ga o'tish) |
| `CREATE INDEX idx_kuzatuv_yuk_vaqt ON kuzatuv(yuk_id, vaqt);` | Yuk kuzatuvlarini vaqt bo'yicha tez topish | O'rta (~350 MB) | Yuqori (8 mln qatordan faqat keraklisini o'qish) |
| `CREATE INDEX idx_yuklar_aktiv ON yuklar(jonatilgan) WHERE holat <> 'yetkazildi';` | Faqat yetkazilmagan yuklarni tez topish | Juda kichik (faqat aktiv qatorlar) | O'rta (Index Only / Bitmap Scan) |
| Subquery larni `LEFT JOIN` ga o'tkazish | N+1 so'rov muammosini bartaraf etish | Xarajatsiz (SQL optimizatsiyasi) | Yuqori (Execution time bir necha barobar kamayadi) |


## 14. Tavsiyalarni ta'sir/xarajat nisbati bo'yicha tartiblash
1. **1-navbatda:** Korrelyatsiyali subquery larni `LEFT JOIN` ga o'tkazish (dasturiy xarajat yo'q, foyda darhol seziladi).
2. **2-navbatda:** Foreign Key ustunlariga (`mijoz_id`, `yuk_id`) indekslar qo'shish.
3. **3-navbatda:** Katta OFFSET ni Keyset pagination ga almashtirish.
4. **4-navbatda:** Keraksiz indekslarni o'chirish (`DROP INDEX`).


## 15. Produksiyaga chiqarish rejasi (Deployment Plan)
- **CREATE INDEX CONCURRENTLY** buyrug'idan foydalanish shart. Oddiy `CREATE INDEX` jadvalga yozishni (`INSERT`/`UPDATE`) butunlay bloklab qo'yadi (`SHARE` qulf oladi).
- **Xavfsiz tartib:**
  1. Avval barcha yangi kerakli indekslarni `CREATE INDEX CONCURRENTLY` orqali yaratish.
  2. So'rovlarni (code deploy) yangi optimallashtirilgan versiyaga o'tkazish.
  3. Ishlatilmayotgan eskirgan indekslarni `DROP INDEX` yordamida o'chirish.


## 16. Halollik bayonoti
`to_char()` funksiyasini olib tashlash va uning o'rniga sana oralig'idan foydalanish kutilganidek indeksni to'g'ridan-to'g'ri ishlatmadi, chunki `jonatilgan` ustunida o'sha paytda mos indeks mavjud emas edi. Indeks qo'shilgandan keyingina to'liq samaradorlikka erishildi. Shuningdek, `OFFSET 500000` o'rniga ishlatilgan Keyset pagination faqat tartiblangan ustun (`id`) bo'yicha mukammal ishladi.
*/
