R2 — Takrorlash: migratsiya va performance bo'yicha amaliyot
Урок 13 из 14
· 3 раздела
📝
Matn
Matn
#1
Ikkinchi yarim yakunlandi — modeldan production'gacha
8-11-darslarda siz butunlay yangi mas'uliyat qatlamini o'rgandingiz: model yozish yetarli emas, uni HAQIQIY, ishlab turgan production bazasiga XAVFSIZ yetkazish kerak. Alembic'ning revision zanjiri (8-dars), uch bosqichli xavfsiz ustun qo'shish (9-dars), autogenerate'ning haqiqiy xatolari (10-dars, aynan shu platformaning o'z tarixidan), va nihoyat ORM'ning performance tuzoqlari — over-fetching va connection pool (11-dars). Bu — "men modelni to'g'ri yozaman" bilan "men production'ni buzmayman" orasidagi farqni beruvchi bilim.

Eng ko'p uchraydigan xatolar — ikkinchi yarim bo'yicha
Bitta katta migratsiyada hammasini qilish (9-dars) — nullable qo'shish, backfill va NOT NULL qilish ALOHIDA migratsiyalarda bo'lishi kerak.
autogenerate'ni ko'rmasdan qo'llash (10-dars) — rename har doim drop+add sifatida taklif qilinishi mumkin, bu MA'LUMOT YO'QOTADI.
Session'ni yopmasdan qoldirish (11-dars) — bu pool tugashiga va butun serverning to'xtashiga olib kelishi mumkin.
Tranzaksiya ichida tashqi chaqiruv qilish (11-dars) — ulanishni keraksiz uzoq band qilib qo'yadi.
Bu darsning loyihasi — migratsiya + performance birgalikda
Bugungi amaliy loyihada siz R1'dagi LessonFeedback tizimiga yangi funksiya qo'shasiz: helpful_count ustuni (boshqa talabalar "foydali" deb belgilagan fikrlar soni). Bu ustunni XAVFSIZ qo'shish (9-dars naqshi bo'yicha), so'ngra uni ko'rsatuvchi so'rovni over-fetching'siz yozish (11-dars) — ikkala yarimni bitta amaliy vazifada birlashtiradi.

Kurs oxiriga tayyorgarlik
13-dars — capstone — butun kursning yakuniy sinovi: yangi funksiya uchun to'liq ORM sxemasi VA migratsiya rejasini boshidan oxirigacha loyihalash. U yerda 0-12-darslarning barcha tushunchalari kerak bo'ladi — bu darsning maqsadi shu oxirgi qadam uchun tayyorgarlik ko'rishdir.

Nega bu ikkita mavzu (migratsiya + performance) birga tekshiriladi
Real loyihalarda bu ikkalasi kamdan-kam alohida uchraydi: yangi ustun qo'shish (migratsiya masalasi) deyarli har doim "bu ustunni qanday samarali o'qish kerak" (performance masalasi) degan savol bilan birga keladi. Shuning uchun bu checkpoint ularni ATAYLAB birlashtirgan holda sinaydi — xuddi production'da bo'lgani kabi, ikkalasi bir vaqtda hal qilinishi kerak bo'lgan yagona vazifa sifatida.

O'z-o'zini tekshirish: ikkinchi yarim uchun
Loyihani topshirishdan oldin: har bir migratsiyangiz alohida-alohida downgrade()ga egami? Katta jadval uchun mo'ljallangan o'zgarishlarni 3 bosqichga bo'ldingizmi? So'rovlaringizda faqat kerakli ustunlar yuklanadimi? Session'lar har doim async with orqali yopiladimi? Bu savollarning har biriga "ha" deb javob bera olish — bu darsning haqiqiy maqsadi.

