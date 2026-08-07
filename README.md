-- ═══════════════════════════════════════════════════════════════════════
-- E-commerce sxemasi v2 — Asoslari kursidagi capstone sxemasini
-- ushbu kursdagi hamma narsani qo'llab qayta loyihalash
-- ═══════════════════════════════════════════════════════════════════════

DROP TABLE IF EXISTS zaxira_harakatlari;
DROP TABLE IF EXISTS tolovlar;
DROP TABLE IF EXISTS buyurtma_elementlari;
DROP TABLE IF EXISTS buyurtmalar;
DROP TABLE IF EXISTS manzillar;
DROP TABLE IF EXISTS mahsulotlar;
DROP TABLE IF EXISTS kategoriyalar;
DROP TABLE IF EXISTS mijozlar;
DROP TABLE IF EXISTS shaharlar;

-- ── TUZATISH 2: shahar erkin matn emas, lug'at jadval (3NF) ───────────
CREATE TABLE shaharlar (
    id       INTEGER     GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nomi     VARCHAR(60) NOT NULL,
    viloyati VARCHAR(60) NOT NULL,
    CONSTRAINT shaharlar_nomi_viloyat_uq UNIQUE (nomi, viloyati)
);

CREATE TABLE mijozlar (
    id              INTEGER      GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    -- TUZATISH: ism uchun yetarli uzunlik
    ism             VARCHAR(120) NOT NULL,
    email           VARCHAR(160) NOT NULL,
    shahar_id       INTEGER      REFERENCES shaharlar(id) ON DELETE SET NULL,
    royxatdan       TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    -- TUZATISH: soft delete — mijozni o'chirmasdan "yo'q" qilish
    ochirilgan_sana TIMESTAMPTZ,
    CONSTRAINT mijozlar_email_uq  UNIQUE (email),
    CONSTRAINT mijozlar_email_fmt CHECK (email LIKE '%_@_%._%')
);

-- ── TUZATISH: kategoriya ierarxiyasi (self-referential 1:N) ───────────
CREATE TABLE kategoriyalar (
    id     INTEGER     GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    ota_id INTEGER     REFERENCES kategoriyalar(id) ON DELETE RESTRICT,
    nomi   VARCHAR(60) NOT NULL,
    CONSTRAINT kategoriyalar_nomi_uq UNIQUE (nomi),
    -- o'zi o'zining otasi bo'lolmaydi
    CONSTRAINT kategoriyalar_ota_ozi_emas CHECK (ota_id IS NULL OR ota_id <> id)
);

