-- =====================================================================
-- AMALIY TOPSHIRIQ: Filiallar bo'yicha bonus reytingi va tahliliy hisobot
-- Texnologiya: PostgreSQL
-- =====================================================================

-- 1. Jadvalni yaratish
DROP TABLE IF EXISTS employee_bonuses CASCADE;

CREATE TABLE employee_bonuses (
    id SERIAL PRIMARY KEY,
    xodim VARCHAR(50) NOT NULL,
    filial VARCHAR(50) NOT NULL,
    oy DATE NOT NULL,
    bonus NUMERIC(10, 2) NOT NULL
);

-- Ma'lumotlarni kiritish (3 ta filial, 9 ta xodim, 6 ta oy = 54 qator)
-- Eslatma: 'Toshkent' filialida Alisher va Bekzod ning bonuslari AYNAN teng qilib kiritilgan.
INSERT INTO employee_bonuses (xodim, filial, oy, bonus) VALUES
-- Toshkent filiali
('Alisher', 'Toshkent', '2026-01-01', 1200.00),
('Bekzod', 'Toshkent', '2026-01-01', 1200.00),
('Diyor', 'Toshkent', '2026-01-01', 1500.00),
('Alisher', 'Toshkent', '2026-02-01', 1300.00),
('Bekzod', 'Toshkent', '2026-02-01', 1300.00),
('Diyor', 'Toshkent', '2026-02-01', 1400.00),
('Alisher', 'Toshkent', '2026-03-01', 1100.00),
('Bekzod', 'Toshkent', '2026-03-01', 1100.00),
('Diyor', 'Toshkent', '2026-03-01', 1600.00),
('Alisher', 'Toshkent', '2026-04-01', 1400.00),
('Bekzod', 'Toshkent', '2026-04-01', 1400.00),
('Diyor', 'Toshkent', '2026-04-01', 1550.00),
('Alisher', 'Toshkent', '2026-05-01', 1250.00),
('Bekzod', 'Toshkent', '2026-05-01', 1250.00),
('Diyor', 'Toshkent', '2026-05-01', 1450.00),
('Alisher', 'Toshkent', '2026-06-01', 1500.00),
('Bekzod', 'Toshkent', '2026-06-01', 1500.00),
('Diyor', 'Toshkent', '2026-06-01', 1700.00),

-- Samarqand filiali
('Jasur', 'Samarqand', '2026-01-01', 1000.00),
('Malika', 'Samarqand', '2026-01-01', 1100.00),
('Nodira', 'Samarqand', '2026-01-01', 950.00),
('Jasur', 'Samarqand', '2026-02-01', 1050.00),
('Malika', 'Samarqand', '2026-02-01', 1150.00),
('Nodira', 'Samarqand', '2026-02-01', 1000.00),
('Jasur', 'Samarqand', '2026-03-01', 1200.00),
('Malika', 'Samarqand', '2026-03-01', 1250.00),
('Nodira', 'Samarqand', '2026-03-01', 1100.00),
('Jasur', 'Samarqand', '2026-04-01', 1150.00),
('Malika', 'Samarqand', '2026-04-01', 1200.00),
('Nodira', 'Samarqand', '2026-04-01', 1050.00),
('Jasur', 'Samarqand', '2026-05-01', 1300.00),
('Malika', 'Samarqand', '2026-05-01', 1350.00),
('Nodira', 'Samarqand', '2026-05-01', 1200.00),
('Jasur', 'Samarqand', '2026-06-01', 1400.00),
('Malika', 'Samarqand', '2026-06-01', 1450.00),
('Nodira', 'Samarqand', '2026-06-01', 1300.00),

-- Buxoro filiali
('Oybek', 'Buxoro', '2026-01-01', 900.00),
('Sarvar', 'Buxoro', '2026-01-01', 950.00),
('Madina', 'Buxoro', '2026-01-01', 1050.00),
('Oybek', 'Buxoro', '2026-02-01', 950.00),
('Sarvar', 'Buxoro', '2026-02-01', 1000.00),
('Madina', 'Buxoro', '2026-02-01', 1100.00),
('Oybek', 'Buxoro', '2026-03-01', 1000.00),
('Sarvar', 'Buxoro', '2026-03-01', 1050.00),
('Madina', 'Buxoro', '2026-03-01', 1150.00),
('Oybek', 'Buxoro', '2026-04-01', 1100.00),
('Sarvar', 'Buxoro', '2026-04-01', 1150.00),
('Madina', 'Buxoro', '2026-04-01', 1250.00),
('Oybek', 'Buxoro', '2026-05-01', 1150.00),
('Sarvar', 'Buxoro', '2026-05-01', 1200.00),
('Madina', 'Buxoro', '2026-05-01', 1300.00),
('Oybek', 'Buxoro', '2026-06-01', 1250.00),
('Sarvar', 'Buxoro', '2026-06-01', 1300.00),
('Madina', 'Buxoro', '2026-06-01', 1400.00);