Har bir darsning bir jumlada xulosasi (8-11-darslar)
8-dars: migratsiya — modeldan production bazasigacha bo'lgan versiyalangan ko'prik; revision zanjiri qaysi tartibda qo'llanishini belgilaydi.
9-dars: katta jadvalga NOT NULL ustun qo'shish — har doim nullable+backfill+NOT NULL uch bosqichida, BITTA migratsiyada emas.
10-dars: autogenerate rename'ni drop+add deb ko'radi — bu ma'lumot yo'qotadi; har bir taklif qo'lda tekshirilishi shart.
11-dars: over-fetching — kerakmagan ustunlarni ham yuklash; connection pool cheklangan, Session yopilmasa u tugaydi.
Nega aynan helpful_count tanlandi
Bu misol ataylab tanlangan: u YANGI ustun qo'shishni (migratsiya), uni over-fetching'siz ko'rsatishni (performance) va poyga holatidan xavfsiz oshirishni (tranzaksiya) bitta kichik, lekin real vazifada birlashtiradi — xuddi production'da haqiqiy funksiya so'ralganda bo'lgani kabi.

Nima uchun checkpoint loyihalari kichik, lekin to'liq bo'ladi
R1 va R2'dagi loyihalar ataylab kichik hajmda saqlanadi — maqsad "ko'p kod yozish" emas, balki "har bir tushunchani to'g'ri joyida qo'llash"ni tekshirishdir. Kichik hajm katta ma'noni yashirmaydi: bitta UniqueConstraint noto'g'ri qo'yilgan bo'lsa, yoki bitta selectinload() unutilgan bo'lsa, bu kichik loyihada ham xuddi katta production kodidagi kabi aniq ko'rinadi.

💻
Kod
Kod
#2
python
 Nusxalash
# ============================================================
# Bugungi loyiha: helpful_count — 9-dars (xavfsiz migratsiya) +
# 11-dars (over-fetching'siz so'rov) birgalikda
# ============================================================

# --- 9-dars naqshi: 3 bosqichli xavfsiz ustun qo'shish ---
# Migratsiya 1:
def upgrade_add_helpful_count() -> None:
    op.add_column(
        'lesson_feedback',
        sa.Column('helpful_count', sa.Integer(), nullable=True, server_default='0'),
    )

# Migratsiya 2 — backfill (aslida barchasi 0 bo'lgani uchun bu holatda
# server_default allaqachon yetarli, lekin agar boshqa jadvaldan
# hisoblash kerak bo'lsa — bu qadam ZARUR bo'lardi):
def upgrade_backfill_helpful_count() -> None:
    connection = op.get_bind()
    connection.execute(sa.text(
        "UPDATE lesson_feedback SET helpful_count = 0 WHERE helpful_count IS NULL"
    ))

# Migratsiya 3:
def upgrade_finalize_helpful_count() -> None:
    op.alter_column('lesson_feedback', 'helpful_count', nullable=False)


# --- 11-dars naqshi: over-fetching'siz ko'rsatish ---
from sqlalchemy.orm import load_only

async def get_top_helpful_feedback(db, lesson_id: int, limit: int = 5):
    stmt = (
        select(LessonFeedback)
        .where(LessonFeedback.lesson_id == lesson_id)
        .options(load_only(LessonFeedback.id, LessonFeedback.rating, LessonFeedback.helpful_count))
        .order_by(LessonFeedback.helpful_count.desc())
        .limit(limit)
    )
    return (await db.execute(stmt)).scalars().all()
    # comment ustuni (potentsial uzun matn) YUKLANMAYDI — faqat ro'yxat
    # ko'rinishida kerak bo'lgan qisqa maydonlar keladi.


# --- 6-dars naqshi: helpful_count'ni xavfsiz oshirish (poyga holati) ---
from sqlalchemy import update

async def mark_feedback_helpful(db, feedback_id: int) -> bool:
    stmt = (
        update(LessonFeedback)
        .where(LessonFeedback.id == feedback_id)
        .values(helpful_count=LessonFeedback.helpful_count + 1)   # DB darajasida +1 — poyga xavfsiz
    )
    result = await db.execute(stmt)
    await db.commit()
    return result.rowcount > 0
    # Diqqat: bu yerda avval o'qib, keyin +1 qilib yozish (read-modify-write)
    # o'RNIGA to'g'ridan-to'g'ri UPDATE ... SET x = x + 1 ishlatildi — bu
    # ikkita parallel so'rov bir-birining ustidan yozib yubormasligini
    # kafolatlaydi (Core darajasidagi atomik operatsiya, 1-darsni eslang).

