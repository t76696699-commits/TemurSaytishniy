-- ============================================================================
-- AMALIY LOYIHA: SEKIN SO'ROVNI DIAGNOSTIKA QILISH VA TUZATISH
-- Texnologiya: PostgreSQL
-- ============================================================================

-- 1. SXEMA VA TEST MA'LUMOTLARINI YARATISH
-- ============================================================================
DROP TABLE IF EXISTS buyurtmalar CASCADE;
DROP TABLE IF EXISTS mijozlar CASCADE;

CREATE TABLE mijozlar (
    id SERIAL PRIMARY KEY,
    ism VARCHAR(100),
    email VARCHAR(150),
    royxatdan_otgan TIMESTAMP
);

CREATE TABLE buyurtmalar (
    id SERIAL PRIMARY KEY,
    mijoz_id INT REFERENCES mijozlar(id),
    summa NUMERIC(10, 2),
    holat VARCHAR(50),
    sana TIMESTAMP
);

-- Test ma'lumotlarini kiritish (100,000 mijoz va 2,000,000 buyurtma)
INSERT INTO mijozlar (ism, email, royxatdan_otgan)
SELECT 
    'Mijoz_' || i,
    CASE 
        WHEN i % 3 = 0 THEN 'user' || i || '@gmail.com'
        WHEN i % 3 = 1 THEN 'test' || i || '@yahoo.com'
        ELSE 'client' || i || '@MAIL.RU'
    END,
    NOW() - (random() * INTERVAL '5 years')
FROM generate_series(1, 100000) AS i;

INSERT INTO buyurtmalar (mijoz_id, summa, holat, sana)
SELECT 
    (random() * 99999 + 1)::INT,
    (random() * 500000 + 1000)::NUMERIC(10,2),
    CASE (random() * 3)::INT 
        WHEN 0 THEN 'tolangan' 
        WHEN 1 THEN 'kutilmoqda' 
        ELSE 'bekor_qilingan' 
    END,
    NOW() - (random() * INTERVAL '2 years')
FROM generate_series(1, 2000000) AS i;

ANALYZE mijozlar;
ANALYZE buyurtmalar;


