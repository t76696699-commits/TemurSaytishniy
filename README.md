-- =====================================================================
-- AMALIY TOPSHIRIQ: Kategoriyalar daraxti va rekursiv rollup
-- Texnologiya: PostgreSQL, WITH RECURSIVE CTE
-- =====================================================================

-- 0. Oldindan mavjud bo'lsa o'chirish (tozalik uchun)
DROP TABLE IF EXISTS categories CASCADE;

-- 1. Jadvalni yaratish (Foreign Key ataylab vaqtincha cheklanmagan holda yoki
-- tsiklni sinash uchun avval FK siz yoki deferred holatda yaratamiz)
-- Talab: Foreign key halqani bloklamasligi uchun yoki siklni kiritish imkonini berishi kerak.
-- Odatda INSERT paytida xatolik bermasligi uchun FK ni qisman yoki keyinroq qo'shamiz,
-- yoki ALTER TABLE orqali tsikl hosil qilgandan so'ng sinab ko'rsatiladi.
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    nomi VARCHAR(100) NOT NULL,
    ota_id INT, -- REFERENCES categories(id) - sikl kiritishni sinash uchun uni quyiroqda qo'shamiz
    mahsulot_soni INT NOT NULL DEFAULT 0
);

-- 2. Ma'lumotlarni kiritish (12+ tugun, 4 daraja chuqurlik, 2 ta ildiz)
INSERT INTO categories (id, nomi, ota_id, mahsulot_soni) VALUES
-- 1-ildiz: Elektronika
(1, 'Elektronika', NULL, 10),
(2, 'Smartfonlar', 1, 15),
(3, 'Apple', 2, 25),
(4, 'Samsung', 2, 30),
(5, 'Noutbuklar', 1, 12),
(6, 'Gaming noutbuklar', 5, 8),

-- 2-ildiz: Kiyim-kechak
(7, 'Kiyim-kechak', NULL, 5),
(8, 'Erkaklar kiyimi', 7, 20),
(9, 'Kostyumlar', 8, 7),
(10, 'Ayollar kiyimi', 7, 35),
(11, 'Ko‘ylaklar', 10, 18),
(12, 'Aksessuarlar', 7, 14);

-- ID ketma-ketligini to'g'rilab qo'yish (manual ID kiritilgani uchun)
SELECT setval('categories_id_seq', (SELECT MAX(id) FROM categories));

