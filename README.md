# ============================================================
# HAQIQIY misol: backend/alembic/versions/ff0011223344_widen_phone_column_to_50.py
# (aynan shu fayl, izohlar qo'shilgan)
# ============================================================
"""widen phone column to 50

Revision ID: aa11bb22cc33
Revises: aabb11223344
Create Date: 2026-07-10

Gennis API'dan keladigan telefon raqamlari eski 20-belgili chegaradan
oshib ketishi mumkin, bu esa login paytida StringDataRightTruncationError
xatosiga olib keladi.
"""
from alembic import op
import sqlalchemy as sa

revision = 'ff0011223344'          # shu faylning o'ziga xos ID'si
down_revision = 'aabb11223344'     # bu migratsiya QAYSI migratsiyadan keyin keladi


def upgrade() -> None:
    op.alter_column(
        'students', 'phone',
        existing_type=sa.String(20),   # bazadagi HOZIRGI holat
        type_=sa.String(50),           # YANGI holat
        existing_nullable=True,
    )


def downgrade() -> None:
    op.alter_column(
        'students', 'phone',
        existing_type=sa.String(50),   # HOZIRGI (upgrade'dan keyingi) holat
        type_=sa.String(20),           # ORQAGA qaytariladigan holat
        existing_nullable=True,
    )

# ============================================================
# 1) Alembic buyruqlari — kunlik ish oqimi
# ============================================================
# Yangi bo'sh migratsiya yaratish (qo'lda yozish uchun):
#   alembic revision -m "widen phone column to 50"
#
# Modelga asoslanib avtomatik migratsiya TAKLIF qildirish (QAYTA TEKSHIRISH SHART):
#   alembic revision --autogenerate -m "add is_pinned to student_notes"
#
# Barcha kutilayotgan migratsiyalarni qo'llash:
#   alembic upgrade head
#
# Bitta migratsiyani orqaga qaytarish:
#   alembic downgrade -1
#
# Hozirgi holatni ko'rish:
#   alembic current
#   alembic history --verbose

# ============================================================
# 2) Revision zanjiri — nima uchun down_revision muhim
# ============================================================
# migratsiya A: revision="aaa111", down_revision=None            (birinchi migratsiya)
# migratsiya B: revision="bbb222", down_revision="aaa111"        (A'dan keyin)
# migratsiya C: revision="ccc333", down_revision="bbb222"        (B'dan keyin)
#
# `alembic upgrade head` — A -> B -> C tartibida, ZANJIR bo'yicha qo'llaydi.
# Agar ikkita migratsiya bir xil down_revision'ga ishora qilsa (ikki
# dasturchi parallel branch'da yozgan) — Alembic "multiple heads" xatosini
# beradi, buni `alembic merge` bilan hal qilish kerak.

# ============================================================
# 3) Round-trip tekshiruvi — production'dan OLDIN har doim bajariladi
# ============================================================
#   alembic upgrade head       # yangi migratsiyani qo'llash
#   alembic downgrade -1       # orqaga qaytarish — downgrade() ishlaydimi?
#   alembic upgrade head       # yana oldinga — hech narsa buzilmadimi?
#
# Agar ikkinchi qadamda xato chiqsa (masalan downgrade() yozilmagan yoki
# noto'g'ri) — bu migratsiya production'ga HALI tayyor emas.

# ============================================================
# 4) alembic.ini — ko'pincha ilova URL'idan farqli, SINXRON ulanish
# ============================================================
# alembic.ini:
#   sqlalchemy.url = postgresql+psycopg2://user:pass@localhost/student_platform
#
# app/db/database.py (ilovaning o'zi):
#   DATABASE_URL = postgresql+asyncpg://user:pass@localhost/student_platform
#
# Diqqat: drayver farqli (+psycopg2 vs +asyncpg), lekin baza bir xil.
# Bu ataylab — migratsiya kodi sinxron, oddiy va bashoratlanadigan bo'lishi
# uchun, ilova esa yuqori parallellik uchun asinxron qoladi.

# ============================================================
# 5) Yana bir haqiqiy misol — bu platformaning o'z tarixidan,
#    achievements jadvaliga category/icon qo'shish
# ============================================================
"""add category and icon to achievements

Revision ID: ff2233445566
Revises: ee11223344aa
Create Date: 2026-06-17
"""
from alembic import op
import sqlalchemy as sa

revision = 'ff2233445566'
down_revision = 'ee11223344aa'


def upgrade() -> None:
    op.add_column('achievements', sa.Column('category', sa.String(50), nullable=True, server_default='general'))
    op.add_column('achievements', sa.Column('icon', sa.String(20), nullable=True, server_default='🏆'))


def downgrade() -> None:
    op.drop_column('achievements', 'icon')
    op.drop_column('achievements', 'category')

# Diqqat: downgrade() ustunlarni QO'SHISHGA teskari tartibda o'chiradi
# (avval icon, keyin category) — bu bir-biriga bog'liq har qanday
# operatsiyalar ketma-ketligi uchun umumiy qoida.

# ============================================================
# Foydali alembic history buyruqlari — sxema tarixini o'qish
# ============================================================
#   alembic history --verbose          # barcha xabarlar bilan to'liq zanjir
#   alembic history -r ee11223344aa:   # aniq revisiondan boshlab hammasi
#   alembic show ff2233445566          # aniq migratsiyaning tarkibi
#   alembic heads                       # bir nechta "bosh" borligini tekshirish
#                                        # (birlashtirilmagan parallel branch belgisi)

# ============================================================
# 6) alembic.ini — haqiqatan ishlatiladigan minimal sozlamalar to'plami
# ============================================================
# [alembic]
# script_location = alembic
# sqlalchemy.url = postgresql+psycopg2://user:pass@localhost/student_platform
#
# [loggers]
# keys = root,sqlalchemy,alembic
#
# Diqqat: bu yerdagi sqlalchemy.url ilovaning settings.DATABASE_URL'idan
# ALOHIDA sozlanadi (odatda env.py ichida, muhit o'zgaruvchilarini
# o'qiydigan qayta yozish orqali) — bu ataylab: migratsiyalarni ilovaning
# to'liq konfiguratsiyasi mavjud bo'lmagan joyda ham (masalan alohida CI
# bosqichida) ishga tushirish mumkin bo'lishi uchun.