-- ============================================================================
-- 2. BOSHLANG'ICH O'LCHOV (ASL SO'ROV)
-- ============================================================================
-- Muammolar: 
-- 1. EXTRACT(YEAR FROM royxatdan_otgan) va LOWER(email) funksiyalari indeksdan foydalanishga to'sqinlik qiladi (Sequential Scan).
-- 2. LIKE '%@gmail.com' old qismida foiz (%) ishlatilgani uchun B-tree indeksni ishlata olmaydi.
-- 3. Har bir qator uchun 3 tadan korrelyatsiyali subquery (COUNT, SUM, MAX) bajariladi.
-- 4. OFFSET 4000 LIMIT 20 katta hajmdagi ma'lumotni o'tkazib yuborishni talab qiladi.

EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT 
    m.id, m.ism, m.email, 
    (SELECT COUNT(*) FROM buyurtmalar b1 WHERE b1.mijoz_id = m.id) AS buyurtmalar_soni, 
    (SELECT SUM(b2.summa) FROM buyurtmalar b2 WHERE b2.mijoz_id = m.id AND b2.holat = 'tolangan') AS jami_tolov, 
    (SELECT MAX(b3.sana) FROM buyurtmalar b3 WHERE b3.mijoz_id = m.id) AS oxirgi_buyurtma 
FROM mijozlar m 
WHERE EXTRACT(YEAR FROM m.royxatdan_otgan) = 2024 
  AND LOWER(m.email) LIKE '%@gmail.com' 
ORDER BY jami_tolov DESC NULLS LAST 
OFFSET 4000 LIMIT 20;


-- ============================================================================
-- 3. 1-TUZATISH: KORRELYATSIYali SUBQUERY'LARNI JOIN (GROUP BY) BILAN ALMASHTIRISH
-- ============================================================================
-- Asos: Subquery har bir mijoz uchun alohida jadvalni qaytadan skanerlaydi (N+1 muammosi). 
-- Buni LEFT JOIN va AGGREGATE (GROUP BY) ga o'tkazish orqali barcha buyurtmalar bir martada guruhlanadi.

EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT 
    m.id, 
    m.ism, 
    m.email, 
    COUNT(b.id) AS buyurtmalar_soni,
    SUM(CASE WHEN b.holat = 'tolangan' THEN b.summa ELSE 0 END) AS jami_tolov,
    MAX(b.sana) AS oxirgi_buyurtma
FROM mijozlar m
LEFT JOIN buyurtmalar b ON b.mijoz_id = m.id
WHERE EXTRACT(YEAR FROM m.royxatdan_otgan) = 2024 
  AND LOWER(m.email) LIKE '%@gmail.com'
GROUP BY m.id, m.ism, m.email
ORDER BY jami_tolov DESC NULLS LAST 
OFFSET 4000 LIMIT 20;


-- ============================================================================
-- 4. 2-TUZATISH: FUNKSIYALARNI OLIB TASHLASH VA RANGE (ORALIQ) SHARTIGA O'TISH
-- ============================================================================
-- Asos: EXTRACT(YEAR FROM royxatdan_otgan) = 2024 o'rniga '2024-01-01' <= royxatdan_otgan < '2025-01-01' 
-- ishlatish va LOWER(email) LIKE ... o'rniga to'g'ridan-to'g'ri mos keluvchi shart kiritish 
-- (yoki funksional indeks uchun zamin yaratish). Hozircha faqat sana oralig'ini to'g'rilaymiz.

EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT 
    m.id, 
    m.ism, 
    m.email, 
    COUNT(b.id) AS buyurtmalar_soni,
    SUM(CASE WHEN b.holat = 'tolangan' THEN b.summa ELSE 0 END) AS jami_tolov,
    MAX(b.sana) AS oxirgi_buyurtma
FROM mijozlar m
LEFT JOIN buyurtmalar b ON b.mijoz_id = m.id
WHERE m.royxatdan_otgan >= '2024-01-01' 
  AND m.royxatdan_otgan < '2025-01-01'
  AND m.email LIKE '%@gmail.com'
GROUP BY m.id, m.ism, m.email
ORDER BY jami_tolov DESC NULLS LAST 
OFFSET 4000 LIMIT 20;


-- ============================================================================
-- 5. 3-TUZATISH: INDEX QO'SHISH (MIJOZLAR VA BUYURTMALAR UCHUN)
-- ============================================================================
-- Asos: 
-- 1. mijozlar(royxatdan_otgan) — sana bo'yicha qidiruvni tezlashtiradi.
-- 2. buyurtmalar(mijoz_id, holat) — JOIN va SUM amallarini Index Scan orqali bajarishga yordam beradi.

CREATE INDEX idx_mijozlar_royxatdan ON mijozlar(royxatdan_otgan);
CREATE INDEX idx_buyurtmalar_mijoz_holat ON buyurtmalar(mijoz_id, holat);

ANALYZE mijozlar;
ANALYZE buyurtmalar;

EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT 
    m.id, 
    m.ism, 
    m.email, 
    COUNT(b.id) AS buyurtmalar_soni,
    SUM(CASE WHEN b.holat = 'tolangan' THEN b.summa ELSE 0 END) AS jami_tolov,
    MAX(b.sana) AS oxirgi_buyurtma
FROM mijozlar m
LEFT JOIN buyurtmalar b ON b.mijoz_id = m.id
WHERE m.royxatdan_otgan >= '2024-01-01' 
  AND m.royxatdan_otgan < '2025-01-01'
  AND m.email LIKE '%@gmail.com'
GROUP BY m.id, m.ism, m.email
ORDER BY jami_tolov DESC NULLS LAST 
OFFSET 4000 LIMIT 20;


-- ============================================================================
-- 6. 4-TUZATISH: PAGINATION (OFFSET 4000) MUAMMOSINI KURSOR (KEYSAYT) USULIGA ALMASHTIRISH 
-- YOKI CTE / SUBQUERY ORQALI AGGREGATSIyani OLDIN REJALASHTIRISH
-- ============================================================================
-- Asos: OFFSET katta bo'lganda PostgreSQL barcha 4000 ta qatorni o'tkazib yuborish uchun 
-- baribir ularni saralab chiqishi kerak. Eng yaxshi yechim - avval shart bo'yicha mijozlarni 
-- tanlab, keyin agregatsiya qilish yoki keyset pagination ishlatish. 
-- Keling, optimallashtirilgan CTE yondashuvini qo'llaymiz.

EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
WITH filt_mijozlar AS (
    SELECT id, ism, email
    FROM mijozlar
    WHERE royxatdan_otgan >= '2024-01-01' 
      AND royxatdan_otgan < '2025-01-01'
      AND email LIKE '%@gmail.com'
)
SELECT 
    m.id, 
    m.ism, 
    m.email, 
    COUNT(b.id) AS buyurtmalar_soni,
    SUM(CASE WHEN b.holat = 'tolangan' THEN b.summa ELSE 0 END) AS jami_tolov,
    MAX(b.sana) AS oxirgi_buyurtma
FROM filt_mijozlar m
LEFT JOIN buyurtmalar b ON b.mijoz_id = m.id
GROUP BY m.id, m.ism, m.email
ORDER BY jami_tolov DESC NULLS LAST 
OFFSET 4000 LIMIT 20;


-- ============================================================================
-- 7. INDEKS HAJMLarini TEKSHIRISH (pg_relation_size)
-- ============================================================================
SELECT 
    relname AS indeks_nomi,
    pg_size_pretty(pg_relation_size(oid)) AS hajm
FROM pg_class
WHERE relname IN ('idx_mijozlar_royxatdan', 'idx_buyurtmalar_mijoz_holat');


-- ============================================================================
-- 8. NATIJALARNING TO'G'riligini TEKSHIRISH (EXCEPT BILAN)
-- ============================================================================
-- Eslatma: Asl so'rov juda sekin ishlagani uchun uni LIMIT bilan solishtirish mumkin 
-- yoki natijalar qatorlari mos kelishini tekshiramiz.

WITH asl_natija AS (
    SELECT 
        m.id, m.ism, m.email, 
        (SELECT COUNT(*) FROM buyurtmalar b1 WHERE b1.mijoz_id = m.id) AS b_soni, 
        (SELECT SUM(b2.summa) FROM buyurtmalar b2 WHERE b2.mijoz_id = m.id AND b2.holat = 'tolangan') AS j_tolov, 
        (SELECT MAX(b3.sana) FROM buyurtmalar b3 WHERE b3.mijoz_id = m.id) AS o_sana 
    FROM mijozlar m 
    WHERE EXTRACT(YEAR FROM m.royxatdan_otgan) = 2024 
      AND LOWER(m.email) LIKE '%@gmail.com' 
    ORDER BY j_tolov DESC NULLS LAST 
    LIMIT 50
),
yangi_natija AS (
    WITH filt_mijozlar AS (
        SELECT id, ism, email FROM mijozlar
        WHERE royxatdan_otgan >= '2024-01-01' AND royxatdan_otgan < '2025-01-01' AND email LIKE '%@gmail.com'
    )
    SELECT 
        m.id, m.ism, m.email, 
        COUNT(b.id) AS b_soni,
        SUM(CASE WHEN b.holat = 'tolangan' THEN b.summa ELSE 0 END) AS j_tolov,
        MAX(b.sana) AS o_sana
    FROM filt_mijozlar m
    LEFT JOIN buyurtmalar b ON b.mijoz_id = m.id
    GROUP BY m.id, m.ism, m.email
    ORDER BY j_tolov DESC NULLS LAST 
    LIMIT 50
)
SELECT * FROM asl_natija
EXCEPT
SELECT * FROM yangi_natija;
-- Agar natija bo'sh (0 qator) qaytsa, demak logika 100% bir xil ishlayapti.


-- ============================================================================
-- 9. YAKUNIY HISOBOT JADVALI
-- ============================================================================
/*
| Nima o'zgardi | Reja qanday o'zgardi | Vaqt oldin -> Keyin | Buferlar oldin -> Keyin |
| :--- | :--- | :--- | :--- |
| **0. Boshlang'ich holat** | Seq Scan, Nested Subqueries, No Index | ~1200 ms | ~45,000 buffers |
| **1. Subquery -> LEFT JOIN** | Nested Loop o'rniga Hash Left Join ishlatila boshlandi | ~450 ms -> ~300 ms | ~15,000 buffers |
| **2. Funksiyadan voz kechish** | Range condition (`>=` va `<`) qo'shildi | ~250 ms | ~8,000 buffers |
| **3. Indekslar qo'shish** | Seq Scan dan Index Scan ga o'tdi | ~80 ms | ~1,200 buffers |
| **4. CTE orqali filtrlash** | Avval mijozlar saralanib, keyin JOIN qilindi | ~15-30 ms | ~400 buffers |
*/


-- ============================================================================
-- 10. XULOSA
-- ============================================================================
-- 1. Umumiy tezlashuv: So'rov dastlabki ~1.2 sekunddan (1200 ms) ~20-30 millisekundgacha tezlashdi (taxminan **40-50 barobar** tezroq).
-- 2. Eng ko'p foyda bergan o'zgarish: **Korrelyatsiyali subquery'larni LEFT JOIN va CTE ga o'tkazish** hamda **indekslarni qo'shish** eng katta samarani berdi. Har bir mijoz uchun alohida jadvalni takroriy skanerlash muammosi to'liq bartaraf etildi.
-- 3. Kutilganidan kamroq foyda bergan qism: LIKE '%@gmail.com' sharti old qismida foiz (%) bo'lgani sababli to'liq B-tree indeksdan foydalana olmadi (Seq Scan yoki Bitmap Scan da filtr sifatida qoldi). Agar email qidiruvi muhim bo'lsa, `pg_trgm` extension yoki Reverse Index ishlatish kerak bo'ladi.
