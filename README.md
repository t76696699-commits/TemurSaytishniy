-- ═══════════════════════════════════════════════════════════════════════
-- Qulflar (locks), SKIP LOCKED navbati va deadlock
-- ═══════════════════════════════════════════════════════════════════════

DROP TABLE IF EXISTS navbat;
DROP TABLE IF EXISTS hisoblar;

CREATE TABLE hisoblar (
    id     SERIAL        PRIMARY KEY,
    egasi  VARCHAR(40)   NOT NULL,
    balans NUMERIC(12,2) NOT NULL CHECK (balans >= 0)
);
INSERT INTO hisoblar (egasi, balans) VALUES
    ('Aziz', 1000000), ('Dilnoza', 500000), ('Sardor', 300000);

-- ─────────────────────────────────────────────────────────────────────
-- 1) FOR UPDATE — o'qib-o'zgartirish naqshini himoyalash
-- ─────────────────────────────────────────────────────────────────────
BEGIN;
    -- Bu qator endi tranzaksiya oxirigacha BAND. Boshqa sessiya uni
    -- o'zgartirmoqchi bo'lsa — kutadi.
    SELECT id, egasi, balans FROM hisoblar WHERE id = 1 FOR UPDATE;
    UPDATE hisoblar SET balans = balans - 100000 WHERE id = 1;
COMMIT;

-- Eslatma: agar hisob-kitob bazaning o'zida bajarilsa, FOR UPDATE
-- umuman kerak emas — UPDATE qatorni o'zi qulflaydi va qiymatni
-- ATOMAR o'qib-yozadi:
--     UPDATE hisoblar SET balans = balans - 100000 WHERE id = 1;
-- Bu "lost update" ga qarshi eng oddiy va eng ishonchli himoya.

-- ─────────────────────────────────────────────────────────────────────
-- 2) Qulf turlari (kuchlidan kuchsizga)
-- ─────────────────────────────────────────────────────────────────────
BEGIN;
    SELECT id FROM hisoblar WHERE id = 1 FOR UPDATE;        -- eng kuchli
    SELECT id FROM hisoblar WHERE id = 2 FOR NO KEY UPDATE; -- UPDATE shuni oladi
    SELECT id FROM hisoblar WHERE id = 3 FOR SHARE;         -- o'zgartirishni bloklaydi
    SELECT id FROM hisoblar WHERE id = 3 FOR KEY SHARE;     -- FK tekshiruvi shuni oladi
COMMIT;

-- ─────────────────────────────────────────────────────────────────────
-- 3) NOWAIT — kutish o'rniga darhol xato
-- ─────────────────────────────────────────────────────────────────────
BEGIN;
    SELECT id FROM hisoblar WHERE id = 1 FOR UPDATE NOWAIT;
    -- Qator band bo'lsa:
    --   ERROR:  could not obtain lock on row in relation "hisoblar"
    -- Interaktiv ilovada "yozuv band, keyinroq urinib ko'ring" uchun qulay.
COMMIT;

-- ─────────────────────────────────────────────────────────────────────
-- 4) SKIP LOCKED — ko'p ishchili navbat (Celery/Sidekiq ning SQL asosi)
-- ─────────────────────────────────────────────────────────────────────
CREATE TABLE navbat (
    id      BIGSERIAL   PRIMARY KEY,
    vazifa  TEXT        NOT NULL,
    holat   VARCHAR(20) NOT NULL DEFAULT 'kutmoqda',
    olingan TIMESTAMPTZ
);
INSERT INTO navbat (vazifa) SELECT 'Vazifa ' || g FROM generate_series(1, 10) g;

-- Qisman indeks: faqat kutayotgan vazifalar indekslanadi (5-darsga qarang)
CREATE INDEX idx_navbat_holat ON navbat(holat, id) WHERE holat = 'kutmoqda';

-- Har bir ishchi shu so'rovni bajaradi. SKIP LOCKED tufayli ular
-- BIR-BIRINI KUTMAYDI: 2-ishchi 1-ishchi olgan qatorlarni sakrab o'tadi.
BEGIN;
    WITH keyingi AS (
        SELECT id FROM navbat
        WHERE holat = 'kutmoqda'
        ORDER BY id
        FOR UPDATE SKIP LOCKED
        LIMIT 3
    )
    UPDATE navbat n
    SET holat = 'bajarilmoqda', olingan = NOW()
    FROM keyingi k
    WHERE n.id = k.id
    RETURNING n.id, n.vazifa;
