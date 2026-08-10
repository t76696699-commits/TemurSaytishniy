# ============================================================
# HAQIQIY misol: app/services/lesson_service.py'dagi get_lessons_by_course
# (soddalashtirilgan, lekin naqsh o'zgarmagan)
# ============================================================
from sqlalchemy import select, and_, or_, func
from sqlalchemy.orm import selectinload
from typing import List, Optional


async def get_lessons_by_course(db, course_id: int) -> List["Lesson"]:
    result = await db.execute(
        select(Lesson)
        .where(Lesson.course_id == course_id, Lesson.is_active == True)  # AND — vergul bilan
        .order_by(Lesson.order)
    )
    return result.scalars().all()   # bir nechta natija -> ro'yxat


# ============================================================
# 1) .where() — AND (vergul) va OR (or_())
# ============================================================
and_stmt = select(Lesson).where(Lesson.course_id == 41, Lesson.points_reward >= 15)
or_stmt = select(Lesson).where(or_(Lesson.order == 0, Lesson.order == 1))
combined_stmt = select(Lesson).where(
    and_(Lesson.course_id == 41, or_(Lesson.order == 0, Lesson.points_reward > 20))
)
# 107-kursdagi: WHERE course_id = 41 AND (order = 0 OR points_reward > 20)

# ============================================================
# 2) .join() — ikki usul
# ============================================================
# (a) relationship() orqali — ORM FOREIGN KEY'ni o'zi topadi:
join_via_rel = select(Lesson).join(Lesson.course).where(Course.title.ilike("%SQL%"))

# (b) qo'lda, ustun orqali — relationship() bo'lmasa yoki murakkab shart kerak bo'lsa:
join_manual = select(Lesson).join(Course, Course.id == Lesson.course_id).where(Course.id == 41)

# ============================================================
# 3) order_by + limit + offset — sahifalash
# ============================================================
PAGE_SIZE = 10
page_2 = (
    select(Lesson)
    .where(Lesson.course_id == 41)
    .order_by(Lesson.order)
    .limit(PAGE_SIZE)
    .offset(PAGE_SIZE * 1)   # 2-sahifa
)
# LIMIT'siz so'rov — production'da xavfli: agar jadvalda 1 000 000 qator
# bo'lsa, ORM ularning HAMMASINI Python obyektiga aylantirishga urinadi.

# ============================================================
# 4) Natijani olish shakllari — noto'g'ri tanlov aniq xatoga olib keladi
# ============================================================
# Aniq bitta natija kutilganda (masalan ID bo'yicha):
one_lesson = (await db.execute(select(Lesson).where(Lesson.id == 5))).scalar_one()
# -> topilmasa: NoResultFound, bir nechtasi topilsa: MultipleResultsFound

# Bitta yoki hech nima (masalan ixtiyoriy qidiruv):
maybe_lesson = (await db.execute(select(Lesson).where(Lesson.id == 999))).scalar_one_or_none()
# -> topilmasa: None (xato emas)

# Bir nechta natija kutilganda:
all_lessons = (await db.execute(select(Lesson).where(Lesson.course_id == 41))).scalars().all()

# ============================================================
# 5) func.count / func.avg — agregatsiya
# ============================================================
lesson_count = (await db.execute(
    select(func.count(Lesson.id)).where(Lesson.course_id == 41)
)).scalar_one()

avg_points = (await db.execute(
    select(func.avg(Lesson.points_reward)).where(Lesson.course_id == 41)
)).scalar_one()

print(f"Kurs 41: {lesson_count} ta dars, o'rtacha {avg_points:.1f} ball")

# ============================================================
# 6) ilike / in_ / not_ — matn qidiruvi va ro'yxat bilan solishtirish
# ============================================================
text_search = select(Course).where(Course.title.ilike("%SQL%"))
id_in_list = select(Course).where(Course.id.in_([41, 98, 107]))
negation = select(Lesson).where(~Lesson.is_active)   # yoki not_(Lesson.is_active)

# ============================================================
# 7) exists() — "hech qanday namunasi yo'q darslarni top"
# ============================================================
from sqlalchemy import exists

no_sample_stmt = select(Lesson).where(
    ~exists().where(LessonSample.lesson_id == Lesson.id)
)
# 107-kursdagi: WHERE NOT EXISTS (SELECT 1 FROM lesson_samples WHERE lesson_id = lessons.id)
# .exists() natijani "bor/yo'q"ga aylantiradi — bazaga butun subquery natijasini
# emas, faqat mantiqiy javobni so'raydi.

# ============================================================
# 8) To'liq funksiya — filtr, join, sahifalash va agregatsiyani birlashtiradi
# ============================================================
async def list_lessons_with_progress(db, course_id: int, student_id: int, page: int = 1, page_size: int = 10):
    """Kurs darslari ro'yxati, har bir dars uchun shu talaba nechta
    mashqni to'g'ri yechganini ko'rsatuvchi — kurs sahifasida tipik
    so'rov."""
    completed_subq = (
        select(func.count(ExerciseAttempt.id))
        .where(
            ExerciseAttempt.student_id == student_id,
            ExerciseAttempt.exercise_id.in_(
                select(Exercise.id).where(Exercise.lesson_id == Lesson.id)
            ),
            ExerciseAttempt.is_correct == True,
        )
        .correlate(Lesson)
        .scalar_subquery()
    )
    stmt = (
        select(Lesson.id, Lesson.title, Lesson.order, completed_subq.label("completed_count"))
        .where(Lesson.course_id == course_id, Lesson.is_active == True)
        .order_by(Lesson.order)
        .limit(page_size)
        .offset((page - 1) * page_size)
    )
    return (await db.execute(stmt)).all()
    # Har bir qator: (id, title, order, completed_count) — Lesson obyekti
    # EMAS, Row — chunki bu yerda faqat ro'yxat ko'rinishi uchun kerakli
    # yassi ma'lumot kifoya (11-darsdagi over-fetching mavzusiga bog'liq).
