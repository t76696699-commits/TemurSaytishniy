# Generate the SQL file for the social network schema
sql_content = """
-- =========================================================================
-- IJTIMOIY TARMOQ (INSTAGRAM/TWITTER TYPE) - SQL SXEMASI
-- Texnologiya: PostgreSQL
-- =========================================================================

-- Jadvallarni tozalash (tartib bo'yicha)
DROP TABLE IF EXISTS layklar CASCADE;
DROP TABLE IF EXISTS obunalar CASCADE;
DROP TABLE IF EXISTS izohlar CASCADE;
DROP TABLE IF EXISTS postlar CASCADE;
DROP TABLE IF EXISTS profillar CASCADE;
DROP TABLE IF EXISTS foydalanuvchilar CASCADE;

-- 1. FOYDALANUVCHILAR JADVALI
CREATE TABLE foydalanuvchilar (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT username_format CHECK (username ~ '^[a-zA-Z0-9_]{3,20}$') -- Regex: 3-20 belgi, alphanumeric+underscore
);

-- 2. PROFILLAR (1:1 Foydalanuvchilar bilan)
CREATE TABLE profillar (
    user_id INT PRIMARY KEY, -- PK = FK
    bio TEXT,
    avatar_url TEXT,
    CONSTRAINT fk_profil_user FOREIGN KEY (user_id) REFERENCES foydalanuvchilar(id) ON DELETE CASCADE
);

-- 3. POSTLAR (Trigger uchun layklar_soni ustuni bilan)
CREATE TABLE postlar (
    id SERIAL PRIMARY KEY,
    user_id INT NOT NULL,
    content TEXT NOT NULL,
    layklar_soni INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_post_user FOREIGN KEY (user_id) REFERENCES foydalanuvchilar(id) ON DELETE CASCADE
);

-- 4. IZOHLAR (Self-referential 1:N uchun)
CREATE TABLE izohlar (
    id SERIAL PRIMARY KEY,
    post_id INT NOT NULL,
    user_id INT NOT NULL,
    ota_izoh_id INT, -- Threaded izohlar uchun
    content TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_izoh_post FOREIGN KEY (post_id) REFERENCES postlar(id) ON DELETE CASCADE,
    CONSTRAINT fk_izoh_user FOREIGN KEY (user_id) REFERENCES foydalanuvchilar(id) ON DELETE CASCADE,
    CONSTRAINT fk_izoh_parent FOREIGN KEY (ota_izoh_id) REFERENCES izohlar(id) ON DELETE CASCADE
);

-- 5. LAYKLAR (Kompozit PK)
CREATE TABLE layklar (
    user_id INT NOT NULL,
    post_id INT NOT NULL,
    PRIMARY KEY (user_id, post_id),
    CONSTRAINT fk_layk_user FOREIGN KEY (user_id) REFERENCES foydalanuvchilar(id) ON DELETE CASCADE,
    CONSTRAINT fk_layk_post FOREIGN KEY (post_id) REFERENCES postlar(id) ON DELETE CASCADE
);

-- 6. OBUNALAR (Self-referential N:N)
CREATE TABLE obunalar (
    obunachi_id INT NOT NULL,
    obuna_bolingan_id INT NOT NULL,
    PRIMARY KEY (obunachi_id, obuna_bolingan_id),
    CONSTRAINT fk_obunachi FOREIGN KEY (obunachi_id) REFERENCES foydalanuvchilar(id) ON DELETE CASCADE,
    CONSTRAINT fk_obuna_qilingan FOREIGN KEY (obuna_bolingan_id) REFERENCES foydalanuvchilar(id) ON DELETE CASCADE,
    CONSTRAINT chk_self_follow CHECK (obunachi_id <> obuna_bolingan_id)
);

-- TRIGGERS (layklar_sonini avtomatik yangilash)
CREATE OR REPLACE FUNCTION update_like_count() RETURNS TRIGGER AS $$
BEGIN
    IF (TG_OP = 'INSERT') THEN
        UPDATE postlar SET layklar_soni = layklar_soni + 1 WHERE id = NEW.post_id;
    ELSIF (TG_OP = 'DELETE') THEN
        UPDATE postlar SET layklar_soni = layklar_soni - 1 WHERE id = OLD.post_id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_like_count
AFTER INSERT OR DELETE ON layklar
FOR EACH ROW EXECUTE FUNCTION update_like_count();

-- INDEKSLAR
CREATE INDEX idx_post_user_id ON postlar(user_id);
CREATE INDEX idx_izoh_post_id ON izohlar(post_id);
CREATE INDEX idx_obunalar_obuna_bolingan ON obunalar(obuna_bolingan_id);

-- TEST MA'LUMOTLAR
INSERT INTO foydalanuvchilar (username, email) VALUES 
('ali', 'ali@test.com'), ('vali', 'vali@test.com'), ('sarvar', 'sarvar@test.com'), 
('ziyod', 'ziyod@test.com'), ('kamola', 'kamola@test.com'), ('nargiza', 'nargiza@test.com');

INSERT INTO profillar (user_id, bio) VALUES (1, 'Salom'), (2, 'Salom'), (3, 'Salom'), (4, 'Salom'), (5, 'Salom'), (6, 'Salom');

INSERT INTO postlar (user_id, content) SELECT id, 'Content post ' || id FROM foydalanuvchilar CROSS JOIN generate_series(1,3);

INSERT INTO layklar (user_id, post_id) VALUES (1, 1), (2, 1), (3, 1), (4, 1), (5, 1), (1, 2), (2, 2);

INSERT INTO izohlar (post_id, user_id, ota_izoh_id, content) VALUES 
(1, 2, NULL, 'Birinchi izoh'), (1, 3, 1, 'Javob izoh'), (1, 4, 2, 'Javobning javobi');

INSERT INTO obunalar (obunachi_id, obuna_bolingan_id) VALUES (1, 2), (2, 1), (1, 3), (2, 3), (3, 4), (4, 5);

-- HISOBOFLAR
-- 1. Foydalanuvchi lentasi (o'zi obuna bo'lganlar postlari)
-- SELECT p.* FROM postlar p JOIN obunalar o ON p.user_id = o.obuna_bolingan_id WHERE o.obunachi_id = 1;

-- 2. O'zaro obunalar
SELECT a1.obunachi_id, a1.obuna_bolingan_id 
FROM obunalar a1 
JOIN obunalar a2 ON a1.obunachi_id = a2.obuna_bolingan_id AND a1.obuna_bolingan_id = a2.obunachi_id;

-- 3. TOP-5 mashhur post
SELECT * FROM postlar ORDER BY layklar_soni DESC LIMIT 5;

-- 4. Obunachilari eng ko'p 3 foydalanuvchi
SELECT obuna_bolingan_id, COUNT(*) AS followers_count FROM obunalar GROUP BY obuna_bolingan_id ORDER BY followers_count DESC LIMIT 3;

-- 5. Recursive CTE (Threaded izohlar)
WITH RECURSIVE comment_tree AS (
    SELECT id, post_id, content, ota_izoh_id, 0 as level FROM izohlar WHERE ota_izoh_id IS NULL
    UNION ALL
    SELECT i.id, i.post_id, i.content, i.ota_izoh_id, ct.level + 1
    FROM izohlar i JOIN comment_tree ct ON i.ota_izoh_id = ct.id
) SELECT * FROM comment_tree;

-- 6. Hech kim obuna bo'lmagan foydalanuvchilar
SELECT id, username FROM foydalanuvchilar WHERE id NOT IN (SELECT obuna_bolingan_id FROM obunalar);

-- 7. Keshni solishtirish so'rovi
SELECT p.id, p.layklar_soni AS kesh_layklar, COUNT(l.post_id) AS real_layklar
FROM postlar p LEFT JOIN layklar l ON p.id = l.post_id GROUP BY p.id HAVING p.layklar_soni <> COUNT(l.post_id);
"""

with open("social_network.sql", "w") as f:
    f.write(sql_content)

# Generate ER Diagram in Mermaid
mermaid_content = """
# Ijtimoiy Tarmoq ER Diagrammasi

```mermaid
erDiagram
    FOYDALANUVCHILAR ||--|| PROFILLAR : bor
    FOYDALANUVCHILAR ||--o{ POSTLAR : yozadi
    FOYDALANUVCHILAR ||--o{ IZOHLAR : qoldiradi
    FOYDALANUVCHILAR ||--o{ LAYKLAR : bosadi
    FOYDALANUVCHILAR ||--o{ OBUNALAR : "obuna bo'ladi (obunachi)"
    FOYDALANUVCHILAR ||--o{ OBUNALAR : "obuna qilinadi (target)"
    
    POSTLAR ||--o{ IZOHLAR : "ostida"
    POSTLAR ||--o{ LAYKLAR : "oladi"
    
    IZOHLAR ||--o{ IZOHLAR : "javob"