-- Endi Foreign Key ni qo'shamiz (PostgreSQL'da standart FK INSERT vaqtida tekshiriladi, 
-- agar mavjud bo'lmagan ID ga ko'rsatsa xato beradi, lekin mavjud ID ga (hatto pastdagi yuqoriga qaratilgan bo'lsa ham) 
-- FK to'sqinlik qilmaydi, chunki o'sha ID jadvalda mavjud bo'ladi!).
ALTER TABLE categories 
ADD CONSTRAINT fk_categories_ota 
FOREIGN KEY (ota_id) REFERENCES categories(id);

-- =====================================================================
-- 6-TALAB & IZOH: 
-- Ma'lumotda ataylab sikl hosil qilish va Foreign Key nima uchun buni bloklamasligi:
-- Agar biz 1-ildizning (Elektronika) ota_id sini uning avlodi bo'lgan 6-tugunga ('Gaming noutbuklar') 
-- bog'lasak, halqa hosil bo'ladi (1 -> 5 -> 6 -> 1). 
-- Foreign Key bu yerda bloklamaydi, chunki 6-raqamli tugun bazada jismonan MAVJUD. 
-- FK faqat "ko'rsatilayotgan ID ota jadvalda bormi?" deb tekshiradi, u daraxtda halqa (sikl) 
-- hosil bo'lishini tekshirmaydi!
-- =====================================================================
UPDATE categories SET ota_id = 6 WHERE id = 1;


-- =====================================================================
-- 7-TALAB (a): ARRAY yo'l va NOT ... = ANY(yol) orqali sikl himoyasi
-- =====================================================================
CREATE OR REPLACE VIEW v_full_tree_array_protection AS
WITH RECURSIVE tree_path AS (
    -- Anchor (Baza) qismi: Ildizlar (ota_id IS NULL yoki maxsus shart)
    -- Eslatma: Sikl tufayli haqiqiy NULL ildiz qolmagan bo'lishi mumkin, shuning uchun 
    -- xavfsiz boshlanish yoki ota_id larni tekshiramiz.
    SELECT 
        id,
        nomi,
        ota_id,
        mahsulot_soni,
        1 AS daraja,
        ARRAY[id] AS yol,
        CAST(nomi AS TEXT) ASfull_path
    FROM categories
    WHERE ota_id IS NULL OR ota_id NOT IN (SELECT id FROM categories) -- yoki vaqtincha tekshiruv

    UNION ALL

    -- Rekursiv qism
    SELECT 
        c.id,
        c.nomi,
        c.ota_id,
        c.mahsulot_soni,
        tp.daraja + 1,
        tp.yol || c.id,
        tp.full_path || ' > ' || c.nomi
    FROM categories c
    JOIN tree_path tp ON c.ota_id = tp.id
    -- SIKL HIMOYASI (a usul): Agar navbatdagi ID allaqachon yo'lda qatnashgan bo'lsa, o'tkazib yuboramiz
    WHERE NOT c.id = ANY(tp.yol)
)
SELECT * FROM tree_path;


-- =====================================================================
-- 7-TALAB (b): PostgreSQL 14+ dagi CYCLE bandi orqali sikl himoyasi
-- Izoh: CYCLE bandi halqani jimgina kesib o'tmaydi, balki uni aniqlab, 
-- is_cycle deb nomlangan boolean ustunni TRUE qilib belgilaydi.
-- =====================================================================
CREATE OR REPLACE VIEW v_full_tree_cycle_clause AS
WITH RECURSIVE tree_cycle AS (
    SELECT 
        id,
        nomi,
        ota_id,
        mahsulot_soni,
        1 AS daraja,
        CAST(nomi AS TEXT) AS full_path
    FROM categories
    WHERE ota_id IS NULL OR ota_id NOT IN (SELECT id FROM categories)

    UNION ALL

    SELECT 
        c.id,
        c.nomi,
        c.ota_id,
        c.mahsulot_soni,
        tc.daraja + 1,
        tc.full_path || ' > ' || c.nomi
    FROM categories c
    JOIN tree_cycle tc ON c.ota_id = tc.id
) CYCLE id SET is_cycle USING path -- CYCLE bandi orqali avtomatik himoya
SELECT * FROM tree_cycle WHERE NOT is_cycle; -- Halqaga uchragan qismlarni过滤 qilamiz


-- =====================================================================
-- 2 & 3 & 4-TALABLAR: Butun daraxt, ixtiyoriy shox va teskari zanjir
-- =====================================================================

-- 2-talab: Butun daraxtni chekinish va tartibli yo'l bilan chiqarish (ARRAY yo'l bo'yicha saralash)
SELECT 
    repeat('---- ', daraja - 1) || nomi AS chiroyli_daraxt,
    daraja,
    full_path
FROM v_full_tree_array_protection
ORDER BY yol; -- ARRAY bo'yicha saralash daraxt tuzilishini kafolatlaydi


-- 3-talab: Ixtiyoriy tugun ostidagi butun shox (Masalan, id = 2, ya'ni Smartfonlar shoxi)
WITH RECURSIVE branch AS (
    SELECT id, nomi, ota_id, 1 AS daraja, ARRAY[id] AS yol
    FROM categories
    WHERE id = 2 -- Ixtiyoriy tugun
    
    UNION ALL
    
    SELECT c.id, c.nomi, c.ota_id, b.daraja + 1, b.yol || c.id
    FROM categories c
    JOIN branch b ON c.ota_id = b.id
    WHERE NOT c.id = ANY(b.yol)
)
SELECT * FROM branch;


-- 4-talab: Bargdan ildizgacha teskari zanjir (Masalan, id = 3, ya'ni 'Apple' dan boshlab yuqoriga)
-- IZOH: Pastga yurishda (OTA -> BOLA) JOIN sharti `child.ota_id = parent.id` bo'ladi.
-- Yuqoriga yurishda (BOLA -> OTA) esa JOIN sharti `current.ota_id = parent.id` (ya'ni o'zining ota_id si orqali yuqoridagi elementga intilish) bo'ladi. Farqi FAQAT JOIN yo'nalishi va shartidagi qaysi ustun ulanishidadir.
WITH RECURSIVE upward AS (
    SELECT id, nomi, ota_id, 1 AS daraja, ARRAY[id] AS yol
    FROM categories
    WHERE id = 3 -- Barg tugun (Apple)
    
    UNION ALL
    
    SELECT c.id, c.nomi, c.ota_id, u.daraja + 1, u.yol || c.id
    FROM categories c
    JOIN upward u ON u.ota_id = c.id -- Farqi shu yerda: o'zining ota_id si orqali ota elementga ulanamiz
    WHERE NOT c.id = ANY(u.yol)
)
SELECT * FROM upward;


-- =====================================================================
-- 5-TALAB: Rekursiv rollup (O'zi + barcha avlodlari bo'yicha mahsulotlar yig'indisi)
-- =====================================================================
-- Qo'lda tekshiruv uchun izoh:
-- Masalan, id = 2 ('Smartfonlar'): o'zida 15 ta, uning avlodlari 3 ('Apple' - 25 ta) va 4 ('Samsung' - 30 ta).
-- Jami mahsulot soni = 15 + 25 + 30 = 70 ta. Avlodlar soni = 2 ta.
WITH RECURSIVE rollup_tree AS (
    -- Boshlang'ich qism: Har bir tugun o'zini o'ziiradi
    SELECT 
        id AS root_id,
        id AS descendant_id,
        mahsulot_soni
    FROM categories
    
    UNION ALL
    
    -- Rekursiv qism: Avlodlarni topib qo'shamiz
    rt.root_id,
    c.id,
    c.mahsulot_soni
    FROM rollup_tree rt
    JOIN categories c ON c.ota_id = rt.descendant_id
    -- Sikl himoyasi qo'shamiz
    WHERE rt.root_id <> c.id -- oddiy cheklov yoki to'liq array tekshiruvi
)
SELECT 
    cat.id,
    cat.nomi,
    SUM(rt.mahsulot_soni) AS jami_mahsulotlar_soni,
    COUNT(rt.descendant_id) - 1 AS avlodlar_soni -- o'zini ayirib tashlaymiz
FROM categories cat
JOIN rollup_tree rt ON cat.id = rt.root_id
GROUP BY cat.id, cat.nomi
ORDER BY cat.id;


-- =====================================================================
-- 8-TALAB: Skript oxirida ma'lumotni dastlabki (halqasiz) holatiga qaytarish
-- =====================================================================
UPDATE categories SET ota_id = NULL WHERE id = 1;


-- =====================================================================
-- 9-TALAB (Nazariy izoh):
-- Nima uchun bu masalani ilova darajasidagi (Python/JS) tsikl bilan yechish N+1 muammosiga olib keladi?
-- Chunki ilova darajasida butun daraxtni yoki har bir tugunning bolalarini olish uchun 
-- har bir daraja va har bir element uchun alohida SQL so'rov yuborishga to'g'ri keladi (1 ta ildiz 
-- uchun so'rov -> N ta bola uchun alohida N ta so'rov -> ularning bolalari uchun yana so'rovlar). 
-- Bu tarmoq trafigini keskin oshiradi va N+1 SELECT muammosini keltirib chiqaradi. 
-- SQL'dagi WITH RECURSIVE esa bu ishni ma'lumotlar bazasining o'zida yagona, 
-- juda tezkor iterativ jarayon sifatida bajarib tugatadi.
-- =====================================================================
