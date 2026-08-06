-- =====================================================================
-- AMALIY LOYIHA: Sotuvchilar samaradorligi hisoboti
-- Texnologiya: PostgreSQL, Window Functions, CTE, generate_series
-- =====================================================================

-- 0. Eski jadvallarni tozalash (agar mavjud bo'lsa)
DROP TABLE IF EXISTS sotuv_qatorlari CASCADE;
DROP TABLE IF EXISTS sotuvlar CASCADE;
DROP TABLE IF EXISTS mahsulotlar CASCADE;
DROP TABLE IF EXISTS sotuvchilar CASCADE;

-- =====================================================================
-- 1. SXEMANI YARATISH
-- =====================================================================

-- Sotuvchilar jadvali
CREATE TABLE sotuvchilar (
    id SERIAL PRIMARY KEY,
    ism VARCHAR(100) NOT NULL,
    hudud VARCHAR(50) NOT NULL
);

-- Mahsulotlar jadvali
CREATE TABLE mahsulotlar (
    id SERIAL PRIMARY KEY,
    nomi VARCHAR(100) NOT NULL,
    narx NUMERIC(10, 2) NOT NULL
);

-- Sotuvlar (cheklar) jadvali
CREATE TABLE sotuvlar (
    id SERIAL PRIMARY KEY,
    sotuvchi_id INT REFERENCES sotuvchilar(id),
    sana DATE NOT NULL
);

-- Sotuv qatorlari (chek tarkibi)
CREATE TABLE sotuv_qatorlari (
    id SERIAL PRIMARY KEY,
    sotuv_id INT REFERENCES sotuvlar(id),
    mahsulot_id INT REFERENCES mahsulotlar(id),
    miqdor INT NOT NULL
);

-- =====================================================================
-- 2. TEST MA'LUMOTLARINI GENERATSIYA QILISH (10 000+ qator)
-- =====================================================================

-- Sotuvchilarni kiritish (Har bir hududda bir nechta sotuvchi)
INSERT INTO sotuvchilar (ism, hudud) VALUES
('Anvar Karimov', 'Toshkent'),
('Malika Rahimova', 'Toshkent'),
('Jasur Tursunov', 'Samarqand'),
('Zilola Umarova', 'Samarqand'),
('Bekzod Bekov', 'Fargona'),
('Nodira Sharipova', 'Fargona');

-- Mahsulotlarni kiritish
INSERT INTO mahsulotlar (nomi, narx) VALUES
('Smartfon', 3500.00),
('Noutbuk', 7500.00),
('Quloqchin', 300.00),
('Smart soat', 1200.00),
('Planshet', 4000.00);

-- Sotuvlar va sotuv qatorlarini generate_series orqali 10 000+ qator qilib yaratish
-- 1. Sotuvlar jadvalini to'ldirish (~2500 ta chek)
INSERT INTO sotuvlar (sotuvchi_id, sana)
SELECT 
    (1 + (i % 6)) AS sotuvchi_id,
    ('2025-01-01'::DATE + (i % 365) * INTERVAL '1 day')::DATE AS sana
FROM generate_series(1, 2500) AS i;

-- 2. Sotuv qatorlari jadvalini to'ldirish (~12 500 ta qator)
INSERT INTO sotuv_qatorlari (sotuv_id, mahsulot_id, miqdor)
SELECT 
    (1 + (i % 2500)) AS sotuv_id,
    (1 + (i % 5)) AS mahsulot_id,
    (1 + (i % 4)) AS miqdor
FROM generate_series(1, 12500) AS i;


-- =====================================================================
-- 3-9. TAHLILIY HISOBOT VA WINDOW FUNKSIYALAR
-- =====================================================================

/*
  IZOH (8-talab): 
  Skript quyidagi ikki bosqichli CTE tuzilmasiga ega:
  1. `monthly_sales` (Tayyorlash bosqichi): Xom ma'lumotlarni oylik kesimda guruhlab, 
     har bir sotuvchining oylik tushumini hisoblaydi.
  2. `enriched_report` (Boyitish bosqichi): Window funksiyalar yordamida oylik o'zgarish (MoM), 
     jamlanma tushum, hududdagi o'rin va ulushlarni hisoblaydi.
*/

WITH monthly_sales AS (
    -- 1-bosqich: Tayyorlash - har bir sotuvchi uchun oylik tushumni hisoblash
    SELECT 
        s.id AS sotuvchi_id,
        s.ism,
        s.hudud,
        DATE_TRUNC('month', sv.sana)::DATE AS oy,
        SUM(sq.miqdor * m.narx) AS oylik_tushum
    FROM sotuvchilar s
    JOIN sotuvlar sv ON s.id = sv.sotuvchi_id
    JOIN sotuv_qatorlari sq ON sv.id = sq.sotuv_id
    JOIN mahsulotlar m ON sq.mahsulot_id = m.id
    GROUP BY s.id, s.ism, s.hudud, DATE_TRUNC('month', sv.sana)
),
enriched_report AS (
    -- 2-bosqich: Boyitish - Window funksiyalarni qo'llash
    SELECT 
        sotuvchi_id,
        ism,
        hudud,
        oy,
        oylik_tushum,
        
        -- 3-talab: LAG va MoM foiz o'zgarish (NULLIF bilan nolga bo'linishdan himoya)
        LAG(oylik_tushum) OVER w_sotuvchi_vaqt AS otgan_oy_tushumi,
        ROUND(
            ((oylik_tushum - LAG(oylik_tushum) OVER w_sotuvchi_vaqt) * 100.0) / 
            NULLIF(LAG(oylik_tushum) OVER w_sotuvchi_vaqt, 0), 2
        ) AS mom_foiz_ozgarish,
        
        -- 4-talab: Jamlanma tushum (ROWS freymi OSHKORA yozildi)
        SUM(oylik_tushum) OVER (
            PARTITION BY sotuvchi_id 
            ORDER BY oy 
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS yil_boshidan_jamlanma,
        
        -- 5-talab: Hudud tushumidagi ulush (ORDER BY SIZ hisoblandi)
        ROUND(
            (oylik_tushum * 100.0) / SUM(oylik_tushum) OVER (PARTITION BY hudud, oy), 2
        ) AS hudud_tushum_ulushi,
        
        -- 6-talab: RANK va tie-breaker bilan ROW_NUMBER orqali hududdagi o'rin
        RANK() OVER w_hudud_oy AS hudud_rank,
        ROW_NUMBER() OVER (PARTITION BY hudud, oy ORDER BY oylik_tushum DESC, sotuvchi_id ASC) AS hudud_row_num
    FROM monthly_sales
    WINDOW 
        w_sotuvchi_vaqt AS (PARTITION BY sotuvchi_id ORDER BY oy),
        w_hudud_oy AS (PARTITION BY hudud, oy ORDER BY oylik_tushum DESC)
),
top2_filtered AS (
    -- 7-talab: Top-2 sotuvchilarni alohida CTE orqali tanlash (WHERE da window funksiya ishlatilmadi)
    -- Izoh: Window funksiyalar SQL ning WHERE qismida ishlamaydi, chunki ular mantiqiy bajarilish
    -- tartibida WHERE dan keyin (SELECT bosqichida) hisoblanadi. Shuning uchun CTE ishlatiladi.
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY hudud, oy ORDER BY hudud_rank ASC, oylik_tushum DESC) as top_filter
    FROM enriched_report
)
-- Yakuniy natija: Barcha ustunlarni chiqarish va har bir hududdan eng yaxshi 2 taligni saralash
SELECT 
    ism,
    hudud,
    oy,
    ROUND(oylik_tushum, 2) AS oylik_tushum,
    otgan_oy_tushumi,
    mom_foiz_ozgarish,
    yil_boshidan_jamlanma,
    hudud_tushum_ulushi,
    hudud_rank
FROM top2_filtered
WHERE top_filter <= 2
ORDER BY oy DESC, hudud, hudud_rank;
