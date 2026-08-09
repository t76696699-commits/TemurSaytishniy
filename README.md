-- ============================================================================
-- AMALIY TOPSHIRIQ: HODISALAR JADVALINI BO'LISH VA ARXIVLASH STRATEGIYASI
-- Texnologiya: PostgreSQL
-- ============================================================================


-- ============================================================================
-- 1. YOZMA QAROR (STRATEGIYA TANLOVI VA ASOSLASH)
-- ============================================================================
/*
QAROR: RANGE bo'yicha bo'lish (Partitioning) tanlanadi.
ASOS:
- Telemetriya jurnali va vaqt qatorlari (time-series) ma'lumotlari uchun vaqt/sana (timestamp) 
  eng tabiiy o'lcham hisoblanadi. So'rovlarning 95% i oxirgi 30 kunlik sana diapazoni bo'yicha 
  filtrlashini hisobga olsak, RANGE partition PostgreSQL'ga kerak emas bo'limlarni butunlay 
  chetlab o'tishga (Partition Pruning) imkon beradi. 12 oydan eski ma'lumotlarni oylik 
  bo'limlarga ajratib, ularni osongina DETACH qilish va arxivlash mumkin.

QOLGAN IKKI STRATEGIYA NEGA MOS EMAS?
1. LIST: Agar ma'lumotlar aniq bir kategoriya (masalan, mamlakat, status yoki tenant_id) 
   bo'yicha guruhlanganda mos kelardi. Vaqt bo'yicha uzluksiz o'sib boruvchi telemetriya 
   jurnali uchun LIST bo'yicha bo'lish mantiqsiz, chunki har bir vaqt oralig'i uchun yangi 
   qiymatlarni oldindan bashorat qilib bo'lmaydi va boshqarish qiyinlashadi.
2. HASH: Qatorlarni bir xilda taqsimlash (load balancing) uchun ishlatiladi. HASH bo'yicha 
   bo'lingan jadvalda ma'lum bir sana oralig'idagi ma'lumotlarni qidirish yoki eski oylik 
   bo'limlarni DETACH qilish imkonsiz, chunki bitta vaqtdagi ma'lumotlar barcha hash 
   bo'limlarga tarqalib ketadi.

PARTITIONING UMMAN KERAK BO'LMAYDIGAN VAZIYAT:
- Agar jadvaldagi umumiy qatorlar soni kichik bo'lsa (masalan, bir necha yuz ming yoki 
  hatto bir necha million qator ham to'g'ri indekslangan bo'lsa bitta jadvalda sekundning 
  ulushida ishlaydi), yoki ma'lumotlar hech qachon o'chirilmasa/arxivlanmasa, partitioning 
  qo'shimcha murakkablik keltirib chiqaradi va shunchaki to'g'ri B-tree indeks (masalan, 
  sana bo'yicha) hamda vaqti-vaqti bilan eski ma'lumotlarni arxivlash skriptlari yetarli bo'lardi.
*/


-- ============================================================================
-- 2. RANGE BO'YICHA BO'LINGAN JADVALNI YARATISH
-- ============================================================================
DROP TABLE IF EXISTS telemetriya CASCADE;

-- Asosiy (parent) jadval. PostgreSQL talabiga ko'ra, PRIMARY KEY ga partition kaliti (sana) kiritilishi shart.
CREATE TABLE telemetriya (
    id SERIAL,
    sana TIMESTAMP NOT NULL,
    qurilma_id INT,
    voqea_turi VARCHAR(100),
    qiymat NUMERIC,
    PRIMARY KEY (id, sana)
) PARTITION BY RANGE (sana);

-- Oylik bo'limlar (Partitions) va DEFAULT bo'lim
CREATE TABLE telemetriya_default PARTITION OF telemetriya DEFAULT;

CREATE TABLE telemetriya_2026_05 PARTITION OF telemetriya 
    FOR VALUES FROM ('2026-05-01 00:00:00') TO ('2026-06-01 00:00:00');

CREATE TABLE telemetriya_2026_06 PARTITION OF telemetriya 
    FOR VALUES FROM ('2026-06-01 00:00:00') TO ('2026-07-01 00:00:00');

CREATE TABLE telemetriya_2026_07 PARTITION OF telemetriya 
    FOR VALUES FROM ('2026-07-01 00:00:00') TO ('2026-08-01 00:00:00');

CREATE TABLE telemetriya_2026_08 PARTITION OF telemetriya 
    FOR VALUES FROM ('2026-08-01 00:00:00') TO ('2026-09-01 00:00:00');


-- ============================================================================
-- 3. TEST MA'LUMOTLARINI YUKLASH VA TAQSIMOTNI TEKSHIRISH
-- ============================================================================
-- generate_series yordamida 200 000 dan ortiq qator kiritamiz
INSERT INTO telemetriya (sana, qurilma_id, voqea_turi, qiymat)
SELECT 
    '2026-05-01 00:00:00'::timestamp + (random() * INTERVAL '120 days'),
    (random() * 1000 + 1)::INT,
    CASE (random() * 3)::INT 
        WHEN 0 THEN 'click' 
        WHEN 1 THEN 'scroll' 
        ELSE 'error' 
    END,
    random() * 100
FROM generate_series(1, 250000) AS i;

