1. 📊 Mermaid ER Diagramma
Фрагмент кода
erDiagram
    users ||--o{ profiles : "1:1"
    users ||--o{ properties : "1:N (host)"
    users ||--o{ bookings : "1:N (guest)"
    properties ||--o{ property_amenities : "1:N"
    amenities ||--o{ property_amenities : "1:N"
    properties ||--o{ bookings : "1:N"
    bookings ||--o{ payments : "1:N"
    bookings ||--o{ reviews : "1:N"
    reviews ||--o{ reviews : "self-referential (reply)"

    users {
        int user_id PK
        varchar email
        varchar password_hash
        timestamptz created_at
    }

    profiles {
        int profile_id PK
        int user_id FK
        varchar first_name
        varchar last_name
        varchar phone
        varchar role
    }

    properties {
        int property_id PK
        int host_id FK
        varchar title
        text description
        varchar city
        varchar address
        numeric base_price
        timestamptz created_at
    }

    amenities {
        int amenity_id PK
        varchar name
    }

    property_amenities {
        int property_id PK, FK
        int amenity_id PK, FK
    }

    bookings {
        int booking_id PK
        int property_id FK
        int guest_id FK
        daterange date_range
        varchar status
        numeric total_price
        timestamptz created_at
    }

    payments {
        int payment_id PK
        int booking_id FK
        numeric amount
        varchar status
        timestamptz paid_at
    }

    reviews {
        int review_id PK
        int booking_id FK
        int property_id FK
        int guest_id FK
        int rating
        text comment
        int parent_review_id FK
        timestamptz created_at
    }
2. 🛠️ PostgreSQL Skripti (schema.sql)
SQL
-- Extensions
CREATE EXTENSION IF NOT EXISTS btree_gist;

-- 1. Users jadvali
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- 2. Profiles jadvali (1:1 users bilan)
CREATE TABLE profiles (
    profile_id SERIAL PRIMARY KEY,
    user_id INT UNIQUE NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    phone VARCHAR(30),
    role VARCHAR(20) NOT NULL,
    CONSTRAINT chk_profile_role CHECK (role IN ('guest', 'host', 'admin')),
    CONSTRAINT fk_profile_user FOREIGN KEY (user_id) 
        REFERENCES users(user_id) 
        ON DELETE CASCADE -- Foydalanuvchi o'chsa profili ham o'chadi
);

-- 3. Properties jadvali (Obyektlar)
CREATE TABLE properties (
    property_id SERIAL PRIMARY KEY,
    host_id INT NOT NULL,
    title VARCHAR(250) NOT NULL,
    description TEXT,
    city VARCHAR(100) NOT NULL,
    address TEXT NOT NULL,
    base_price NUMERIC(14,2) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT chk_base_price CHECK (base_price > 0),
    CONSTRAINT fk_property_host FOREIGN KEY (host_id) 
        REFERENCES users(user_id) 
        ON DELETE RESTRICT -- Obyekti bor hostni o'chirishni bloklaymiz
);

-- 4. Amenities jadvali (Qulayliklar)
CREATE TABLE amenities (
    amenity_id SERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL
);

-- 5. Property_Amenities (N:N junction jadval)
CREATE TABLE property_amenities (
    property_id INT NOT NULL,
    amenity_id INT NOT NULL,
    PRIMARY KEY (property_id, amenity_id),
    CONSTRAINT fk_pa_property FOREIGN KEY (property_id) 
        REFERENCES properties(property_id) 
        ON DELETE CASCADE,
    CONSTRAINT fk_pa_amenity FOREIGN KEY (amenity_id) 
        REFERENCES amenities(amenity_id) 
        ON DELETE CASCADE
);

-- 6. Bookings jadvali (Bronlar)
CREATE TABLE bookings (
    booking_id SERIAL PRIMARY KEY,
    property_id INT NOT NULL,
    guest_id INT NOT NULL,
    date_range DATERANGE NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    total_price NUMERIC(14,2) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT chk_booking_status CHECK (status IN ('pending', 'confirmed', 'cancelled', 'completed')),
    CONSTRAINT chk_total_price CHECK (total_price > 0),
    CONSTRAINT fk_booking_property FOREIGN KEY (property_id) 
        REFERENCES properties(property_id) 
        ON DELETE RESTRICT,
    CONSTRAINT fk_booking_guest FOREIGN KEY (guest_id) 
        REFERENCES users(user_id) 
        ON DELETE RESTRICT,
    -- Bir obyekt bir vaqtda ikki marta bron qilinmasligi uchun EXCLUDE constraint
    EXCLUDE USING gist (
        property_id WITH =,
        date_range WITH &&
    ) WHERE (status != 'cancelled')
);

-- 7. Payments jadvali (To'lovlar)
CREATE TABLE payments (
    payment_id SERIAL PRIMARY KEY,
    booking_id INT NOT NULL,
    amount NUMERIC(14,2) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    paid_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT chk_payment_amount CHECK (amount > 0),
    CONSTRAINT chk_payment_status CHECK (status IN ('pending', 'success', 'failed', 'refunded')),
    CONSTRAINT fk_payment_booking FOREIGN KEY (booking_id) 
        REFERENCES bookings(booking_id) 
        ON DELETE CASCADE
);

-- 8. Reviews jadvali (Sharhlar + Self-referential javoblar)
CREATE TABLE reviews (
    review_id SERIAL PRIMARY KEY,
    booking_id INT NOT NULL,
    property_id INT NOT NULL,
    guest_id INT NOT NULL,
    rating INT NOT NULL,
    comment TEXT,
    parent_review_id INT DEFAULT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT chk_rating CHECK (rating BETWEEN 1 AND 5),
    CONSTRAINT fk_review_booking FOREIGN KEY (booking_id) 
        REFERENCES bookings(booking_id) 
        ON DELETE RESTRICT,
    CONSTRAINT fk_review_property FOREIGN KEY (property_id) 
        REFERENCES properties(property_id) 
        ON DELETE CASCADE,
    CONSTRAINT fk_review_guest FOREIGN KEY (guest_id) 
        REFERENCES users(user_id) 
        ON DELETE RESTRICT,
    CONSTRAINT fk_review_parent FOREIGN KEY (parent_review_id) 
        REFERENCES reviews(review_id) 
        ON DELETE CASCADE
);

-- Indekslar (Barcha FK ustunlariga va kompozit/partial indekslar)
CREATE INDEX idx_profiles_user_id ON profiles(user_id);
CREATE INDEX idx_properties_host_id ON properties(host_id);
CREATE INDEX idx_pa_property_id ON property_amenities(property_id);
CREATE INDEX idx_pa_amenity_id ON property_amenities(amenity_id);
CREATE INDEX idx_bookings_property_id ON bookings(property_id);
CREATE INDEX idx_bookings_guest_id ON bookings(guest_id);
CREATE INDEX idx_payments_booking_id ON payments(booking_id);
CREATE INDEX idx_reviews_booking_id ON reviews(booking_id);
CREATE INDEX idx_reviews_property_id ON reviews(property_id);
CREATE INDEX idx_reviews_guest_id ON reviews(guest_id);
CREATE INDEX idx_reviews_parent_id ON reviews(parent_review_id);

-- Kompozit va Partial Unique Index misoli
CREATE UNIQUE INDEX idx_unique_completed_booking_review 
ON reviews(booking_id) 
WHERE parent_review_id IS NULL; -- Har bir asosiy bron uchun faqat bitta boshlang'ich sharh bo'lishi mumkin
3. 📝 DIZAYN.md (Arxitektura javoblari)
1. Sanalar kesishishini qanday bloklaysiz va nega aynan shu usulni tanladingiz?
Usul: PostgreSQL-ning btree_gist kengaytmasi yordamida EXCLUDE USING gist cheklovidan foydalandik va sanalarni daterange turi sifatida saqladik.

Nega: An'anaviy CHECK yoki dasturiy kod orqali tekshirish Race Condition (bir vaqtning o'zida kelgan so'rovlar) paytida xatolikka yo'l qo'yishi mumkin. GIST indeksi darajasidagi eksklyuziv qulf ma'lumotlar bazasi yadrosida ishlaydi va ikki foydalanuvchi bir vaqtning o'zida bir xil sanalarga bron yuborsa ham, ikkinchisining so'rovini bazaning o'zi avtomatik ravishda rad etishini kafolatlaydi.

2. Narx qayerda saqlanadi — obyektda, narx kalendarida yoki bronda? Nega?
Javob: Boshlang'ich narx properties.base_price da saqlanadi, lekin har bir aniq bookings.total_price ustunida tarixiy nusxa sifatida qat'iy saqlanadi.

Nega: Agar kelgusida uy egasi o'z uyining narxini oshirsa, eski qilingan bronlar narxi o'zgarmasligi kerak (moliyaviy va huquqiy shaffoflik uchun). Tarixiy nusxa saqlash bron qilingan paytdagi kelishuvni o'zgarmas saqlaydi.

3. Sharh yozish huquqini (faqat yashab chiqqan mehmon) qanday cheklaysiz?
Javob: Bazadagi reviews jadvaliga yozishdan oldin Trigger yoki CHECK jarayoni orqali bron holati completed ekanligi va sharh yozayotgan shaxs shu bronning guest_id siga teng ekanligi tekshiriladi. (Bonus qismida keltirilgan trigger aynan shu vazifani bajaradi).

4. Bekor qilingan bron DELETE qilinadimi yoki holat bilan belgilanadimi? To'lov o'tgan bo'lsa nima bo'ladi?
Javob: Aslo DELETE qilinmaydi! Bron status = 'cancelled' holatiga o'tkaziladi.

Nega: Tizimda audit, analitika va moliyaviy hisobotlar uchun barcha tarix saqlanishi shart. Agar to'lov allaqachon o'tgan bo'lsa (payments.status = 'success'), u holda to'lov yozuvi o'chirilmaydi, balki uning holati refunded (qaytarildi) deb o'zgartiriladi va bu buxgalteriya hisobotlarida to'g'ri aks etadi.

4. 🚀 Bonus Qismlar (Trigger, Materialized View va Hisobotlar)
Trigger: Faqat tugatilgan bron uchun sharh yozishni tekshirish
SQL
CREATE OR REPLACE FUNCTION check_review_permission()
RETURNS TRIGGER AS $$
BEGIN
    IF NOT EXISTS (
        SELECT 1 FROM bookings 
        WHERE booking_id = NEW.booking_id 
          AND guest_id = NEW.guest_id 
          AND status = 'completed'
    ) THEN
        RAISE EXCEPTION 'Faqatgina yakunlangan (completed) bronlar uchun sharh yozish mumkin!';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_enforce_review_rule
BEFORE INSERT ON reviews
FOR EACH ROW
EXECUTE FUNCTION check_review_permission();
Materialized View: Bandlik Dashboardi
SQL
CREATE MATERIALIZED VIEW mv_property_occupancy_dashboard AS
p.property_id,
    p.title,
    p.city,
    COUNT(b.booking_id) AS total_bookings,
    COALESCE(SUM(b.total_price), 0.00) AS total_revenue
FROM properties p
LEFT JOIN bookings b ON p.property_id = b.property_id AND b.status = 'confirmed'
GROUP BY p.property_id, p.title, p.city;
8 ta Asosiy Hisobot So'rovlari (SQL)
Sana oralig'ida bo'sh obyektlar:

SQL
SELECT * FROM properties p
WHERE NOT EXISTS (
    SELECT 1 FROM bookings b 
    WHERE b.property_id = p.property_id 
      AND b.status != 'cancelled'
      AND b.date_range && daterange('2026-09-01', '2026-09-10', '[]');
);
Bandlik foizi (90 kunlik):

SQL
SELECT p.property_id, p.title,
       ROUND(COALESCE(SUM(UPPER(b.date_range) - LOWER(b.date_range)), 0) * 100.0 / 90, 2) AS occupancy_rate_pct
FROM properties p
LEFT JOIN bookings b ON p.property_id = b.property_id 
     AND b.status = 'confirmed'
     AND b.date_range && daterange(CURRENT_DATE, CURRENT_DATE + 90, '[]');
GROUP BY p.property_id, p.title;
Uy egalari daromad reytingi:

SQL
SELECT u.user_id, p.first_name, p.last_name, SUM(pay.amount) AS total_earned
FROM users u
JOIN profiles p ON u.user_id = p.user_id
JOIN properties pr ON pr.host_id = u.user_id
JOIN bookings b ON b.property_id = pr.property_id
JOIN payments pay ON pay.booking_id = b.booking_id
WHERE pay.status = 'success'
GROUP BY u.user_id, p.first_name, p.last_name
ORDER BY total_earned DESC;
O'rtacha bahosi TOP-5 obyekt (3+ sharh bilan):

SQL
SELECT p.property_id, p.title, ROUND(AVG(r.rating), 2) AS avg_rating, COUNT(r.review_id) AS review_count
FROM properties p
JOIN reviews r ON p.property_id = r.property_id
WHERE r.parent_review_id IS NULL
GROUP BY p.property_id, p.title
HAVING COUNT(r.review_id) >= 3
ORDER BY avg_rating DESC
LIMIT 5;
Qisman / to'lanmagan bronlar:

SQL
SELECT b.booking_id, b.guest_id, b.total_price, COALESCE(pay.status, 'no_payment') AS payment_status
FROM bookings b
LEFT JOIN payments pay ON b.booking_id = pay.booking_id
WHERE b.status = 'confirmed' AND (pay.status IS NULL OR pay.status != 'success');
Mehmonlar bo'yicha bekor qilish darajasi:

SQL
SELECT u.user_id, p.first_name, p.last_name,
       COUNT(b.booking_id) AS total_bookings,
       SUM(CASE WHEN b.status = 'cancelled' THEN 1 ELSE 0 END) AS cancelled_count,
       ROUND(SUM(CASE WHEN b.status = 'cancelled' THEN 1 ELSE 0 END) * 100.0 / COUNT(b.booking_id), 2) AS cancellation_rate
FROM users u
JOIN profiles p ON u.user_id = p.user_id
JOIN bookings b ON u.user_id = b.guest_id
GROUP BY u.user_id, p.first_name, p.last_name
HAVING COUNT(b.booking_id) > 0;
Qulayliklar va baho bog'liqligi (N:N + agregat):

SQL
SELECT a.name AS amenity_name, ROUND(AVG(r.rating), 2) AS avg_property_rating
FROM amenities a
JOIN property_amenities pa ON a.amenity_id = pa.amenity_id
JOIN properties p ON pa.property_id = p.property_id
JOIN reviews r ON p.property_id = r.property_id
GROUP BY a.amenity_id, a.name
ORDER BY avg_property_rating DESC;
Oylik daromad trendi (LAG orqali):

SQL
WITH monthly_revenue AS (
    SELECT DATE_TRUNC('month', pay.paid_at) AS revenue_month, SUM(pay.amount) AS monthly_total
    FROM payments pay
    WHERE pay.status = 'success'
    GROUP BY revenue_month
)
SELECT revenue_month, monthly_total,
       LAG(monthly_total, 1) OVER (ORDER BY revenue_month) AS previous_month_total,
       monthly_total - LAG(monthly_total, 1) OVER (ORDER BY revenue_month) AS revenue_diff
FROM monthly_revenue;
