-- ═══════════════════════════════════════════════════════════════════════
-- EXPLAIN ANALYZE ni chuqur o'qish
-- ═══════════════════════════════════════════════════════════════════════

DROP TABLE IF EXISTS tolovlar;

CREATE TABLE tolovlar (
    id       BIGSERIAL     PRIMARY KEY,
    mijoz_id INTEGER       NOT NULL,
    sana     DATE          NOT NULL,
    holat    VARCHAR(20)   NOT NULL,
    summa    NUMERIC(12,2) NOT NULL
);

-- 200 000 qator. Kichik jadvalda rejalar ishonchsiz: 100 qatorlik
-- jadvalni to'liq skanerlash HAR DOIM arzon, shuning uchun indeks
-- foydasi umuman ko'rinmaydi.
INSERT INTO tolovlar (mijoz_id, sana, holat, summa)
SELECT
    (random() * 5000)::INT + 1,
    DATE '2023-01-01' + (random() * 700)::INT,
    (ARRAY['yangi','tolangan','bekor','qaytarilgan'])[(random() * 3)::INT + 1],
    (random() * 900000 + 10000)::NUMERIC(12,2)
FROM generate_series(1, 200000);

-- ANALYZE statistikani yangilaydi. Busiz planner jadval haqida deyarli
-- hech narsa bilmaydi va butunlay noto'g'ri reja tanlashi mumkin.
ANALYZE tolovlar;

-- ─────────────────────────────────────────────────────────────────────
-- 1) EXPLAIN — bajarmaydi, faqat TAXMIN qiladi
-- ─────────────────────────────────────────────────────────────────────
EXPLAIN SELECT * FROM tolovlar WHERE holat = 'bekor';
--  Seq Scan on tolovlar  (cost=0.00..4073.00 rows=66853 width=31)
--                              ^^^^^^^^^^^^  ^^^^^^^^^  ^^^^^^^^
--                              |             |          bitta qator ~31 bayt
--                              |             planner 66853 qator kutmoqda
--                              birinchi..oxirgi qator narxi

-- ─────────────────────────────────────────────────────────────────────
-- 2) Seq Scan — qatorlarning katta qismi kerak bo'lganda
-- ─────────────────────────────────────────────────────────────────────
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM tolovlar WHERE summa > 20000;
--  Seq Scan ... rows=197700 ... (actual time=0.005..16.116 rows=197787 loops=1)
--    Rows Removed by Filter: 2213
--    Buffers: shared hit=1573
--  Taxmin 197700, haqiqat 197787 — planner deyarli aniq. Yaxshi belgi.

-- ─────────────────────────────────────────────────────────────────────
-- 3) Indeks BOR, lekin baribir Seq Scan — va bu TO'G'RI qaror
-- ─────────────────────────────────────────────────────────────────────
CREATE INDEX idx_tolovlar_holat ON tolovlar(holat);
CREATE INDEX idx_tolovlar_mijoz ON tolovlar(mijoz_id);
ANALYZE tolovlar;

-- 'bekor' EMAS -> qatorlarning ~75% i. Indeks orqali ularni bittalab
-- olish, butun jadvalni ketma-ket o'qishdan QIMMATROQ.
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM tolovlar WHERE holat <> 'bekor';
--  Seq Scan on tolovlar ... (actual time=0.006..13.427 rows=133440 loops=1)
--  Indeks bor, lekin ishlatilmadi. Planner haq.

-- ─────────────────────────────────────────────────────────────────────
-- 4) Bitmap Heap Scan — "o'rta" holat (~25%)
--    Ikki bosqich: Bitmap Index Scan sahifalar xaritasini yig'adi,
--    Bitmap Heap Scan esa ularni DISK TARTIBIDA o'qiydi.
-- ─────────────────────────────────────────────────────────────────────
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM tolovlar WHERE holat = 'bekor';
--  Bitmap Heap Scan on tolovlar  (cost=740.16..3138.75 rows=66047 ...)
--    Recheck Cond: ((holat)::text = 'bekor'::text)
--    Heap Blocks: exact=1573
--    ->  Bitmap Index Scan on idx_tolovlar_holat (actual ... rows=66560 ...)
--  cost 3138 < Seq Scan ning 4073 si — shuning uchun bitmap tanlandi.
--  "lossy=" paydo bo'lsa: bitmap work_mem ga sig'magan.

-- ─────────────────────────────────────────────────────────────────────
-- 5) Index Scan — unikal qidiruv
-- ─────────────────────────────────────────────────────────────────────
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM tolovlar WHERE id = 123456;
--  Index Scan using tolovlar_pkey ... (actual time=0.015..0.015 rows=1 loops=1)
--    Buffers: shared hit=7        <-- atigi 7 sahifa
--  Execution Time: 0.025 ms

-- ORDER BY + LIMIT ham Index Scan ni "chaqiradi": indeks allaqachon
-- tartiblangan, shuning uchun 20 ta qator olib to'xtash mumkin.
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM tolovlar WHERE mijoz_id BETWEEN 100 AND 300 ORDER BY mijoz_id LIMIT 20;