-- Qatorlarning bo'limlar bo'yicha taqsimotini tableoid orqali ko'rish
SELECT 
    tableoid::regclass AS bolim_nomi,
    COUNT(*) AS qatorlar_soni
FROM telemetriya
GROUP BY tableoid
ORDER BY bolim_nomi;


-- ============================================================================
-- 4. INDEKS YARATISH
-- ============================================================================
/*
Izoh: Ona jadvalda (parent table) yaratilgan indeks PostgreSQL tomonidan avtomatik 
ravishda barcha mavjud va keyinchalik yaratiladigan bo'limlarga (partitions) tarqatiladi.
*/
CREATE INDEX idx_telemetriya_sana ON telemetriya (sana);


-- ============================================================================
-- 5. PRUNING ISHLAGAN HOLAT
-- ============================================================================
-- Filtr to'g'ridan-to'g'ri partition kalitiga qo'yilganda, EXPLAIN rejasida 
-- faqat bitta mos bo'lim (Append ostida bitta node) qatnashishini ko'ramiz.
EXPLAIN (FORMAT TEXT)
SELECT * FROM telemetriya 
WHERE sana >= '2026-06-01 00:00:00' AND sana < '2026-07-01 00:00:00';


-- ============================================================================
-- 6. PRUNING BUZILGAN HOLAT VA UNI TUZATISH
-- ============================================================================
-- BUZILGAN HOLAT: Kalitga funksiya qo'llanganda (EXTRACT), optimizator qaysi 
-- bo'limga tegishli ekanini oldindan bilolmaydi va BARCHA bo'limlarni skanerlaydi.
EXPLAIN (FORMAT TEXT)
SELECT * FROM telemetriya 
WHERE EXTRACT(MONTH FROM sana) = 6 AND EXTRACT(YEAR FROM sana) = 2026;

-- TUZATILGAN VARIANT: Funksiyadan voz kechib, to'g'ridan-to'g'ri sana oralig'i (Range) ishlatiladi.
EXPLAIN (FORMAT TEXT)
SELECT * FROM telemetriya 
WHERE sana >= '2026-06-01 00:00:00' AND sana < '2026-07-01 00:00:00';


-- ============================================================================
-- 7. DEFAULT BO'LIMSIZ INSERT Xatosi
-- ============================================================================
/*
Izoh: Agar bo'limlar orasiga tushmaydigan sana qiymati kiritilsa va DEFAULT bo'lim 
mavjud bo'lmasa, quyidagi kabi xato yuzaga keladi:
"ERROR: no partition of relation "telemetriya" found for row"
(Buni sinab ko'rish uchun telemetriya_default ni vaqtincha o'chirib, 
oraliqqa kirmaydigan sanali qator kiritish kifoya).
*/


-- ============================================================================
-- 8. ARXIVLASH: DETACH VA ATTACH
-- ============================================================================
-- Ajratishdan oldingi umumiy qator soni
SELECT COUNT(*) AS ajratishdan_oldin FROM telemetriya;

-- Eng eski bo'limni (2026-yil May) ajratib olish (Detach)
ALTER TABLE telemetriya DETACH PARTITION telemetriya_2026_05;

-- Ajratishdan keyingi ona jadvaldagi va alohida olingan arxiv jadvalidagi qator sonlari
SELECT 'Ona jadval' AS qism, COUNT(*) AS qator_soni FROM telemetriya
UNION ALL
SELECT 'Arxiv jadval (telemetriya_2026_05)' AS qism, COUNT(*) AS qator_soni FROM telemetriya_2026_05;

-- Ma'lumot yo'qolmagani isbotlandi (ikkala jadval yig'indisi asl qatorlar soniga teng).
-- Arxivdan qayta ulash (Attach):
ALTER TABLE telemetriya ATTACH PARTITION telemetriya_2026_05 
    FOR VALUES FROM ('2026-05-01 00:00:00') TO ('2026-06-01 00:00:00');


-- ============================================================================
-- 9. UNIQUE CONSTRAINT CHEKlovi
-- ============================================================================
/*
XATO: PostgreSQL'da partitioned jabvallarda UNIQUE yoki PRIMARY KEY cheklovlari 
albatta partition kalitini (bu yerda 'sana') o'z ichiga olishi SHART. 
Aks holda quyidagi xato kelib chiqadi:
"ERROR: unique constraint on partitioned table must include all partitioning columns"
*/
-- Xato urinish namunasi (agar ishga tushirilsa xato beradi):
-- ALTER TABLE telemetriya ADD CONSTRAINT telemetriya_id_uq UNIQUE (id);

-- TO'G'RI VARIANT (Partition kaliti bilan birga kiritish):
-- ALTER TABLE telemetriya ADD CONSTRAINT telemetriya_id_sana_uq UNIQUE (id, sana);


-- ============================================================================
-- 10. YAKUNIY SAVol (BIR JUMLADA)
-- ============================================================================
-- Agar bo'lish LIST bo'yicha (masalan ijarachi/tenant) bo'lganida, sxemada PARTITION BY LIST (tenant_id) 
-- ishlatilib har bir tenant uchun alohida bo'limlar ochilar edi, so'rovlarda эса sana o'rniga 
-- WHERE tenant_id = X sharti ishlatilib pruning o'sha tenant bo'limiga yo'naltirilar edi.