# ============================================================
# To'liq Alembic migratsiya fayli — 3 bosqichni HAQIQIY fayl formatida
# (odatda bular 3 ta ALOHIDA fayl bo'ladi, bu yerda o'quv maqsadida
# ketma-ket bitta faylda ko'rsatilgan)
# ============================================================
"""add helpful_count to lesson_feedback (3-step safe pattern)

Revision ID: ff66aa77bb88
Revises: ee55ff66aa77
Create Date: 2026-08-01
"""
from alembic import op
import sqlalchemy as sa

revision = 'ff66aa77bb88'
down_revision = 'ee55ff66aa77'


def upgrade() -> None:
    # 1-bosqich: nullable ustun, server_default bilan (deyarli zararsiz)
    op.add_column(
        'lesson_feedback',
        sa.Column('helpful_count', sa.Integer(), nullable=True, server_default='0'),
    )
    # 2-bosqich: backfill — bu holatda server_default allaqachon 0 qo'ygani
    # uchun texnik jihatdan ortiqcha, lekin naqshni ko'rsatish uchun qoldirilgan
    connection = op.get_bind()
    connection.execute(sa.text(
        "UPDATE lesson_feedback SET helpful_count = 0 WHERE helpful_count IS NULL"
    ))
    # 3-bosqich: endi hammasi to'lgani aniq — NOT NULL qilish xavfsiz
    op.alter_column('lesson_feedback', 'helpful_count', nullable=False)


def downgrade() -> None:
    op.drop_column('lesson_feedback', 'helpful_count')

# ============================================================
# Round-trip sinovi — production'ga qo'llashdan OLDIN (8-dars naqshi)
# ============================================================
#   alembic upgrade head
#   alembic downgrade -1
#   alembic upgrade head
# Uchala buyruq ham xatosiz o'tishi kerak — aks holda downgrade() da xato bor.

# ============================================================
# To'liq hayotiy tsikl: FastAPI endpoint'idan xavfsiz yozishgacha
# ============================================================
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession

router = APIRouter()


@router.post("/lessons/{lesson_id}/feedback/{feedback_id}/helpful")
async def mark_helpful_endpoint(
    lesson_id: int, feedback_id: int, db: AsyncSession = Depends(get_db)
):
    ok = await mark_feedback_helpful(db, feedback_id)
    return {"success": ok}
    # `db` — Depends(get_db) orqali Session (6-dars), funksiya qisqa va
    # atomik UPDATE ishlatadi (6-dars), ortiqcha ustunlarni o'qimaydi
    # (11-dars), va ustunning o'zi xavfsiz 3 bosqichli migratsiya bilan
    # qo'shilgan (9-dars). Bitta kichik endpoint — lekin unda kursning
    # besh darsi mujassam.

# ============================================================
# Atomik UPDATE'siz muqobil — nega undan qochiladi
# ============================================================
async def mark_feedback_helpful_RISKY(db, feedback_id: int) -> bool:
    feedback = await db.get(LessonFeedback, feedback_id)
    if feedback is None:
        return False
    feedback.helpful_count = feedback.helpful_count + 1   # Python'da read-modify-write
    await db.commit()
    return True
# Agar ikkita so'rov bir vaqtda helpful_count=5'ni o'qisa, ikkalasi ham
# 6'ni hisoblab, 6'ni yozadi — garchi to'g'ri natija 7 bo'lishi kerak
# bo'lsa ham. Bu xato hech qanday istisno (exception) tashlamaydi va
# loglarda ko'rinmaydi — bitta "foydali" ovoz jimgina yo'qoladi. Aynan
# shuning uchun yuqoridagi mark_feedback_helpful()da baza darajasidagi
# UPDATE ... SET x = x + 1 ishlatiladi, bu versiya emas.
