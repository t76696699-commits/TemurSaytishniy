-- =====================================================================
-- AMALIY TOPSHIRIQ: Mijozlar va buyurtmalar tahlili, subquery, JOIN va EXPLAIN
-- Texnologiya: PostgreSQL
-- =====================================================================

-- 0. Eski jadvallarni tozalash
DROP TABLE IF EXISTS buyurtmalar CASCADE;
DROP TABLE IF EXISTS mijozlar CASCADE;

-- 1. Sxemani yaratish
CREATE TABLE mijozlar (
    id SERIAL PRIMARY KEY,
    ism VARCHAR(100) NOT NULL
);

CREATE TABLE buyurtmalar (
    id SERIAL PRIMARY KEY,
    mijoz_id INT REFERENCES mijozlar(id), -- (1-b) talab: NULL qiymatlarga ruxsat beriladi (mehmon buyurtmasi)
    status VARCHAR(50) NOT NULL, -- 'yakunlangan', 'jarayonda', 'bekor_qilingan'
    summa NUMERIC(10, 2) NOT NULL
);

-- Ma'lumotlarni kiritish (Talablarga muvofiq: 
-- (a) bitta mijozda 2 tadan ortiq yakunlangan buyurtma, 
-- (b) buyurtmalarda NULL mijoz_id, 
-- (c) hech qachon buyurtma bermagan mijoz mavjud)
INSERT INTO mijozlar (id, ism) VALUES
(1, 'Anvar Karimov'),
(2, 'Malika Rahimova'),
(3, 'Jasurbek Tursunov'), -- (1-c) talab: Hech qachon buyurtma bermagan mijoz
(4, 'Zilola Umarova');

INSERT INTO buyurtmalar (mijoz_id, status, summa) VALUES
-- Anvar Karimov (id=1): 3 ta yakunlangan buyurtma ((1-a) talab bajarildi)
(1, 'yakunlangan', 1500.00),
(1, 'yakunlangan', 2300.00),
(1, 'yakunlangan', 1200.00),
-- Malika Rahimova (id=2): Aralash statuslar
(2, 'yakunlangan', 800.00),
(2, 'jarayonda', 400.00),
-- Mehmon buyurtmasi (mijoz_id NULL) -> (1-b) talab bajarildi
(NULL, 'yakunlangan', 5000.00),
(NULL, 'jarayonda', 300.00);


-- =====================================================================
-- 2, 3-TALABLAR: (A) savoli uchun uch xil yozuv va JOIN muammolari
-- =====================================================================

-- 1-usul: IN-subquery
SELECT id, ism 
FROM mijozlar 
WHERE id IN (
    SELECT mijoz_id 
    FROM buyurtmalar 
    WHERE status = 'yakunlangan' AND mijoz_id IS NOT NULL
);

-- 2-usul: EXISTS
SELECT m.id, m.ism 
FROM mijozlar m
WHERE EXISTS (
    SELECT 1 
    FROM buyurtmalar b 
    WHERE b.mijoz_id = m.id AND b.status = 'yakunlangan'
);