-- ─────────────────────────────────────────────────────────────────────
-- 6) KUTILMAGAN NATIJA: 200 000 dan atigi 38 qator, lekin baribir Bitmap
-- ─────────────────────────────────────────────────────────────────────
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM tolovlar WHERE mijoz_id = 777;
--  Bitmap Heap Scan ... (actual time=0.028..0.074 rows=38 loops=1)
--    Heap Blocks: exact=38
--  38 qator 38 ta TURLI sahifada yotibdi (ma'lumot tasodifiy kiritilgan).
--  Ya'ni rejaga faqat qatorlar SONI emas, ularning JISMONIY joylashuvi
--  ham ta'sir qiladi.

-- ─────────────────────────────────────────────────────────────────────
-- 7) Index Only Scan — va nega u darhol ishlamaydi
-- ─────────────────────────────────────────────────────────────────────
CREATE INDEX idx_tolovlar_mijoz_summa ON tolovlar(mijoz_id, summa);
ANALYZE tolovlar;

-- Kerakli ikkala ustun ham indeksda bor. Lekin reja hali ham Bitmap:
EXPLAIN (ANALYZE, BUFFERS) SELECT mijoz_id, summa FROM tolovlar WHERE mijoz_id = 777;
--  Bitmap Heap Scan ... Buffers: shared hit=42
--  Sababi: indeks qatorning KO'RINUVCHANLIGINI bilmaydi. Buni visibility
--  map saqlaydi, uni esa VACUUM to'ldiradi. Ommaviy yuklashdan keyin u bo'sh.

-- DIQQAT: VACUUM ni tranzaksiya ichida bajarib BO'LMAYDI.
--   ERROR:  VACUUM cannot run inside a transaction block
-- Quyidagi qatorni alohida, avtomatik commit rejimida ishga tushiring:
VACUUM (ANALYZE) tolovlar;

EXPLAIN (ANALYZE, BUFFERS) SELECT mijoz_id, summa FROM tolovlar WHERE mijoz_id = 777;
--  Index Only Scan using idx_tolovlar_mijoz_summa ...
--    Heap Fetches: 0              <-- jadvalga UMUMAN murojaat qilinmadi
--    Buffers: shared hit=4        <-- 42 o'rniga 4 sahifa
--  Bu "covering index" ning butun ma'nosi.

-- ─────────────────────────────────────────────────────────────────────
-- 8) Planner ADASHGANDA: bog'liq ustunlar
-- ─────────────────────────────────────────────────────────────────────
DROP TABLE IF EXISTS manzillar;
CREATE TABLE manzillar (
    id      BIGSERIAL     PRIMARY KEY,
    shahar  VARCHAR(30)   NOT NULL,
    viloyat VARCHAR(30)   NOT NULL,
    summa   NUMERIC(10,2) NOT NULL
);

-- shahar viloyatni TO'LIQ aniqlaydi — funksional bog'liqlik bor
INSERT INTO manzillar (shahar, viloyat, summa)
SELECT s.shahar, s.viloyat, (random() * 100000)::NUMERIC(10,2)
FROM generate_series(1, 200000) g
CROSS JOIN LATERAL (
    SELECT * FROM (VALUES
        ('Nurafshon','Toshkent'), ('Chirchiq','Toshkent'), ('Angren','Toshkent'),
        ('Urgut','Samarqand'),    ('Kattaqorgon','Samarqand'),
        ('Gijduvon','Buxoro'),    ('Kogon','Buxoro'),
        ('Margilon','Fargona'),   ('Qoqon','Fargona'), ('Quva','Fargona')
    ) AS v(shahar, viloyat) OFFSET (g % 10) LIMIT 1
) s;
ANALYZE manzillar;

EXPLAIN (ANALYZE, TIMING OFF)
SELECT * FROM manzillar WHERE shahar = 'Margilon' AND viloyat = 'Fargona';
--  Seq Scan ... rows=6154 ... (actual rows=20000 loops=1)
--               ^^^^^^^^^              ^^^^^^^^^^
--               taxmin                 haqiqat — 3 baravar farq!
--  Sabab: planner ikki ustunni MUSTAQIL deb hisoblab, tanlanuvchanliklarni
--  ko'paytirdi. Aslida shahar viloyatni to'liq aniqlaydi.

-- Yechim — ko'p ustunli kengaytirilgan statistika:
CREATE STATISTICS st_manzil (dependencies) ON shahar, viloyat FROM manzillar;
ANALYZE manzillar;

EXPLAIN (ANALYZE, TIMING OFF)
SELECT * FROM manzillar WHERE shahar = 'Margilon' AND viloyat = 'Fargona';
--  Seq Scan ... rows=19293 ... (actual rows=20000 loops=1)
--  Endi taxmin haqiqatga juda yaqin. Katta so'rovda bu noto'g'ri JOIN
--  turini tanlashning oldini oladi.

-- ─────────────────────────────────────────────────────────────────────
-- 9) JOIN turlari bitta rejada
-- ─────────────────────────────────────────────────────────────────────
DROP TABLE IF EXISTS mijozlar;
CREATE TABLE mijozlar (id SERIAL PRIMARY KEY, ism VARCHAR(60) NOT NULL);
INSERT INTO mijozlar (ism) SELECT 'Mijoz ' || g FROM generate_series(1, 5000) g;
ANALYZE mijozlar;

-- Kam qator -> Nested Loop: tashqi tomondagi har qator uchun ichki
-- tomonda indeks bo'yicha qidiruv
EXPLAIN (ANALYZE)
SELECT m.ism, t.summa
FROM mijozlar m JOIN tolovlar t ON t.mijoz_id = m.id
WHERE m.id = 777;

-- Ko'p qator -> Hash Join: kichik jadvaldan xesh quriladi, katta jadval
-- bir marta o'tib chiqiladi
EXPLAIN (ANALYZE)
SELECT m.ism, SUM(t.summa)
FROM mijozlar m JOIN tolovlar t ON t.mijoz_id = m.id
GROUP BY m.ism;
