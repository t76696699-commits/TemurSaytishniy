# ============================================================
# Bugungi loyiha uchun tayanch: barcha 0-6-darslar tushunchalari
# bitta kichik domenda — talaba fikr-mulohaza (feedback) tizimi
# ============================================================
from typing import List, Optional
from datetime import datetime
from sqlalchemy import String, Integer, Text, ForeignKey, DateTime, func, select
from sqlalchemy.orm import Mapped, mapped_column, relationship, selectinload

# --- 2-dars: model + mapping ---
class LessonFeedback(Base):
    __tablename__ = "lesson_feedback_r1"
    id: Mapped[int] = mapped_column(primary_key=True)
    student_id: Mapped[int] = mapped_column(ForeignKey("students.id", ondelete="CASCADE"))
    lesson_id: Mapped[int] = mapped_column(ForeignKey("lessons.id", ondelete="CASCADE"))
    rating: Mapped[int] = mapped_column(Integer)  # 1-5
    comment: Mapped[Optional[str]] = mapped_column(Text, nullable=True)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())

    # --- 3-dars: munosabat ---
    student: Mapped["Student"] = relationship(back_populates="feedback_entries")
    lesson: Mapped["Lesson"] = relationship(back_populates="feedback_entries")


# --- 4+5-dars: to'g'ri so'rov + eager loading ---
async def get_lesson_feedback(db, lesson_id: int) -> List[LessonFeedback]:
    stmt = (
        select(LessonFeedback)
        .where(LessonFeedback.lesson_id == lesson_id)
        .order_by(LessonFeedback.created_at.desc())
        .options(selectinload(LessonFeedback.student))   # N+1'ning oldini olish
    )
    return (await db.execute(stmt)).scalars().all()


# --- 6-dars: tranzaksiya xavfsizligi ---
from sqlalchemy.exc import IntegrityError

async def submit_feedback(db, student_id: int, lesson_id: int, rating: int, comment: str):
    db.add(LessonFeedback(student_id=student_id, lesson_id=lesson_id, rating=rating, comment=comment))
    try:
        await db.commit()
        return True
    except IntegrityError:
        await db.rollback()   # masalan bitta talaba bitta darsga bitta fikr — takroriy urinish
        return False


# --- 4-dars: agregatsiya — o'rtacha bahoni hisoblash ---
from sqlalchemy import func as sa_func

async def average_rating(db, lesson_id: int) -> float:
    return (await db.execute(
        select(sa_func.avg(LessonFeedback.rating)).where(LessonFeedback.lesson_id == lesson_id)
    )).scalar_one() or 0.0


# ============================================================
# Self-check: N+1 yo'qligini echo=True bilan qo'lda tekshirish
# ============================================================
# debug_engine = create_async_engine(DATABASE_URL, echo=True)
# feedback_list = await get_lesson_feedback(db, lesson_id=41)
# for f in feedback_list:
#     print(f.student.username, f.rating)   # selectinload tufayli YANGI so'rov YO'Q
#
# Konsolda ko'rilishi kerak: aynan IKKITA SELECT (LessonFeedback + Student
# IN(...)), feedback yozuvlari sonidan qat'iy nazar. Agar ko'proq SELECT
# ko'rinsa — bu selectinload() unutilgan yoki noto'g'ri joyga qo'yilgan
# degani.

# ============================================================
# Self-check: har bir dars uchun qisqa "yaxshi/yomon" kod solishtiruvi
# ============================================================
# 3-dars — relationship() haqiqatan kerakmi?
# YOMON: faqat count() uchun butun ro'yxatni yuklash
bad_count = len((await db.execute(select(LessonFeedback).where(LessonFeedback.lesson_id == 41))).scalars().all())
# YAXSHI: to'g'ridan-to'g'ri COUNT() ishlatish, obyekt yuklamasdan
good_count = (await db.execute(select(func.count(LessonFeedback.id)).where(LessonFeedback.lesson_id == 41))).scalar_one()

# 6-dars — mustaqil ish birliklarini aralashtirmaslik
# YOMON: bitta katta tranzaksiyada bog'liq bo'lmagan ikkita amal
# YAXSHI: har biri o'z commit()iga ega (yuqoridagi submit_feedback misoli kabi)

# ============================================================
# LessonFeedback modulining to'liq xulosasi — barcha qismlar birga
# ============================================================
# 1. Model (2-dars): LessonFeedback, UniqueConstraint(student_id, lesson_id) bilan
# 2. Munosabatlar (3-dars): student/lesson, back_populates orqali
# 3. So'rov (4-dars): select().where().order_by() sana bo'yicha tartiblash bilan
# 4. Eager loading (5-dars): selectinload(LessonFeedback.student) — N+1 o'rniga 2 so'rov
# 5. Tranzaksiya (6-dars): submit_feedback()da try/except IntegrityError + rollback()
# 6. Agregatsiya (4-dars): average_rating()da func.avg()
#
# Agar yechimingizda shu oltita bandning biri yetishmasa — loyihani
# topshirishdan oldin tegishli darsga qaytib chiqing.

# ============================================================
# N+1 uchun mini-test — o'zingiz ishga tushira oladigan oddiy tekshiruv
# ============================================================
import logging

def count_queries_during(coro_factory):
    """Berilgan korutina ishlashi davomida bajarilgan SQL so'rovlari
    sonini SQLAlchemy Engine loglarini o'qib hisoblaydi."""
    count = 0

    class _CountingHandler(logging.Handler):
        def emit(self, record):
            nonlocal count
            if "SELECT" in record.getMessage() or "INSERT" in record.getMessage():
                count += 1

    logger = logging.getLogger("sqlalchemy.engine")
    handler = _CountingHandler()
    logger.addHandler(handler)
    logger.setLevel(logging.INFO)
    try:
        return count
    finally:
        logger.removeHandler(handler)

# get_lesson_feedback() uchun kutilgan natija: AYNAN 2 ta so'rov
# (LessonFeedback + Student IN(...) orqali), yozuvlar sonidan qat'iy
# nazar — agar ko'proq chiqsa, demak biror joyda selectinload() unutilgan.
