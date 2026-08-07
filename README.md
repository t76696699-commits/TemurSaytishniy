-- =========================================================================
-- SHAHAR KUTUBXONA TIZIMI - TO'LIQ NORMALLASHTIRILGAN SQL SXEMASI
-- Texnologiya: PostgreSQL
-- =========================================================================

/*
   --- NORMALIZATSIYA BO'YICHA QISQACHA TUSHUNTURISh (BONUS) ---
   1. 1NF (Birinchi Normal Forma): 
      - Barcha ustunlar atomar (bo'linmas) qiymatlarga ega (masalan, ism va familiya alohida).
      - Har bir jadvalda takrorlanuvchi guruhlar mavjud emas.
   2. 2NF (Ikkinchi Normal Forma):
      - 1NF talablari bajarilgan va barcha non-key ustunlar to'liq PRIMARY KEY ga funksional bog'langan (masalan, "kitob_mualliflari" jadvalidagi kompozit kalit ikkala ustunga ham bog'liq).
   3. 3NF (Uchinchi Normal Forma):
      - 2NF talablari bajarilgan va tranzitiv bog'liqliklar yo'q (masalan, "azo_profillari" jadvalidagi pasport va manzil to'g'ridan-to'g'ri `azo_id` ga bog'langan, boshqa no-kalit ustunga emas).
*/

