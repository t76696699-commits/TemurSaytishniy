-- ============================================================
-- 1) Xom SQL yondashuvi (107-kursda ko'rgan uslub)
-- ============================================================
SELECT l.id, l.title, l."order", c.title AS course_title
FROM lessons l
JOIN courses c ON c.id = l.course_id
WHERE l.course_id = 41
ORDER BY l."order";

-- Python tomonida natijani QO'LDA obyektga aylantirish kerak bo'ladi:
-- rows = await conn.execute(text(raw_sql))
-- lessons = []
-- for row in rows:
--     lessons.append({"id": row.id, "title": row.title, "course_title": row.course_title})
-- Bu ishlaydi, lekin har bir so'rov uchun shu aylantirish qayta-qayta yoziladi.

-- ============================================================
-- 2) Xuddi shu narsa — SQLAlchemy 2.x deklarativ ORM bilan
-- ============================================================
from __future__ import annotations
from typing import Optional, List
from sqlalchemy import Integer, String, ForeignKey
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    # Ushbu platformaning app/db/base_class.py'dagi Base'iga o'xshash —
    # barcha modellar shundan meros oladi.
    pass


class Course(Base):
    __tablename__ = "courses"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(150))

    # Munosabat: bitta kursning ko'p darsi bor (one-to-many).
    # Bu Python atributi, jadvalda ustun EMAS — 3-darsda batafsil ko'ramiz.
    lessons: Mapped[List["Lesson"]] = relationship(back_populates="course")


class Lesson(Base):
    __tablename__ = "lessons"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(500))
    order: Mapped[int] = mapped_column(Integer, default=0)
    course_id: Mapped[int] = mapped_column(ForeignKey("courses.id"))

    course: Mapped["Course"] = relationship(back_populates="lessons")


# ============================================================
# 3) So'rov — endi SQL matni emas, Python obyekti qaytadi
# ============================================================
from sqlalchemy import select

stmt = select(Lesson).where(Lesson.course_id == 41).order_by(Lesson.order)
result = await db.execute(stmt)
lessons = result.scalars().all()

for lesson in lessons:
    # .course — bu FOREIGN KEY emas, munosabat orqali AVTOMATIK yuklangan obyekt
    print(lesson.title, "|", lesson.course.title)

# Hosil bo'lgan haqiqiy SQL'ni har doim ko'rish mumkin — ORM hech narsani
# yashirmaydi, faqat qo'lda yozishdan qutqaradi:
print(str(stmt.compile(compile_kwargs={"literal_binds": True})))
# -> SELECT lessons.id, lessons.title, lessons."order", lessons.course_id
#    FROM lessons WHERE lessons.course_id = 41 ORDER BY lessons."order"

# ============================================================
# 4) Identity map — impedance mismatch'ning "identity" muammosini
#    Session qanday yechadi (6-darsda chuqurroq)
# ============================================================
lesson_a = (await db.execute(select(Lesson).where(Lesson.id == 1))).scalar_one()
lesson_b = (await db.execute(select(Lesson).where(Lesson.id == 1))).scalar_one()
assert lesson_a is lesson_b  # bir xil Session ichida — bir xil Python obyekti!
# Bu shunchaki tasodif emas: Session har bir qatorni faqat bir marta
# xotiraga yuklaydi va keyingi so'rovlarda xuddi shu obyektni qaytaradi.

# ============================================================
# 5) "Talabaning yakunlagan kurslari + o'rtacha bahosi" — impedance
#    mismatch'ning aynan matnda tasvirlangan misoli, ikki usulda
# ============================================================

# --- Xom SQL: bitta JOIN + GROUP BY, natija tekis jadval ---
REPORT_SQL = '''
SELECT c.title, AVG(g.score) AS avg_score
FROM courses c
JOIN student_courses sc ON sc.course_id = c.id
JOIN grades g ON g.course_id = c.id AND g.student_id = sc.student_id
WHERE sc.student_id = :student_id
GROUP BY c.title;
'''

# --- ORM: xuddi shu ma'lumot, lekin natija ichma-ich obyektlar sifatida ---
from sqlalchemy import func

stmt = (
    select(Course.title, func.avg(Grade.score).label("avg_score"))
    .join(student_courses, student_courses.c.course_id == Course.id)
    .join(Grade, (Grade.course_id == Course.id) & (Grade.student_id == student_courses.c.student_id))
    .where(student_courses.c.student_id == 7)
    .group_by(Course.title)
)
# rows = (await db.execute(stmt)).all()
# for title, avg_score in rows:
#     print(f"{title}: {avg_score:.1f}")
# Diqqat: bu yerda ham natija hali ham "tekis" qator — chunki aggregatsiya
# qilingan so'rov ORM'da ham Core kabi Row qaytaradi, to'liq obyekt daraxti
# emas. To'liq ichma-ich obyekt daraxti kerak bo'lsa, relationship() orqali
# navigatsiya qilinadi (3-4-darslarda).

# ============================================================
# 6) Identity amalda — aynan shu platformaning identity map misoli
# ============================================================
# app/services/lesson_service.py bitta HTTP so'rov davomida ikki marta
# chaqiriladi (masalan logging middleware + endpoint'ning o'zi):
first_call = await get_lesson_by_id(db, lesson_id=5)
second_call = await get_lesson_by_id(db, lesson_id=5)
assert first_call is second_call
# Ikkalasi ham AYNAN BIR XIL Python obyektini qaytaradi — Session ikkinchi
# marta yangi Lesson(5) yaratmaydi, uni identity map'dan oladi. Aynan shu
# mexanizm ORM darsning boshida tasvirlangan impedance mismatch'ning
# "identity muammosi"ni bitta ham qo'shimcha kod qatorisiz yechadi.
