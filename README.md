-- ═══════════════════════════════════════════════════════════════════════
-- Indeks turlari: B-tree, kompozit, qisman, GIN, covering, BRIN
-- ═══════════════════════════════════════════════════════════════════════

DROP TABLE IF EXISTS maqolalar;

CREATE TABLE maqolalar (
    id         BIGSERIAL   PRIMARY KEY,
    muallif_id INTEGER     NOT NULL,
    sarlavha   TEXT        NOT NULL,
    matn       TEXT        NOT NULL,
    teglar     TEXT[]      NOT NULL DEFAULT '{}',
    meta       JSONB       NOT NULL DEFAULT '{}',
    holat      VARCHAR(20) NOT NULL,
    sana       TIMESTAMPTZ NOT NULL
);

INSERT INTO maqolalar (muallif_id, sarlavha, matn, teglar, meta, holat, sana)
SELECT
    (random() * 500)::INT + 1,
    'Maqola ' || g,
    'PostgreSQL indekslari haqida matn ' || g
      -- qatorlarning ~0.5% iga kamyob so'z qo'shamiz: full-text testi uchun
      || CASE WHEN random() < 0.005 THEN ' noyobatama' ELSE '' END,
    CASE (random() * 3)::INT
        WHEN 0 THEN ARRAY['sql','backend']
        WHEN 1 THEN ARRAY['python']
        WHEN 2 THEN ARRAY['sql','performance']
        ELSE        ARRAY['devops']
    END,
    jsonb_build_object(
        'til',  (ARRAY['uz','ru','en'])[(random() * 2)::INT + 1],
        'reja', CASE WHEN random() < 0.01 THEN 'pro' ELSE 'free' END,
        'ko_rishlar', (random() * 1000)::INT
    ),
    CASE WHEN random() < 0.02 THEN 'qoralama' ELSE 'chop_etilgan' END,
    NOW() - (random() * 700 || ' days')::INTERVAL
FROM generate_series(1, 200000) g;
ANALYZE maqolalar;

-- ─────────────────────────────────────────────────────────────────────
-- 1) KOMPOZIT INDEKS — ustunlar tartibi hal qiluvchi
-- ─────────────────────────────────────────────────────────────────────
CREATE INDEX idx_m_muallif_sana ON maqolalar(muallif_id, sana);
ANALYZE maqolalar;

-- (a) Ikkala ustun ham shartda -> indeks to'liq ishlaydi
EXPLAIN (ANALYZE, TIMING OFF)
SELECT * FROM maqolalar WHERE muallif_id = 42 AND sana > NOW() - INTERVAL '30 days';
--  ->  Bitmap Index Scan on idx_m_muallif_sana  (cost=0.00..4.59 ...)

-- (b) Faqat BIRINCHI ustun (chapdan prefiks) -> ishlaydi
EXPLAIN (ANALYZE, TIMING OFF)
SELECT * FROM maqolalar WHERE muallif_id = 42;
--  ->  Bitmap Index Scan on idx_m_muallif_sana  (cost=0.00..11.40 ...)

-- (c) Faqat IKKINCHI ustun -> indeks BOSHDAN-OXIR skanerlanadi
EXPLAIN (ANALYZE, TIMING OFF)
SELECT * FROM maqolalar WHERE sana > NOW() - INTERVAL '3 days';
--  ->  Bitmap Index Scan on idx_m_muallif_sana  (cost=0.00..4592.43 ...)
--                                                        ^^^^^^^
--  Narxni (a) dagi 4.59 bilan solishtiring — 1000 baravar farq.
--  Rejada "Index Scan" so'zi bo'lishi hali hammasi joyida degani emas.

-- ─────────────────────────────────────────────────────────────────────
-- 2) QISMAN (PARTIAL) INDEKS — faqat kerakli qatorlar uchun
-- ─────────────────────────────────────────────────────────────────────
CREATE INDEX idx_m_qoralama ON maqolalar(muallif_id) WHERE holat = 'qoralama';
ANALYZE maqolalar;

EXPLAIN (ANALYZE, TIMING OFF, BUFFERS)
SELECT * FROM maqolalar WHERE holat = 'qoralama' AND muallif_id = 42;
--  ->  Bitmap Index Scan on idx_m_qoralama  (cost=0.00..4.34 ...)

-- Hajmlarni solishtiramiz:
SELECT pg_size_pretty(pg_relation_size('idx_m_muallif_sana')) AS toliq_indeks,
       pg_size_pretty(pg_relation_size('idx_m_qoralama'))     AS qisman_indeks;
--  toliq_indeks | qisman_indeks
--  6184 kB      | 56 kB          <-- 110 baravar kichik

-- ─────────────────────────────────────────────────────────────────────
-- 3) GIN — massiv ichidagi qiymat bo'yicha qidiruv
-- ─────────────────────────────────────────────────────────────────────
CREATE INDEX idx_m_teglar ON maqolalar USING GIN (teglar);
ANALYZE maqolalar;

EXPLAIN (ANALYZE, TIMING OFF)
SELECT id, sarlavha FROM maqolalar WHERE teglar @> ARRAY['devops'];
--  ->  Bitmap Index Scan on idx_m_teglar ... rows=33518