CREATE TABLE mahsulotlar (
    id            INTEGER       GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    kategoriya_id INTEGER       NOT NULL REFERENCES kategoriyalar(id) ON DELETE RESTRICT,
    nomi          VARCHAR(150)  NOT NULL,
    -- TUZATISH 4: NUMERIC(10,2) so'm uchun kichik -> NUMERIC(14,2)
    narx          NUMERIC(14,2) NOT NULL CHECK (narx > 0),
    -- TUZATISH 6: zaxira — zaxira_harakatlari ustidagi KESH.
    -- Uni faqat trigger yangilaydi, ilova kodi emas.
    zaxira        INTEGER       NOT NULL DEFAULT 0 CHECK (zaxira >= 0),
    -- TUZATISH: katalogdan olib tashlash uchun soft delete
    faol          BOOLEAN       NOT NULL DEFAULT TRUE,
    yaratilgan    TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    yangilangan   TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

-- ── TUZATISH 3: mijozning saqlangan manzillari (1:N) ──────────────────
CREATE TABLE manzillar (
    id          INTEGER      GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    mijoz_id    INTEGER      NOT NULL REFERENCES mijozlar(id) ON DELETE CASCADE,
    shahar_id   INTEGER      NOT NULL REFERENCES shaharlar(id) ON DELETE RESTRICT,
    kocha_uy    VARCHAR(200) NOT NULL,
    telefon     VARCHAR(20)  NOT NULL,
    asosiy      BOOLEAN      NOT NULL DEFAULT FALSE
);

-- Bir mijozda faqat BITTA asosiy manzil — partial unique index
CREATE UNIQUE INDEX manzillar_bitta_asosiy
    ON manzillar (mijoz_id) WHERE asosiy;

CREATE TABLE buyurtmalar (
    id           INTEGER     GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    mijoz_id     INTEGER     NOT NULL REFERENCES mijozlar(id) ON DELETE RESTRICT,
    -- TUZATISH 5: bajarilish holati va to'lov holati AJRATILDI
    holat        VARCHAR(20) NOT NULL DEFAULT 'yangi'
                 CHECK (holat IN ('yangi','yigilmoqda','jonatildi','yetkazildi','bekor')),
    -- TUZATISH 3: manzil FK EMAS, balki NUSXA. Mijoz ko'chib ketsa ham
    -- bu buyurtma qayerga yetkazilgani o'zgarmaydi.
    yetkazish_manzili TEXT     NOT NULL,
    yetkazish_telefoni VARCHAR(20) NOT NULL,
    yaratilgan   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE buyurtma_elementlari (
    id          INTEGER       GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    buyurtma_id INTEGER       NOT NULL REFERENCES buyurtmalar(id) ON DELETE CASCADE,
    mahsulot_id INTEGER       NOT NULL REFERENCES mahsulotlar(id) ON DELETE RESTRICT,
    miqdor      INTEGER       NOT NULL CHECK (miqdor > 0),
    narx_birlik NUMERIC(14,2) NOT NULL CHECK (narx_birlik > 0),
    -- TUZATISH 1: eng muhim tuzatish. Busiz bitta mahsulot bitta
    -- buyurtmada ikki marta paydo bo'lib, hisobotlarni buzardi.
    CONSTRAINT bel_buyurtma_mahsulot_uq UNIQUE (buyurtma_id, mahsulot_id)
);

-- ── TUZATISH 5: to'lov — mustaqil mohiyat ─────────────────────────────
CREATE TABLE tolovlar (
    id            INTEGER       GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    buyurtma_id   INTEGER       NOT NULL REFERENCES buyurtmalar(id) ON DELETE RESTRICT,
    summa         NUMERIC(14,2) NOT NULL CHECK (summa > 0),
    usul          VARCHAR(20)   NOT NULL
                  CHECK (usul IN ('naqd','karta','click','payme','bank')),
    holat         VARCHAR(20)   NOT NULL DEFAULT 'kutmoqda'
                  CHECK (holat IN ('kutmoqda','tasdiqlandi','rad_etildi','qaytarildi')),
    tranzaksiya_id VARCHAR(64),
    vaqti         TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    -- tashqi to'lov tizimidagi ID takrorlanmasin
    CONSTRAINT tolovlar_tranzaksiya_uq UNIQUE (tranzaksiya_id)
);

-- ── TUZATISH 6: zaxira harakati — kirim/chiqim tarixi ─────────────────
CREATE TABLE zaxira_harakatlari (
    id          INTEGER     GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    mahsulot_id INTEGER     NOT NULL REFERENCES mahsulotlar(id) ON DELETE RESTRICT,
    -- musbat = kirim (yetkazib berildi), manfiy = chiqim (sotildi)
    ozgarish    INTEGER     NOT NULL CHECK (ozgarish <> 0),
    sabab       VARCHAR(20) NOT NULL
                CHECK (sabab IN ('kirim','sotuv','qaytarish','inventarizatsiya','yaroqsiz')),
    buyurtma_id INTEGER     REFERENCES buyurtmalar(id) ON DELETE SET NULL,
    vaqti       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Kesh ustunini harakatlar bilan sinxron ushlab turuvchi trigger
CREATE OR REPLACE FUNCTION zaxirani_yangilash() RETURNS TRIGGER AS $$
BEGIN
    UPDATE mahsulotlar
    SET zaxira = zaxira + NEW.ozgarish,
        yangilangan = NOW()
    WHERE id = NEW.mahsulot_id;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER zaxira_harakat_trigger
    AFTER INSERT ON zaxira_harakatlari
    FOR EACH ROW EXECUTE FUNCTION zaxirani_yangilash();

-- ── Indekslar: har FK ga + tez-tez ishlatiladigan filtrlarga ──────────
CREATE INDEX mijozlar_shahar_idx        ON mijozlar (shahar_id);
CREATE INDEX kategoriyalar_ota_idx      ON kategoriyalar (ota_id);
CREATE INDEX mahsulotlar_kategoriya_idx ON mahsulotlar (kategoriya_id);
CREATE INDEX manzillar_mijoz_idx        ON manzillar (mijoz_id);
CREATE INDEX buyurtmalar_mijoz_vaqt_idx ON buyurtmalar (mijoz_id, yaratilgan DESC);
CREATE INDEX bel_mahsulot_idx           ON buyurtma_elementlari (mahsulot_id);
CREATE INDEX tolovlar_buyurtma_idx      ON tolovlar (buyurtma_id);
CREATE INDEX zaxira_mahsulot_vaqt_idx   ON zaxira_harakatlari (mahsulot_id, vaqti DESC);

-- ═══════════════════════════════════════════════════════════════════════
-- Test ma'lumot
-- ═══════════════════════════════════════════════════════════════════════
INSERT INTO shaharlar (nomi, viloyati) VALUES
    ('Toshkent',  'Toshkent shahri'),
    ('Samarqand', 'Samarqand viloyati'),
    ('Buxoro',    'Buxoro viloyati');

INSERT INTO mijozlar (ism, email, shahar_id) VALUES
    ('Aziz Karimov',     'aziz@shop.uz',   1),
    ('Dilnoza Rasulova', 'dilya@shop.uz',  2),
    ('Sardor Tursunov',  'sardor@shop.uz', 1);

-- Ierarxik kategoriyalar
INSERT INTO kategoriyalar (ota_id, nomi) VALUES (NULL, 'Elektronika');
INSERT INTO kategoriyalar (ota_id, nomi) VALUES (1, 'Telefonlar'), (1, 'Noutbuklar');

INSERT INTO mahsulotlar (kategoriya_id, nomi, narx) VALUES
    (2, 'iPhone 15',      15000000),
    (2, 'Samsung S24',    12000000),
    (3, 'MacBook Pro 14', 22000000);

-- Zaxira faqat harakat orqali o'zgaradi — trigger keshni yangilaydi
INSERT INTO zaxira_harakatlari (mahsulot_id, ozgarish, sabab) VALUES
    (1, 10, 'kirim'), (2, 8, 'kirim'), (3, 5, 'kirim');

SELECT nomi, zaxira FROM mahsulotlar ORDER BY id;   -- 10, 8, 5

INSERT INTO manzillar (mijoz_id, shahar_id, kocha_uy, telefon, asosiy) VALUES
    (1, 1, 'Amir Temur ko''chasi, 15-uy', '+998901112233', TRUE),
    (1, 1, 'Yunusobod 4-kvartal, 22-uy',  '+998901112233', FALSE),
    (2, 2, 'Registon ko''chasi, 7-uy',    '+998907778899', TRUE);

-- Ikkinchi "asosiy" manzil bloklanadi:
-- UPDATE manzillar SET asosiy = TRUE WHERE id = 2;
-- ERROR:  duplicate key value violates unique constraint "manzillar_bitta_asosiy"

-- Buyurtma: manzil NUSXA sifatida yoziladi, FK sifatida emas
INSERT INTO buyurtmalar (mijoz_id, holat, yetkazish_manzili, yetkazish_telefoni) VALUES
    (1, 'yetkazildi', 'Toshkent, Amir Temur ko''chasi, 15-uy', '+998901112233'),
    (2, 'yigilmoqda', 'Samarqand, Registon ko''chasi, 7-uy',   '+998907778899');

INSERT INTO buyurtma_elementlari (buyurtma_id, mahsulot_id, miqdor, narx_birlik) VALUES
    (1, 1, 1, 15000000),
    (1, 3, 1, 22000000),
    (2, 2, 2, 12000000);

-- Dublikat endi BLOKLANADI (v1 da bu bemalol o'tib ketardi):
-- INSERT INTO buyurtma_elementlari (buyurtma_id, mahsulot_id, miqdor, narx_birlik)
-- VALUES (1, 1, 5, 14000000);
-- ERROR:  duplicate key value violates unique constraint "bel_buyurtma_mahsulot_uq"

-- Sotuv zaxiradan chiqim yaratadi
INSERT INTO zaxira_harakatlari (mahsulot_id, ozgarish, sabab, buyurtma_id) VALUES
    (1, -1, 'sotuv', 1), (3, -1, 'sotuv', 1), (2, -2, 'sotuv', 2);

SELECT nomi, zaxira FROM mahsulotlar ORDER BY id;   -- 9, 6, 4

INSERT INTO tolovlar (buyurtma_id, summa, usul, holat, tranzaksiya_id) VALUES
    (1, 37000000, 'click', 'tasdiqlandi', 'CLK-2026-0001'),
    (2, 12000000, 'karta', 'tasdiqlandi', 'CRD-2026-0002');
-- Diqqat: 2-buyurtma qisman to'langan (24 mln dan 12 mln).
-- v1 sxemasida buni ifodalash IMKONSIZ edi.

-- ═══════════════════════════════════════════════════════════════════════
-- v2 sxema ochgan yangi imkoniyatlar
-- ═══════════════════════════════════════════════════════════════════════

-- 1) Viloyat bo'yicha daromad — v1 da IMKONSIZ edi (viloyat saqlanmagan)
SELECT s.viloyati,
       COUNT(DISTINCT b.id)              AS buyurtmalar,
       SUM(e.miqdor * e.narx_birlik)     AS daromad
FROM buyurtmalar b
JOIN mijozlar m  ON m.id = b.mijoz_id
JOIN shaharlar s ON s.id = m.shahar_id
JOIN buyurtma_elementlari e ON e.buyurtma_id = b.id
GROUP BY s.viloyati
ORDER BY daromad DESC;

-- 2) To'liq to'lanmagan buyurtmalar — v1 da IMKONSIZ edi
SELECT b.id,
       SUM(e.miqdor * e.narx_birlik) AS buyurtma_summasi,
       COALESCE(t.tolangan, 0)       AS tolangan,
       SUM(e.miqdor * e.narx_birlik) - COALESCE(t.tolangan, 0) AS qarz
FROM buyurtmalar b
JOIN buyurtma_elementlari e ON e.buyurtma_id = b.id
LEFT JOIN (
    SELECT buyurtma_id, SUM(summa) AS tolangan
    FROM tolovlar WHERE holat = 'tasdiqlandi'
    GROUP BY buyurtma_id
) t ON t.buyurtma_id = b.id
GROUP BY b.id, t.tolangan
HAVING SUM(e.miqdor * e.narx_birlik) > COALESCE(t.tolangan, 0);

-- 3) Zaxira tarixi — "nega 3 ta kam?" savoliga javob. v1 da IMKONSIZ.
SELECT p.nomi, z.vaqti, z.ozgarish, z.sabab, z.buyurtma_id
FROM zaxira_harakatlari z
JOIN mahsulotlar p ON p.id = z.mahsulot_id
ORDER BY p.nomi, z.vaqti;

-- 4) Kategoriya ierarxiyasi — recursive CTE. v1 da IMKONSIZ.
WITH RECURSIVE kat_yol AS (
    SELECT id, nomi, nomi::TEXT AS yol FROM kategoriyalar WHERE ota_id IS NULL
    UNION ALL
    SELECT k.id, k.nomi, y.yol || ' > ' || k.nomi
    FROM kategoriyalar k JOIN kat_yol y ON k.ota_id = y.id
)
SELECT y.yol AS kategoriya_yoli, COUNT(p.id) AS mahsulotlar
FROM kat_yol y
LEFT JOIN mahsulotlar p ON p.kategoriya_id = y.id
GROUP BY y.yol
ORDER BY y.yol;

-- 5) Kesh tekshiruvi: mahsulotlar.zaxira harakatlar bilan mos keladimi?
SELECT p.id, p.nomi, p.zaxira AS keshdagi,
       COALESCE(SUM(z.ozgarish), 0) AS haqiqiy
FROM mahsulotlar p
LEFT JOIN zaxira_harakatlari z ON z.mahsulot_id = p.id
GROUP BY p.id, p.nomi, p.zaxira
HAVING p.zaxira <> COALESCE(SUM(z.ozgarish), 0);
-- Bo'sh natija = kesh to'g'ri.
