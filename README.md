-- ═══════════════════════════════════════════════════════════════════════
-- Tranzaksiyalar va izolyatsiya darajalari
-- ═══════════════════════════════════════════════════════════════════════

DROP TABLE IF EXISTS hisoblar;
CREATE TABLE hisoblar (
    id     SERIAL        PRIMARY KEY,
    egasi  VARCHAR(40)   NOT NULL,
    balans NUMERIC(12,2) NOT NULL CHECK (balans >= 0)
);
INSERT INTO hisoblar (egasi, balans) VALUES ('Aziz', 1000000), ('Dilnoza', 500000);

-- ─────────────────────────────────────────────────────────────────────
-- 1) Standart daraja
-- ─────────────────────────────────────────────────────────────────────
SHOW default_transaction_isolation;
--  read committed

-- ─────────────────────────────────────────────────────────────────────
-- 2) READ UNCOMMITTED — qabul qilinadi, lekin dirty read BO'LMAYDI
-- ─────────────────────────────────────────────────────────────────────
BEGIN TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
    SHOW transaction_isolation;
    --  read uncommitted
    -- DIQQAT: PostgreSQL buyruqni qabul qildi va shu nom bilan qaytardi,
    -- LEKIN xatti-harakati READ COMMITTED bilan bir xil. MVCC arxitekturasi
    -- commit qilinmagan ma'lumotni o'qishni prinsipial ravishda imkonsiz qiladi.
COMMIT;

-- ─────────────────────────────────────────────────────────────────────
-- 3) REPEATABLE READ — snapshot BIRINCHI so'rovda muzlaydi
-- ─────────────────────────────────────────────────────────────────────
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
    SHOW transaction_isolation;
    SELECT SUM(balans) AS boshlangich FROM hisoblar;
    -- Shu paytdan boshlab, boshqa sessiya nima qilsa ham, bu tranzaksiya
    -- ichidagi barcha SELECT lar AYNAN shu snapshot ni ko'radi.
    SELECT txid_current()        AS tranzaksiya_id;
    SELECT pg_current_snapshot() AS snapshot;
COMMIT;

-- ─────────────────────────────────────────────────────────────────────
-- 4) SERIALIZABLE
-- ─────────────────────────────────────────────────────────────────────
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
    SHOW transaction_isolation;
    SELECT COUNT(*) FROM hisoblar;
COMMIT;

-- ─────────────────────────────────────────────────────────────────────
-- 5) IKKI SESSIYALI SSENARIY — non-repeatable read
--    (bitta skriptda ko'rsatib bo'lmaydi; ikki psql oynasida sinang)
-- ─────────────────────────────────────────────────────────────────────
--   Sessiya A                              Sessiya B
--   ----------------------------------     -----------------------------
--   BEGIN;  -- READ COMMITTED
--   SELECT balans FROM hisoblar
--     WHERE id=1;        --> 1000000
--                                          BEGIN;
--                                          UPDATE hisoblar
--                                            SET balans = 700000
--                                            WHERE id=1;
--                                          COMMIT;
--   SELECT balans FROM hisoblar
--     WHERE id=1;        --> 700000   <-- O'ZGARDI! non-repeatable read
--   COMMIT;
--
--   Ayni ssenariy REPEATABLE READ da:
--   ikkinchi SELECT ham 1000000 qaytaradi — snapshot muzlagan.
--   Agar A o'sha qatorni UPDATE qilmoqchi bo'lsa:
--     ERROR:  could not serialize access due to concurrent update
--     SQLSTATE 40001  -> ilova tranzaksiyani QAYTA boshlashi kerak.

-- ─────────────────────────────────────────────────────────────────────
-- 6) SAVEPOINT — tranzaksiya ichidagi qisman qaytarish
-- ─────────────────────────────────────────────────────────────────────
-- AVVAL: savepoint SIZ nima bo'lishini ko'ramiz
BEGIN;
    UPDATE hisoblar SET balans = balans - 1 WHERE egasi = 'Aziz';
    UPDATE hisoblar SET balans = balans - 999999999 WHERE egasi = 'Dilnoza';
    --  ERROR:  new row for relation "hisoblar" violates check constraint
    SELECT 'bu so''rov ham bajarilmaydi' AS eslatma;
    --  ERROR:  current transaction is aborted, commands ignored until
    --          end of transaction block
COMMIT;
--  Natija: ROLLBACK. Birinchi UPDATE ham yo'qoldi.

-- ENDI: savepoint BILAN
BEGIN;
    UPDATE hisoblar SET balans = balans - 200000 WHERE egasi = 'Aziz';
    SAVEPOINT sp1;
    UPDATE hisoblar SET balans = balans - 999999999 WHERE egasi = 'Dilnoza';
    --  ERROR:  ... violates check constraint "hisoblar_balans_check"
    ROLLBACK TO SAVEPOINT sp1;   -- faqat sp1 dan KEYINGI ish bekor bo'ladi
    UPDATE hisoblar SET balans = balans + 200000 WHERE egasi = 'Dilnoza';
COMMIT;
--  Natija: COMMIT muvaffaqiyatli. O'tkazma amalga oshdi:
--  Aziz 800000.00, Dilnoza 700000.00
SELECT * FROM hisoblar ORDER BY id;

-- ─────────────────────────────────────────────────────────────────────
-- 7) MVCC ni "ko'rish": har qatorning yashirin xizmat ustunlari
-- ─────────────────────────────────────────────────────────────────────
SELECT id, egasi, balans,
       xmin,   -- qatorning bu versiyasini YARATGAN tranzaksiya
       xmax,   -- uni o'chirgan/yangilagan tranzaksiya (0 = hali tirik)
       ctid    -- jismoniy joylashuv: (sahifa, qator)
FROM hisoblar ORDER BY id;

-- UPDATE dan keyin ctid O'ZGARADI — chunki bu joyida o'zgartirish emas,
-- yangi versiya yozish. Eski versiya diskda qoladi va uni VACUUM tozalaydi.
BEGIN;
    UPDATE hisoblar SET balans = balans + 1 WHERE id = 1;
    SELECT id, xmin, xmax, ctid FROM hisoblar WHERE id = 1;
ROLLBACK;

-- ─────────────────────────────────────────────────────────────────────
-- 8) Qayta urinish uchun xato kodlari
-- ─────────────────────────────────────────────────────────────────────
SELECT '40001' AS kod, 'serialization_failure' AS nomi,
       'Tranzaksiyani boshidan qayta bajaring' AS harakat
UNION ALL
SELECT '40P01', 'deadlock_detected',
       'Tranzaksiyani boshidan qayta bajaring';

-- Ilovadagi naqsh (psevdokod):
--   for urinish in range(3):
--       try:
--           with db.begin(isolation_level="REPEATABLE READ"):
--               ...ish...
--           break
--       except SerializationFailure:
--           sleep(0.05 * 2 ** urinish)   -- eksponensial kutish
--           continue
--   else:
--       raise  -- 3 urinishdan keyin ham bo'lmadi