COMMIT;

SELECT holat, COUNT(*) FROM navbat GROUP BY holat ORDER BY holat;
--  bajarilmoqda | 3
--  kutmoqda     | 7

-- ─────────────────────────────────────────────────────────────────────
-- 5) lock_timeout — cheksiz kutmaslik
-- ─────────────────────────────────────────────────────────────────────
BEGIN;
    SET LOCAL lock_timeout = '3s';   -- LOCAL: faqat shu tranzaksiya uchun
    SELECT id FROM hisoblar WHERE id = 1 FOR UPDATE;
    -- 3 sekunddan keyin ham qulf olinmasa:
    --   ERROR:  canceling statement due to lock timeout
COMMIT;

-- ─────────────────────────────────────────────────────────────────────
-- 6) DEADLOCK — ikki sessiyada takrorlanadigan ssenariy
--    Ikki psql oynasini oching va quyidagilarni PARALLEL bajaring:
-- ─────────────────────────────────────────────────────────────────────
--   Sessiya A                               Sessiya B
--   -----------------------------------     -----------------------------------
--   BEGIN;                                  BEGIN;
--   UPDATE hisoblar SET balans=balans-100    UPDATE hisoblar SET balans=balans-50
--     WHERE id = 1;   -- 1-qator qulflandi     WHERE id = 2;   -- 2-qator qulflandi
--   SELECT pg_sleep(2);                      SELECT pg_sleep(2);
--   UPDATE hisoblar SET balans=balans+100    UPDATE hisoblar SET balans=balans+50
--     WHERE id = 2;   -- B ni kutadi           WHERE id = 1;   -- A ni kutadi
--                          \_______ HALQA _______/
--   COMMIT;                                  <-- BU YERDA XATO:
--
--   ERROR:  deadlock detected
--   DETAIL:  Process 1023893 waits for ShareLock on transaction 13418187;
--            blocked by process 1023894.
--            Process 1023894 waits for ShareLock on transaction 13418186;
--            blocked by process 1023893.
--   HINT:  See server log for query details.
--   CONTEXT:  while updating tuple (0,1) in relation "hisoblar"
--
--   Diqqat: FAQAT BITTA sessiya qurbon bo'ldi (B), A esa normal COMMIT bo'ldi.

SHOW deadlock_timeout;
--  1s  <-- PostgreSQL shuncha kutgandan keyingina halqa qidiradi.
--          Tekshiruv qimmat, shuning uchun u darhol bajarilmaydi.

-- ─────────────────────────────────────────────────────────────────────
-- 7) DEADLOCK OLDINI OLISH: har doim BIR XIL tartibda qulflash
--    Bu eng ishonchli usul — halqa matematik jihatdan hosil bo'lolmaydi.
-- ─────────────────────────────────────────────────────────────────────
BEGIN;
    SELECT id, egasi FROM hisoblar
    WHERE id IN (1, 2)
    ORDER BY id            -- <<< ENG MUHIM QATOR
    FOR UPDATE;
    -- Endi ikkala qator ham qulflangan va TARTIB kafolatlangan.
    UPDATE hisoblar SET balans = balans - 100 WHERE id = 1;
    UPDATE hisoblar SET balans = balans + 100 WHERE id = 2;
COMMIT;

SELECT * FROM hisoblar ORDER BY id;

-- ─────────────────────────────────────────────────────────────────────
-- 8) DIAGNOSTIKA: kim kimni bloklayapti (produksiyada)
-- ─────────────────────────────────────────────────────────────────────
SELECT pid,
       state,
       wait_event_type,
       pg_blocking_pids(pid) AS bloklovchilar,   -- bo'sh massiv = bloklanmagan
       LEFT(query, 60)       AS sorov
FROM pg_stat_activity
WHERE datname = current_database()
  AND pid <> pg_backend_pid()
ORDER BY pid;

-- Aniq qulflar ro'yxati:
SELECT locktype, relation::regclass AS jadval, mode, granted
FROM pg_locks
WHERE relation IS NOT NULL
ORDER BY granted, relation
LIMIT 20;

-- Uzoq ishlayotgan tranzaksiyalar — deadlock va bloklanishning asosiy sababi:
SELECT pid, NOW() - xact_start AS davomiylik, state, LEFT(query, 60) AS sorov
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
  AND NOW() - xact_start > INTERVAL '5 seconds'
ORDER BY xact_start;