-- Jadvallarni tozalash (agar avval yaratilgan bo'lsa)
DROP TABLE IF EXISTS qarzlar CASCADE;
DROP TABLE IF EXISTS azo_profillari CASCADE;
DROP TABLE IF EXISTS azolar CASCADE;
DROP TABLE IF EXISTS nusxalar CASCADE;
DROP TABLE IF EXISTS kitob_mualliflari CASCADE;
DROP TABLE IF EXISTS kitoblar CASCADE;
DROP TABLE IF EXISTS mualliflar CASCADE;


-- =========================================================================
-- 1. MUALLIFLAR JADVALI (1NF, 2NF, 3NF)
-- =========================================================================
CREATE TABLE mualliflar (
    id SERIAL PRIMARY KEY,
    ism VARCHAR(100) NOT NULL,
    familiya VARCHAR(100) NOT NULL,
    tugilgan_yil INT CHECK (tugilgan_yil > 1000 AND tugilgan_yil <= EXTRACT(YEAR FROM CURRENT_DATE))
);


-- =========================================================================
-- 2. KITOBLAR JADVALI (Asarlar)
-- =========================================================================
CREATE TABLE kitoblar (
    id SERIAL PRIMARY KEY,
    nomi VARCHAR(255) NOT NULL,
    nashr_yili INT CHECK (nashr_yili > 1000 AND nashr_yili <= EXTRACT(YEAR FROM CURRENT_DATE)),
    janr VARCHAR(100) NOT NULL
);


-- =========================================================================
-- 3. KITOBLAR VA MUALLIFLAR (N:N Munosabat uchun Junction jadval)
-- =========================================================================
CREATE TABLE kitob_mualliflari (
    kitob_id INT NOT NULL,
    muallif_id INT NOT NULL,
    -- Kompozit PRIMARY KEY
    PRIMARY KEY (kitob_id, muallif_id),
    -- FK strategiyalari: Kitob yoki muallif o'chirilsa, bog'lanish ham o'chiriladi
    CONSTRAINT fk_km_kitob FOREIGN KEY (kitob_id) REFERENCES kitoblar(id) ON DELETE CASCADE,
    CONSTRAINT fk_km_muallif FOREIGN KEY (muallif_id) REFERENCES mualliflar(id) ON DELETE CASCADE
);


-- =========================================================================
-- 4. NUSXALAR JADVALI (1:N Munosabat: bitta kitobning bir nechta jismoniy nusxasi)
-- =========================================================================
CREATE TABLE nusxalar (
    id SERIAL PRIMARY KEY,
    kitob_id INT NOT NULL,
    inventar_raqami VARCHAR(50) UNIQUE NOT NULL,
    holati VARCHAR(50) DEFAULT 'mavjud' CHECK (holati IN ('mavjud', 'qo\'lda', ta'mirda)),
    -- FK strategiyasi: Kitob o'chirilsa uning jismoniy nusxalari ham o'chiriladi
    CONSTRAINT fk_nusxa_kitob FOREIGN KEY (kitob_id) REFERENCES kitoblar(id) ON DELETE CASCADE
);


-- =========================================================================
-- 5. AZOLAR JADVALI
-- =========================================================================
CREATE TABLE azolar (
    id SERIAL PRIMARY KEY,
    ism VARCHAR(100) NOT NULL,
    familiya VARCHAR(100) NOT NULL,
    reg_sanasi DATE DEFAULT CURRENT_DATE NOT NULL
);


-- =========================================================================
-- 6. AZO PROFILLARI JADVALI (1:1 Munosabat: azo_id bir vaqtning o'zida PK va FK)
-- =========================================================================
CREATE TABLE azo_profillari (
    azo_id INT PRIMARY KEY,
    pasport_raqami VARCHAR(20) UNIQUE NOT NULL,
    manzil TEXT NOT NULL,
    telefon VARCHAR(20),
    -- FK strategiyasi: A'zo o'chirilsa uning profili ham o'chiriladi
    CONSTRAINT fk_profil_azo FOREIGN KEY (azo_id) REFERENCES azolar(id) ON DELETE CASCADE
);


-- =========================================================================
-- 7. QARZLAR JADVALI (Kitob berish tarixi va holati)
-- =========================================================================
CREATE TABLE qarzlar (
    id SERIAL PRIMARY KEY,
    nusxa_id INT NOT NULL,
    azo_id INT NOT NULL,
    olingan_sana DATE NOT NULL DEFAULT CURRENT_DATE,
    qaytarish_muddati DATE NOT NULL,
    qaytarilgan_sana DATE,
    -- CHECK cheklovlari talabi
    CONSTRAINT chk_muddat CHECK (qaytarish_muddati > olingan_sana),
    CONSTRAINT chk_qaytarilgan CHECK (qaytarilgan_sana IS NULL OR qaytarilgan_sana >= olingan_sana),
    -- FK strategiyalari: Tarix saqlanishi kerakligi uchun a'zo yoki nusxa o'chirilishini cheklaymiz
    CONSTRAINT fk_qarz_nusxa FOREIGN KEY (nusxa_id) REFERENCES nusxalar(id) ON DELETE RESTRICT,
    CONSTRAINT fk_qarz_azo FOREIGN KEY (azo_id) REFERENCES azolar(id) ON DELETE RESTRICT
);

-- MUHIM CHEKLOV: Bir jismoniy nusxa bir vaqtning o'zida faqat bitta odamda bo'lishi mumkin
-- Faqatgina hali qaytarilmagan (qaytarilgan_sana IS NULL) qarzlar uchun nusxa_id takrorlanmasligi kerak
CREATE UNIQUE INDEX idx_unique_active_loan ON qarzlar (nusxa_id) WHERE qaytarilgan_sana IS NULL;


-- =========================================================================
-- TEST MA'LUMOTLARINI KIRITISH
-- =========================================================================

-- 1. Mualliflar (5+ ta)
INSERT INTO mualliflar (id, ism, familiya, tugilgan_yil) VALUES
(1, 'Alisher', 'Navoiy', 1441),
(2, 'Abdulla', 'Qodiriy', 1894),
(3, 'O‘tkir', 'Hoshimov', 1941),
(4, 'Tohir', 'Malik', 1946),
(5, 'Jorj', 'Oruell', 1903);

-- 2. Kitoblar (5+ ta)
INSERT INTO kitoblar (id, nomi, nashr_yili, janr) VALUES
(1, 'Xamsa', 2015, 'Doston'),
(2, 'O‘tkan kunlar', 2018, 'Roman'),
(3, 'Ikki eshik orasi', 2019, 'Roman'),
(4, 'Shaytanat', 2020, 'Detektiv'),
(5, '1984', 2021, 'Fantastika'),
(6, 'Jimjitlik', 2017, 'Roman'); -- Hech qachon olinmagan kitob (5-hisobot uchun)

-- 3. Kitob va mualliflar bog'liqligi (N:N)
INSERT INTO kitob_mualliflari (kitob_id, muallif_id) VALUES
(1, 1),
(2, 2),
(3, 3),
(4, 4),
(5, 5),
(6, 3);

-- 4. Nusxalar (10+ ta)
INSERT INTO nusxalar (id, kitob_id, inventar_raqami, holati) VALUES
(1, 1, 'INV-001', 'mavjud'),
(2, 1, 'INV-002', 'qo\'lda'),
(3, 2, 'INV-003', 'qo\'lda'),
(4, 2, 'INV-004', 'mavjud'),
(5, 3, 'INV-005', 'qo\'lda'),
(6, 3, 'INV-006', 'mavjud'),
(7, 4, 'INV-007', 'qo\'lda'),
(8, 4, 'INV-008', 'mavjud'),
(9, 5, 'INV-009', 'qo\'lda'),
(10, 5, 'INV-010', 'mavjud'),
(11, 6, 'INV-011', 'mavjud');

-- 5. Azolar (5+ ta)
INSERT INTO azolar (id, ism, familiya, reg_sanasi) VALUES
(1, 'Jasur', 'Karimov', '2025-01-10'),
(2, 'Malika', 'Sobirova', '2025-02-15'),
(3, 'Bekzod', 'Tursunov', '2025-03-01'),
(4, 'Zarnigor', 'Aliyeva', '2025-03-10'),
(5, 'Sardor', 'Rahimov', '2025-04-05');

-- 6. Azo profillari (1:1)
INSERT INTO azo_profillari (azo_id, pasport_raqami, manzil, telefon) VALUES
(1, 'AB1234567', 'Toshkent sh., Chilonzor 9', '+998901112233'),
(2, 'AC7654321', 'Toshkent sh., Yunusobod 4', '+998912223344'),
(3, 'AD9876543', 'Samarqand sh., Registon ko\'chasi 12', '+998933334455'),
(4, 'AE4567890', 'Buxoro sh., M.Iqbol 5', '+998944445566'),
(5, 'AF1122334', 'Farg\'ona sh., Al-Farg\'oniy 1', '+998955556677');

-- 7. Qarzlar (8+ ta, ba'zilari hozir qo'lda, ba'zilari muddati o'tgan)
INSERT INTO qarzlar (nusxa_id, azo_id, olingan_sana, qaytarish_muddati, qaytarilgan_sana) VALUES
-- Oldingi yopilgan qarzlar
(1, 1, '2026-01-05', '2026-01-20', '2026-01-15'),
(4, 2, '2026-01-10', '2026-01-25', '2026-01-22'),
-- Hozir qo'lda bo'lganlar (qaytarilgan_sana IS NULL)
(2, 1, '2026-02-01', '2026-02-28', NULL),
(3, 2, '2026-02-05', '2026-02-20', NULL), -- Muddati o'tgan bo'lishi mumkin (joriy sanaga qarab)
(5, 3, '2026-02-10', '2026-03-05', NULL),
(7, 4, '2026-02-12', '2026-03-10', NULL),
(9, 5, '2026-01-15', '2026-02-01', NULL), -- Muddati o'tgan aniq
(6, 3, '2026-01-01', '2026-01-15', '2026-01-14'),
(8, 5, '2026-01-02', '2026-01-20', '2026-01-18');


-- =========================================================================
-- 5 TA HISOBOT (QUERIES)
-- =========================================================================

-- 1. Hozir qo'lda bo'lgan nusxalar (qaytarilmaganlar)
SELECT 
    q.id AS qarz_id,
    k.nomi AS kitob_nomi,
    n.inventar_raqami,
    CONCAT(a.ism, ' ', a.familiya) AS azo_f.i.sh,
    q.olingan_sana,
    q.qaytarish_muddati
FROM qarzlar q
JOIN nusxalar n ON q.nusxa_id = n.id
JOIN kitoblar k ON n.kitob_id = k.id
JOIN azolar a ON q.azo_id = a.id
WHERE q.qaytarilgan_sana IS NULL;

-- 2. Muddati o'tgan qarzlar (hali qaytarilmagan va muddati o'tganlar)
SELECT 
    q.id AS qarz_id,
    k.nomi AS kitob_nomi,
    n.inventar_raqami,
    CONCAT(a.ism, ' ', a.familiya) AS azo_f.i.sh,
    ap.telefon,
    q.olingan_sana,
    q.qaytarish_muddati,
    CURRENT_DATE - q.qaytarish_muddati AS kechikkan_kunlar_soni
FROM qarzlar q
JOIN nusxalar n ON q.nusxa_id = n.id
JOIN kitoblar k ON n.kitob_id = k.id
JOIN azolar a ON q.azo_id = a.id
JOIN azo_profillari ap ON a.id = ap.azo_id
WHERE q.qaytarilgan_sana IS NULL 
  AND q.qaytarish_muddati < CURRENT_DATE;

-- 3. Har asar bo'yicha jami va bo'sh (mavjud) nusxalar soni
SELECT 
    k.id AS kitob_id,
    k.nomi,
    COUNT(n.id) AS jami_nusxalar,
    SUM(CASE WHEN n.holati = 'mavjud' THEN 1 ELSE 0 END) AS bosh_nusxalar_soni
FROM kitoblar k
LEFT JOIN nusxalar n ON k.id = n.kitob_id
GROUP BY k.id, k.nomi;

-- 4. Eng faol 3 a'zo (Eng ko'p kitob olganlar reytingi)
SELECT 
    a.id AS azo_id,
    CONCAT(a.ism, ' ', a.familiya) AS f.i.sh,
    COUNT(q.id) AS olingan_kitoblar_soni
FROM azolar a
JOIN qarzlar q ON a.id = q.azo_id
GROUP BY a.id, a.ism, a.familiya
ORDER BY olingan_kitoblar_soni DESC
LIMIT 3;

-- 5. Hech qachon olinmagan asarlar
SELECT 
    k.id AS kitob_id,
    k.nomi,
    k.nashr_yili,
    k.janr
FROM kitoblar k
LEFT JOIN nusxalar n ON k.id = n.kitob_id
LEFT JOIN qarzlar q ON n.id = q.nusxa_id
WHERE q.id IS NULL;