-- ─────────────────────────────────────────────────────────────────────
-- 4) GIN — jsonb. jsonb_path_ops kichikroq va @> uchun tezroq.
-- ─────────────────────────────────────────────────────────────────────
CREATE INDEX idx_m_meta ON maqolalar USING GIN (meta jsonb_path_ops);
ANALYZE maqolalar;

-- TANLANUVCHAN shart (~1% qator) -> indeks ishlaydi
EXPLAIN (ANALYZE, TIMING OFF)
SELECT id FROM maqolalar WHERE meta @> '{"reja":"pro"}';
--  ->  Bitmap Index Scan on idx_m_meta ... rows=1964,  Execution Time: 1.6 ms

-- Agar shart HAMMA qatorga mos kelsa, planner Seq Scan tanlaydi va HAQ
-- bo'ladi — indeks bor bo'lishi uni ishlatish shart degani emas.

-- ─────────────────────────────────────────────────────────────────────
-- 5) GIN — full-text qidiruv. Eng katta farq shu yerda ko'rinadi.
-- ─────────────────────────────────────────────────────────────────────
-- Avval INDEKSSIZ o'lchaymiz:
EXPLAIN (ANALYZE, TIMING OFF)
SELECT id FROM maqolalar
WHERE to_tsvector('simple', sarlavha || ' ' || matn) @@ to_tsquery('simple', 'noyobatama');
--  Parallel Seq Scan ...  Execution Time: 177.7 ms

CREATE INDEX idx_m_fts ON maqolalar
    USING GIN (to_tsvector('simple', sarlavha || ' ' || matn));
ANALYZE maqolalar;

EXPLAIN (ANALYZE, TIMING OFF)
SELECT id, sarlavha FROM maqolalar
WHERE to_tsvector('simple', sarlavha || ' ' || matn) @@ to_tsquery('simple', 'noyobatama');
--  Bitmap Heap Scan ...  Execution Time: 0.86 ms    <-- ~200 baravar tez
-- DIQQAT: indeksdagi ifoda va WHERE dagi ifoda AYNAN bir xil bo'lishi shart.

-- ─────────────────────────────────────────────────────────────────────
-- 6) COVERING indeks (INCLUDE) — qidiruvda qatnashmaydigan ustunni
--    indeksga "yo'lovchi" sifatida qo'shish
-- ─────────────────────────────────────────────────────────────────────
CREATE INDEX idx_m_cover ON maqolalar(muallif_id) INCLUDE (sarlavha);
ANALYZE maqolalar;

EXPLAIN (ANALYZE, TIMING OFF, BUFFERS)
SELECT muallif_id, sarlavha FROM maqolalar WHERE muallif_id = 42;
-- Eslatma: Index Only Scan olish uchun VACUUM ham kerak (4-darsga qarang).
-- VACUUM siz reja Bitmap Heap Scan bo'lib qolaveradi.

-- ─────────────────────────────────────────────────────────────────────
-- 7) IFODA (expression) indeksi
-- ─────────────────────────────────────────────────────────────────────
CREATE INDEX idx_m_lower ON maqolalar(lower(sarlavha));
ANALYZE maqolalar;

EXPLAIN (ANALYZE, TIMING OFF)
SELECT id FROM maqolalar WHERE lower(sarlavha) = 'maqola 999';
--  Index Scan using idx_m_lower ... (actual rows=1 loops=1)
--  Oddiy maqolalar(sarlavha) indeksi bu yerda ISHLAMAS edi.

-- ─────────────────────────────────────────────────────────────────────
-- 8) BRIN — juda katta, jismonan tartiblangan jadvallar uchun
-- ─────────────────────────────────────────────────────────────────────
CREATE INDEX idx_m_brin ON maqolalar USING BRIN (sana);
ANALYZE maqolalar;

SELECT pg_size_pretty(pg_relation_size('idx_m_brin'))          AS brin_hajmi,
       pg_size_pretty(pg_relation_size('idx_m_muallif_sana'))  AS btree_hajmi;
--  brin_hajmi | btree_hajmi
--  24 kB      | 6184 kB        <-- 250 baravar kichik
-- Lekin BRIN faqat ma'lumot diskda tartiblangan bo'lsa foydali
-- (masalan, faqat qo'shiladigan log jadvali).

-- ─────────────────────────────────────────────────────────────────────
-- 9) Produksiyada: ishlatilmayotgan indekslarni topish
--    (idx_scan real foydalanish statistikasi asosida to'ladi, shuning
--     uchun bu so'rov ishlab turgan bazada ma'noga ega)
-- ─────────────────────────────────────────────────────────────────────
SELECT s.relname AS jadval, s.indexrelname AS indeks, s.idx_scan,
       pg_size_pretty(pg_relation_size(s.indexrelid)) AS hajm
FROM pg_stat_user_indexes s
JOIN pg_index i ON i.indexrelid = s.indexrelid
WHERE s.idx_scan = 0 AND NOT i.indisunique AND NOT i.indisprimary
ORDER BY pg_relation_size(s.indexrelid) DESC;