-- =====================================================================
-- 2. TTAHLILIY HISOBOT VA WINDOW FUNKSIYALAR
-- =====================================================================

/*
  IZOHLAR VA TALABLAR BAJARILISHI:

  - 3-talab (ROW_NUMBER, RANK, DENSE_RANK):
    Toshkent filialida Alisher va Bekzodning bonuslari teng bo'lgani uchun:
    * ROW_NUMBER() - ularga ketma-ket unikal raqam beradi (masalan, 1 va 2).
    * RANK() - bir xil qiymatlarga bir xil o'rin berib, keyingi o'rinni o'tkazib yuboradi (1, 1, 3).
    * DENSE_RANK() - bir xil qiymatlarga bir xil o'rin beradi, lekin keyingi o'rinni o'tkazib yubormaydi (1, 1, 2).

  - 4-talab (ROW_NUMBER tie-breaker):
    ORDER BY ichiga xodim nomi (`xodim`) kiritildi. Bu bir xil bonusli qatorlar tartibini deterministik qilish uchun zarur, aks holda bazaning ichki fizik joylashuviga tayanib qolinardi.

  - 6-talab (Jamlanma bonus freymi):
    `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` ochiq ko'rsatildi. Freymsiz variant (`RANGE` bo'yicha standart) takroriy qiymatlarda butun guruhni qo'shib yuborishi mumkin edi, `ROWS` esa aniq joriy qatorgacha bo'lgan qatorma-qator yig'indini kafolatlaydi.

  - 7-talab (Filialdagi ulush):
    `SUM(bonus) OVER (PARTITION BY filial)` yordamida `ORDER BY` siz hisoblandi.

  - 8-talab (TOP-3 va CTE):
    Window funksiyalar `WHERE` shartida ishlatilmadi, chunki SQL mantiqiy bajarilish tartibida `WHERE` bosqichi window funksiyalardan oldin bajariladi. Shuning uchun natija alohida CTE orqali orab olindi.

  - 9-talab (WINDOW bandi):
    Takrorlanuvchi `OVER` ifodalari `WINDOW` bandi orqali optimallashtirildi.
*/

WITH ranked_report AS (
    SELECT
        xodim,
        filial,
        oy,
        bonus,
        -- 3-talab: Reyting funksiyalari yonma-yon
        ROW_NUMBER() OVER w_filial_order AS rn,
        RANK() OVER w_filial_order AS rnk,
        DENSE_RANK() OVER w_filial_order AS dense_rnk,

        -- 5-talab: LAG bilan oydan oyga foiz o'zgarish (NULLIF bilan nolga bo'linishdan himoya)
        LAG(bonus) OVER w_employee_time AS prev_bonus,
        ROUND(
            ((bonus - LAG(bonus) OVER w_employee_time) * 100.0) /
            NULLIF(LAG(bonus) OVER w_employee_time, 0), 2
        ) AS oy_ozgarish_foiz,

        -- 6-talab: Jamlanma bonus (aniq freym bilan)
        SUM(bonus) OVER (
            PARTITION BY xodim
            ORDER BY oy
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS jamlanma_bonus,

        -- 7-talab: Filial bonus fondidagi ulush (ORDER BY siz)
        ROUND(
            (bonus * 100.0) / SUM(bonus) OVER (PARTITION BY filial), 2
        ) AS filial_ulushi_foiz
    FROM employee_bonuses
    WINDOW
        w_filial_order AS (PARTITION BY filial, oy ORDER BY bonus DESC, xodim ASC),
        w_employee_time AS (PARTITION BY xodim ORDER BY oy)
),
top3_branches AS (
    SELECT
        xodim,
        filial,
        oy,
        bonus,
        rnk,
        ROW_NUMBER() OVER (PARTITION BY filial, oy ORDER BY rnk, bonus DESC) as top_rn
    FROM ranked_report
)
SELECT
    xodim,
    filial,
    oy,
    bonus,
    rn,
    rnk,
    dense_rnk,
    prev_bonus,
    oy_ozgarish_foiz,
    jamlanma_bonus,
    filial_ulushi_foiz
FROM top3_branches
WHERE top_rn <= 3
ORDER BY filial, oy, rnk, xodim;
