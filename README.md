# ============================================================
# XATO NAQSH — bitta migratsiyada NOT NULL + DEFAULT, katta jadvalda xavfli
# ============================================================
def upgrade_BAD() -> None:
    op.add_column(
        'exercise_attempts',
        sa.Column('difficulty_score', sa.Integer(), nullable=False, server_default='0'),
    )
    # Eski PostgreSQL versiyalarida (yoki DEFAULT'ni keyinroq murakkab
    # hisoblash orqali to'ldirish kerak bo'lganda) bu millionlab qatorli
    # jadvalda uzoq EXCLUSIVE LOCK'ga olib kelishi mumkin.

# ============================================================
# TO'G'RI NAQSH — 3 ta ALOHIDA migratsiya
# ============================================================

# --- Migratsiya 1: nullable ustun qo'shish (tez, qulflashsiz) ---
def upgrade_step1() -> None:
    op.add_column(
        'exercise_attempts',
        sa.Column('difficulty_score', sa.Integer(), nullable=True, server_default='0'),
    )

def downgrade_step1() -> None:
    op.drop_column('exercise_attempts', 'difficulty_score')


# --- Migratsiya 2: backfill — kichik partiyalarda ---
def upgrade_step2() -> None:
    connection = op.get_bind()
    while True:
        result = connection.execute(sa.text(
            """
            UPDATE exercise_attempts
            SET difficulty_score = 0
            WHERE id IN (
                SELECT id FROM exercise_attempts
                WHERE difficulty_score IS NULL
                LIMIT 1000
            )
            """
        ))
        if result.rowcount == 0:
            break   # barcha qatorlar to'ldirildi

def downgrade_step2() -> None:
    pass   # backfill'ni orqaga qaytarish shart emas — qiymatlar qoladi


# --- Migratsiya 3: NOT NULL qilish — faqat backfill tugagach ---
def upgrade_step3() -> None:
    op.alter_column('exercise_attempts', 'difficulty_score', nullable=False)

def downgrade_step3() -> None:
    op.alter_column('exercise_attempts', 'difficulty_score', nullable=True)

# ============================================================
# Zero-downtime: ustun nomini o'zgartirish — TO'G'RI usul
# ============================================================
# 1-deploy: yangi ustun qo'shiladi, ESKI kod hali eski ustunga yozadi:
#   op.add_column('courses', sa.Column('image_url', sa.String(500), nullable=True))
#
# 2-deploy: ilova kodi IKKALASIGA ham yozadigan qilib yangilanadi
#   (eski_ustun = qiymat; yangi_ustun = qiymat) — bu davrda eski VA yangi
#   kod versiyalari birga ishlashi mumkin.
#
# 3-deploy: barcha o'qish yangi ustundan bo'lishi ta'minlangach,
#   eski ustun endi HECH QAYERDA o'qilmaydi.
#
# 4-migratsiya: faqat SHUNDAN KEYIN eski ustun o'chiriladi:
#   op.drop_column('courses', 'cover_image_url')

# ============================================================
# NOT VALID CHECK constraint — cheklovni ham 2 bosqichda qo'shish
# ============================================================
# Yangi CHECK constraint qo'shish ham xuddi shu muammoga ega: PostgreSQL
# sukut bo'yicha BARCHA mavjud qatorlarni darhol tekshiradi. NOT VALID
# bilan bu tekshiruv KECHIKTIRILADI:
def upgrade_check_step1() -> None:
    op.execute(
        "ALTER TABLE exercise_attempts "
        "ADD CONSTRAINT ck_score_non_negative CHECK (difficulty_score >= 0) NOT VALID"
    )
    # Bu DARHOL bajariladi — faqat YANGI/o'zgargan qatorlar tekshiriladi.

def upgrade_check_step2() -> None:
    op.execute("ALTER TABLE exercise_attempts VALIDATE CONSTRAINT ck_score_non_negative")
    # Bu ROW-darajasidagi SHARED LOCK bilan ishlaydi (EXCLUSIVE emas) —
    # boshqa yozuvlarni bloklamaydi, faqat mavjud qatorlarni tekshiradi.

# ============================================================
# To'liq, HAQIQIY fayl formatidagi migratsiya — 1-bosqich (nullable qo'shish)
# ============================================================
"""add difficulty_score to exercise_attempts (step 1 of 3)

Revision ID: aa11bb22cc00
Revises: ff0011223344
Create Date: 2026-07-20
"""
from alembic import op
import sqlalchemy as sa

revision = 'aa11bb22cc00'
down_revision = 'ff0011223344'


def upgrade() -> None:
    op.add_column(
        'exercise_attempts',
        sa.Column('difficulty_score', sa.Integer(), nullable=True, server_default='0'),
    )


def downgrade() -> None:
    op.drop_column('exercise_attempts', 'difficulty_score')

# ============================================================
# Backfill vaqtini baholash — sikl qancha davom etishini oldindan bilish
# ============================================================
# Agar jadvalda 5 000 000 qator bo'lsa va har bir partiya (1000 qator)
# ~50ms davom etsa:
#   5_000_000 / 1000 = 5000 ta partiya
#   5000 * 50ms = 250 000ms = ~4.2 daqiqa
# Bu hisob-kitob — backfill migratsiyasini tinch soatga rejalashtirish
# yoki uni fon jarayoni (background job) sifatida bajarish kerakligini
# oldindan aniqlashga yordam beradi.

# ============================================================
# Partiyalar orasida pauza — doimiy yuklamani oldini olish
# ============================================================
import time

def upgrade_backfill_with_pause() -> None:
    connection = op.get_bind()
    while True:
        result = connection.execute(sa.text(
            "UPDATE exercise_attempts SET difficulty_score = 0 "
            "WHERE id IN (SELECT id FROM exercise_attempts "
            "WHERE difficulty_score IS NULL LIMIT 1000)"
        ))
        if result.rowcount == 0:
            break
        time.sleep(0.1)   # qisqa pauza — partiyalar orasida boshqa so'rovlarga vaqt beradi
# Pauzasiz backfill tezroq tugashi mumkin, lekin butun migratsiya
# davomida disk/CPU'ga uzluksiz yuklama beradi, ilovaning odatiy trafigi
# bilan raqobatlashadi.
