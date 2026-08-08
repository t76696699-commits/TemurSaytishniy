-- ═══════════════════════════════════════════════════════════════════════
-- N+1 muammosi: qanday paydo bo'ladi, qanday aniqlanadi, qanday tuzatiladi
-- ═══════════════════════════════════════════════════════════════════════

DROP TABLE IF EXISTS izohlar;
DROP TABLE IF EXISTS teglar_bogi;
DROP TABLE IF EXISTS postlar;

CREATE TABLE postlar (
    id       BIGSERIAL   PRIMARY KEY,
    muallif  VARCHAR(60) NOT NULL,
    sarlavha TEXT        NOT NULL,
    sana     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE TABLE izohlar (
    id      BIGSERIAL   PRIMARY KEY,
    post_id BIGINT      NOT NULL REFERENCES postlar(id) ON DELETE CASCADE,
    muallif VARCHAR(60) NOT NULL,
    matn    TEXT        NOT NULL,
    sana    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE TABLE teglar_bogi (
    post_id BIGINT      NOT NULL REFERENCES postlar(id) ON DELETE CASCADE,
    teg     VARCHAR(30) NOT NULL,
    PRIMARY KEY (post_id, teg)
);

INSERT INTO postlar (muallif, sarlavha)
SELECT 'Muallif ' || (g % 50), 'Post ' || g FROM generate_series(1, 5000) g;

INSERT INTO izohlar (post_id, muallif, matn)
SELECT (random() * 4999)::INT + 1, 'Izohchi ' || (g % 200), 'Izoh matni ' || g
FROM generate_series(1, 40000) g;

INSERT INTO teglar_bogi (post_id, teg)
SELECT DISTINCT (random() * 4999)::INT + 1,
       (ARRAY['sql','python','web'])[(random() * 2)::INT + 1]
FROM generate_series(1, 8000) g;

-- 1-postga ATAYLAB aniq ma'lumot: 4 ta izoh va 3 ta teg (6-bo'lim uchun)
DELETE FROM izohlar     WHERE post_id = 1;
DELETE FROM teglar_bogi WHERE post_id = 1;
INSERT INTO izohlar (post_id, muallif, matn) VALUES
    (1, 'Aziz',    'Birinchi izoh'),
    (1, 'Dilnoza', 'Ikkinchi izoh'),
    (1, 'Sardor',  'Uchinchi izoh'),
    (1, 'Nodira',  'To''rtinchi izoh');
INSERT INTO teglar_bogi (post_id, teg) VALUES (1,'sql'), (1,'python'), (1,'web');

CREATE INDEX idx_izohlar_post ON izohlar(post_id);
ANALYZE postlar; ANALYZE izohlar; ANALYZE teglar_bogi;

-- ─────────────────────────────────────────────────────────────────────
-- 1) N+1 QANDAY KO'RINADI
--    ORM kodi:  for post in Post.query.limit(20): print(post.izohlar)
-- ─────────────────────────────────────────────────────────────────────
-- 1-so'rov — postlar ro'yxati:
EXPLAIN (ANALYZE, TIMING OFF)
SELECT id, sarlavha FROM postlar ORDER BY id LIMIT 20;

-- Keyin HAR BIR post uchun alohida so'rov. Bu yerda bittasi ko'rsatilgan,
-- ilovada esa 20 tasi KETMA-KET ketadi:
EXPLAIN (ANALYZE, TIMING OFF)
SELECT * FROM izohlar WHERE post_id = 1;
-- Har biri ~0.05 ms. Jami SQL ~1 ms. Lekin 21 marta tarmoq borib-kelishi
-- (har biri 1-2 ms) => 40+ ms. Sekin so'rovlar jurnalida HECH NARSA yo'q.

-- ─────────────────────────────────────────────────────────────────────
-- 2) Yechim A: bitta JOIN
-- ─────────────────────────────────────────────────────────────────────
EXPLAIN (ANALYZE, TIMING OFF)
SELECT p.id, p.sarlavha, i.id AS izoh_id, i.matn
FROM (SELECT id, sarlavha FROM postlar ORDER BY id LIMIT 20) p
LEFT JOIN izohlar i ON i.post_id = p.id
ORDER BY p.id, i.id;
-- Kamchiligi: p.sarlavha har bir izoh qatorida takrorlanadi.

-- ─────────────────────────────────────────────────────────────────────
-- 3) Yechim B: bitta batch so'rov (IN) — ORM dagi eager loading aynan shu
-- ─────────────────────────────────────────────────────────────────────
EXPLAIN (ANALYZE, TIMING OFF)
SELECT * FROM izohlar
WHERE post_id IN (SELECT id FROM postlar ORDER BY id LIMIT 20)
ORDER BY post_id, id;
-- 20 ta so'rov o'rniga 2 ta. Natijani kodda post_id bo'yicha guruhlaysiz.

-- ─────────────────────────────────────────────────────────────────────
-- 4) Yechim C: jsonb_agg — API darhol ichma-ich JSON kutsa
-- ─────────────────────────────────────────────────────────────────────
EXPLAIN (ANALYZE, TIMING OFF)
SELECT p.id, p.sarlavha,
       COALESCE(
           jsonb_agg(jsonb_build_object('id', i.id, 'matn', i.matn) ORDER BY i.id)
               FILTER (WHERE i.id IS NOT NULL),
           '[]'::jsonb
       ) AS izohlar