-- 3-usul: JOIN (Izoh: Bu usul takroriy qatorlar beradi, chunki bitta mijozning 
-- bir nechta 'yakunlangan' buyurtmasi bo'lsa, mijoz nomi har bir buyurtma uchun takroran chiqadi).
SELECT DISTINCT m.id, m.ism 
FROM mijozlar m
JOIN buyurtmalar b ON m.id = b.mijoz_id
WHERE b.status = 'yakunlangan';

/*
  IZOH (3-talab): 
  DISTINCT operatori takroriy qatorlarni shunchaki vizual ravishda yashiradi (olib tashlaydi), 
  lekin u asl muammoni — ortiqcha birikish (join) va skanerlash jarayonini hal qilmaydi. 
  Produksiyada katta jadvallar uchun DISTINCT ishlash tezligini sekinlashtiradi. 
  Shuning uchun bu holatda EXISTS yoki IN afzalroq.
*/


-- =====================================================================
-- 4, 5-TALABLAR: (B) savoli – Hech qachon buyurtma bermaganlar va NOT IN xatosi
-- =====================================================================

-- 4-talab: Xato ishlaydigan NOT IN varianti (Bo'sh natija qaytaradi)
-- SABAB (Uch qiymatli mantiq - Three-valued logic): 
-- Ichki so'rov `NULL` qiymatni qaytarganda, SQL dagi shart `id NOT IN (1, 2, NULL)` ko'rinishiga keladi.
-- Bu `NOT (id = 1 OR id = 2 OR id = NULL)` deganidir. SQL da `id = NULL` natijasi `UNKNOWN` (noma'lum) 
-- beradi, `NOT (TRUE OR UNKNOWN)` esa `FALSE` ga tenglashadi. Natijada hech qanday qator topilmaydi!
SELECT * FROM mijozlar 
WHERE id NOT IN (SELECT mijoz_id FROM buyurtmalar); -- Bo'sh natija qaytaradi!


-- 5-talab: B savolining ikkita TO'G'RI varianti

-- Variant 1: NOT EXISTS
SELECT * FROM mijozlar m
WHERE NOT EXISTS (
    SELECT 1 FROM buyurtmalar b WHERE b.mijoz_id = m.id
);

-- Variant 2: LEFT JOIN ... IS NULL (Anti-join)
SELECT m.* FROM mijozlar m
LEFT JOIN buyurtmalar b ON m.id = b.mijoz_id
WHERE b.id IS NULL;


-- =====================================================================
-- 6, 7-TALABLAR: Ko'rsatkichlarni hisoblash va EXPLAIN ANALYZE taqqosi
-- =====================================================================

-- 1-usul: Korrelyatsiyali subquery orqali
SELECT 
    m.ism,
    (SELECT COUNT(*) FROM buyurtmalar b WHERE b.mijoz_id = m.id) AS buyurtmalar_soni,
    (SELECT COALESCE(SUM(summa), 0) FROM buyurtmalar b WHERE b.mijoz_id = m.id) AS jami_summa,
    (SELECT COUNT(*) FROM buyurtmalar b WHERE b.mijoz_id = m.id AND b.status = 'yakunlangan') AS yakunlanganlar_soni
FROM mijozlar m;

-- 2-usul: Bitta LEFT JOIN + GROUP BY + FILTER orqali
SELECT 
    m.ism,
    COUNT(b.id) AS buyurtmalar_soni,
    COALESCE(SUM(b.summa), 0) AS jami_summa,
    COUNT(b.id) FILTER (WHERE b.status = 'yakunlangan') AS yakunlanganlar_soni
FROM mijozlar m
LEFT JOIN buyurtmalar b ON m.id = b.mijoz_id
GROUP BY m.id, m.ism;

-- 7-talab uchun EXPLAIN ANALYZE tahlili:
-- Korrelyatsiyali subquery yondashuvi har bir mijoz uchun alohida ichki so'rovni takroran 
-- ishga tushiradi (Nested Loop / Sequential Scan ko'p marta bajariladi).
EXPLAIN ANALYZE
SELECT 
    m.ism,
    (SELECT COUNT(*) FROM buyurtmalar b WHERE b.mijoz_id = m.id) AS b_soni
FROM mijozlar m;

-- LEFT JOIN + GROUP BY yondashuvi esa jadvallarni bir marta birlashtirib, 
-- bitta skanerlash (Sequential Scan) va guruhlash orqali natija beradi, 
-- bu esa katta jadvallarda ancha samarali ishlaydi.
EXPLAIN ANALYZE
SELECT 
    m.ism,
    COUNT(b.id) AS b_soni
FROM mijozlar m
LEFT JOIN buyurtmalar b ON m.id = b.mijoz_id
GROUP BY m.id, m.ism;


-- =====================================================================
-- 8-TALAB: EXISTS va IN variantlarining EXPLAIN (COSTS OFF) rejalari solishtiruvi
-- =====================================================================
EXPLAIN (COSTS OFF)
SELECT * FROM mijozlar m WHERE id IN (SELECT mijoz_id FROM buyurtmalar WHERE status = 'yakunlangan');

EXPLAIN (COSTS OFF)
SELECT * FROM mijozlar m WHERE EXISTS (SELECT 1 FROM buyurtmalar b WHERE b.mijoz_id = m.id AND b.status = 'yakunlangan');

/*
  XULOSA (8-talab): 
  Kichik jadvallarda `IN` va `EXISTS` rejalari bir xil ko'rinishi mumkin, biroq indekslangan 
  va katta jadvallarda `EXISTS` qidirilayotgan shart topilishi bilan qidiruvni to'xtatgani (short-circuit) 
  uchun ko'pincha `Semi Join` rejasida tezroq ishlaydi. `IN` esa ba'zi eski PostgreSQL versiyalarida 
  ichki ro'yxatni to'liq shakllantirib olgach ishlar edi.
*/


-- =====================================================================
-- 9-TALAB: Qaysi vazifaga qaysi vosita va nega (Jadval-izoh)
-- =====================================================================
/*
+-----------------------------------+--------------------------------+-------------------------------------------------------------+
| Vazifa turi                       | Tavsiya etiladigan vosita      | Nega aynan bu vosita?                                       |
+-----------------------------------+--------------------------------+-------------------------------------------------------------+
| Mavjudlikni tekshirish (A savoli) | EXISTS                         | Shart bajarilishi bilan qidiruvni to'xtatadi, tezkor ishlaydi.|
| Yo'qligini tekshirish (B savoli)  | NOT EXISTS / LEFT JOIN...IS NULL| Uch qiymatli mantiqiy xatolardan (NULL) qochishni ta'minlaydi.|
| Agregatsiya va statistika olish   | LEFT JOIN + GROUP BY + FILTER  | Ma'lumotlarni bir martalik skanerlash va guruhlash orqali     |
|                                   |                                | samarali hisob-kitob qiladi.                                |
| Murakkab bosqichma-bosqich tahlil | CTE (WITH)                     | Kodni o'qishni osonlashtiradi va qismlarga ajratib tahlil     |
|                                   |                                | qilishga imkon beradi.                                      |
+-----------------------------------+--------------------------------+-------------------------------------------------------------+
*/


-- =====================================================================
-- 10-TALAB: Skript xatosiz bajarilib yakunlandi.
-- =====================================================================
