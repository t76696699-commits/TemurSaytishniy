# ============================================================
# 1) N+1 — muammoning o'zi (lazy loading sukut bo'yicha)
# ============================================================
from sqlalchemy import select
from sqlalchemy.orm import relationship, selectinload, joinedload, Mapped
from typing import List

# lazy= ko'rsatilmagan — sukut bo'yicha "select" (lazy):
class StudentLazy(Base):
    __tablename__ = "students_lazy"
    id: Mapped[int] = mapped_column(primary_key=True)
    enrolled_courses: Mapped[List["Course"]] = relationship(secondary=student_courses)


# XATO NAQSH — bu 50 ta talaba uchun 1 + 50 = 51 ta so'rov yuboradi:
students = (await db.execute(select(StudentLazy))).scalars().all()   # so'rov #1
for s in students:
    print(s.username, len(s.enrolled_courses))   # HAR BIR s uchun YANGI so'rov!

# ============================================================
# 2) selectinload() — 51 ta so'rov o'rniga aniq 2 ta
# ============================================================
stmt = select(StudentLazy).options(selectinload(StudentLazy.enrolled_courses))
students = (await db.execute(stmt)).scalars().all()
# Sahna ortida ikkita so'rov:
#   SELECT * FROM students_lazy
#   SELECT * FROM courses JOIN student_courses ON ... WHERE student_courses.student_id IN (1, 2, 3, ..., 50)
for s in students:
    print(s.username, len(s.enrolled_courses))   # YANGI so'rov YO'Q — allaqachon xotirada

# ============================================================
# 3) HAQIQIY misol: app/services/lesson_service.py'dagi naqsh
# ============================================================
async def get_lesson_by_id(db, lesson_id: int):
    result = await db.execute(
        select(Lesson)
        .where(Lesson.id == lesson_id)
        .options(
            selectinload(Lesson.exercises),   # bitta darsning barcha mashqlari
            selectinload(Lesson.files),        # bitta darsning barcha fayllari
        )
    )
    return result.scalar_one_or_none()
# Bu ikki relationship() eager qilinmasa, har bir /lessons/{id} so'rovi
# darsning har bir mashqi va fayli uchun QO'SHIMCHA so'rov yuborar edi.

# ============================================================
# 4) joinedload() — bitta JOIN so'rovi, lekin cartesian ko'payish xavfi bilan
# ============================================================
one_to_one_stmt = select(StudentLazy).options(joinedload(StudentLazy.ranking))
# Ranking — uselist=False (one-to-one), shuning uchun joinedload xavfsiz:
# natijada takrorlanish yo'q, bitta JOIN yetarli.

# XATO ISHLATISH: katta ro'yxat uchun joinedload — 100 ta kursga yozilgan
# talaba uchun natija 100 marta takrorlangan talaba qatorini o'z ichiga oladi:
risky_stmt = select(StudentLazy).options(joinedload(StudentLazy.enrolled_courses))  # XATO NAQSH

# ============================================================
# 5) lazy="raise" — N+1'ni development bosqichida ushlash
# ============================================================
class StudentSafe(Base):
    __tablename__ = "students_safe"
    id: Mapped[int] = mapped_column(primary_key=True)
    enrolled_courses: Mapped[List["Course"]] = relationship(
        secondary=student_courses, lazy="raise"
    )

# students = (await db.execute(select(StudentSafe))).scalars().all()
# for s in students:
#     print(s.enrolled_courses)
# -> sqlalchemy.exc.InvalidRequestError: 'StudentSafe.enrolled_courses' is not
#    available due to lazy='raise' — .options(selectinload(...)) yozishni
#    UNUTGANINGIZNI darhol, test bosqichida ko'rsatadi.

# ============================================================
# 6) contains_eager() — allaqachon join qilingan natijadan foydalanish
# ============================================================
from sqlalchemy.orm import contains_eager

# Filtrlash uchun join allaqachon yozilgan — shu natijadan Course'ni ham to'ldiramiz:
already_joined_stmt = (
    select(StudentLazy)
    .join(StudentLazy.enrolled_courses)
    .where(Course.difficulty_level == "Advanced")
    .options(contains_eager(StudentLazy.enrolled_courses))
)
# selectinload() ishlatilganda bu yerda IKKINCHI marta join bajarilar edi —
# contains_eager() ORM'ga "join natijasi allaqachon bor, qayta so'rama" deydi.

# ============================================================
# 7) So'rovlar sonini taxminiy solishtirish — oldin/keyin
# ============================================================
# Stsenariy: 30 ta guruh ro'yxati sahifasi, har biri uchun a'zolar soni kerak.
#
# lazy (sukut bo'yicha), eager loading'siz:
#   1 so'rov (guruhlar) + 30 so'rov (har bir group.students uchun bittadan) = 31 so'rov
#
# selectinload():
#   1 so'rov (guruhlar) + 1 so'rov (barcha guruhlarning barcha a'zolari IN(...) orqali) = 2 so'rov
#
# joinedload() (xavfsiz, chunki guruhlar ro'yxati kichik):
#   1 so'rov (JOIN darhol barcha ma'lumotni keltiradi) = 1 so'rov,
#   lekin guruh ma'lumotini takrorlaydigan talaba qatorlari bilan (yuqoridagi 4-band).
#
# Xulosa: 31 va 2 so'rov orasidagi farq gipotetik emas — tarmoq kechikishi
# ~5ms/so'rov bo'lganda, bu farq faqat shu bitta endpoint uchun ~155ms va
# ~10ms orasidagi farq degani.

# ============================================================
# 8) Bir necha munosabatni bitta chaqiruvda — selectinload().selectinload()
# ============================================================
deep_stmt = (
    select(Course)
    .options(
        selectinload(Course.lessons).selectinload(Lesson.exercises)
    )
)
# Bu Course -> Lesson -> Exercise'ni aynan 3 ta so'rovda yuklaydi (daraxt
# darajasining har biriga bittadan), agar har bir daraja ichma-ich
# sikllarda lazy yuklansa kelib chiqadigan N+1+1 so'rov o'rniga.