FROM (SELECT id, sarlavha FROM postlar ORDER BY id LIMIT 20) p
LEFT JOIN izohlar i ON i.post_id = p.id
GROUP BY p.id, p.sarlavha
ORDER BY p.id;
-- FILTER (WHERE i.id IS NOT NULL) muhim: usiz izohsiz post uchun
-- massivda [{"id": null, "matn": null}] paydo bo'ladi.

-- ─────────────────────────────────────────────────────────────────────
-- 5) Yechim D: LATERAL — har postdan faqat OXIRGI 3 izoh
--    Bu masalani oddiy JOIN bilan yechib BO'LMAYDI: LIMIT butun
--    natijaga qo'llanadi, har bir guruhga emas.
-- ─────────────────────────────────────────────────────────────────────
EXPLAIN (ANALYZE, TIMING OFF)
SELECT p.id, p.sarlavha, i.id AS izoh_id, i.matn
FROM (SELECT id, sarlavha FROM postlar ORDER BY id LIMIT 20) p
LEFT JOIN LATERAL (
    SELECT id, matn FROM izohlar
    WHERE post_id = p.id           -- <-- LATERAL aynan shuni mumkin qiladi
    ORDER BY id DESC
    LIMIT 3
) i ON TRUE
ORDER BY p.id, i.id DESC;

-- ─────────────────────────────────────────────────────────────────────
-- 6) TUZOQ: ikkita 1:N jadvalni bir vaqtda qo'shish -> dekart ko'paytmasi
-- ─────────────────────────────────────────────────────────────────────
-- 1-postda 4 ta izoh va 3 ta teg bor. Haqiqiy sonlar:
SELECT (SELECT COUNT(*) FROM izohlar     WHERE post_id = 1) AS izoh_soni,
       (SELECT COUNT(*) FROM teglar_bogi WHERE post_id = 1) AS teg_soni;
--  izoh_soni | teg_soni
--          4 |        3

-- Endi ikkalasini bitta so'rovda qo'shamiz:
SELECT COUNT(*) AS notogri_qatorlar
FROM postlar p
LEFT JOIN izohlar i     ON i.post_id = p.id
LEFT JOIN teglar_bogi t ON t.post_id = p.id
WHERE p.id = 1;
--  notogri_qatorlar
--                12        <-- 4 * 3 = 12, ya'ni 4 emas!
-- Endi COUNT(i.id) 12 deb YOLG'ON gapiradi va SUM ham xato bo'ladi.

-- To'g'rilash 1: COUNT(DISTINCT ...) — to'g'ri, lekin saralash qo'shadi
SELECT p.id,
       COUNT(DISTINCT i.id)  AS izoh_soni,
       COUNT(DISTINCT t.teg) AS teg_soni
FROM postlar p
LEFT JOIN izohlar i     ON i.post_id = p.id
LEFT JOIN teglar_bogi t ON t.post_id = p.id
WHERE p.id = 1
GROUP BY p.id;

-- To'g'rilash 2: umuman qo'shmasdan, alohida agregatlar (odatda tezroq)
SELECT p.id,
       (SELECT COUNT(*) FROM izohlar     WHERE post_id = p.id) AS izoh_soni,
       (SELECT COUNT(*) FROM teglar_bogi WHERE post_id = p.id) AS teg_soni
FROM postlar p WHERE p.id = 1;

-- ─────────────────────────────────────────────────────────────────────
-- 7) N+1 NI ANIQLASH — produksiyada
-- ─────────────────────────────────────────────────────────────────────
-- pg_stat_statements kengaytmasi orqali. DIQQAT: bu yerda SEKIN so'rovlar
-- emas, KO'P CHAQIRILGAN so'rovlar qidiriladi — N+1 ning butun mohiyati shu.
--
--   SELECT calls,
--          ROUND(mean_exec_time::NUMERIC, 3)          AS ortacha_ms,
--          ROUND((calls * mean_exec_time)::NUMERIC, 1) AS jami_ms,
--          LEFT(query, 80)                             AS sorov
--   FROM pg_stat_statements
--   ORDER BY calls DESC
--   LIMIT 20;
--
-- Kengaytma o'rnatilganini tekshirish:
SELECT
    (SELECT COUNT(*) FROM pg_available_extensions WHERE name = 'pg_stat_statements') AS mavjud,
    (SELECT COUNT(*) FROM pg_extension            WHERE extname = 'pg_stat_statements') AS ornatilgan;
-- mavjud=1, ornatilgan=0 bo'lsa: postgresql.conf da
-- shared_preload_libraries = 'pg_stat_statements' qo'shib, serverni
-- qayta ishga tushirish va CREATE EXTENSION bajarish kerak.

-- Jadval darajasidagi belgi: seq_scan juda ko'p bo'lsa ham N+1 dan darak
SELECT relname, seq_scan, idx_scan, n_live_tup
FROM pg_stat_user_tables
WHERE n_live_tup > 1000
ORDER BY seq_scan DESC
LIMIT 10;
