# ============================================================
# 1) Lesson modeli — bu platformaning HAQIQIY app/models/lesson.py'idan
#    soddalashtirilgan, lekin muhim qismlari o'zgartirilmagan
# ============================================================
from datetime import datetime
from typing import Optional, TYPE_CHECKING
from sqlalchemy import Integer, String, Text, Boolean, DateTime, ForeignKey, func
from sqlalchemy.orm import Mapped, mapped_column, relationship, DeclarativeBase

if TYPE_CHECKING:
    from app.models.course import Course  # faqat statik tahlil uchun — aylanma import yo'q


class Base(DeclarativeBase):
    pass


class Lesson(Base):
    __tablename__ = "lessons"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    course_id: Mapped[int] = mapped_column(
        Integer, ForeignKey("courses.id", ondelete="CASCADE"), nullable=False
    )
    title: Mapped[str] = mapped_column(String(500), nullable=False)

    # default — Python tomonida INSERT vaqtida qo'yiladi
    # server_default — bazaning o'zida DEFAULT sifatida saqlanadi (ORM'ni chetlab
    # o'tgan INSERT'lar uchun ham ishlaydi)
    order: Mapped[int] = mapped_column(Integer, default=0, server_default="0")
    points_reward: Mapped[int] = mapped_column(Integer, default=10, server_default="10")

    text_content: Mapped[Optional[str]] = mapped_column(Text, nullable=True)
    code_content: Mapped[Optional[str]] = mapped_column(Text, nullable=True)

    is_published: Mapped[bool] = mapped_column(Boolean, default=False, server_default="false")

    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )

    course: Mapped["Course"] = relationship(back_populates="lessons")


# ============================================================
# 2) default vs server_default — amalda farq
# ============================================================
new_lesson = Lesson(course_id=41, title="Yangi dars")
# new_lesson.order hali None — lekin flush/commit vaqtida ORM `default=0`ni qo'yadi
# db.add(new_lesson)
# await db.flush()
# assert new_lesson.order == 0  # Python tomonidagi default ishladi

# Endi tasavvur qiling: boshqa xizmat to'g'ridan-to'g'ri SQL orqali yozadi:
RAW_INSERT = "INSERT INTO lessons (course_id, title) VALUES (41, 'Boshqa xizmat dars')"
# Bu yerda Python default HECH QACHON ishga tushmaydi — lekin server_default="0"
# tufayli baza o'zi order=0 ni qo'yadi. Agar faqat default= bo'lganda edi
# (server_default'siz), bu qator order=NULL bilan yozilgan bo'lardi.

# ============================================================
# 3) Constraint'lar — LessonSample.lesson_id unique misoli
# ============================================================
class LessonSample(Base):
    __tablename__ = "lesson_samples"

    id: Mapped[int] = mapped_column(primary_key=True)
    lesson_id: Mapped[int] = mapped_column(
        ForeignKey("lessons.id", ondelete="CASCADE"),
        unique=True,   # bitta darsda faqat bitta namuna — BAZA darajasida kafolatlanadi
        nullable=False,
    )
    title: Mapped[str] = mapped_column(String(500))

# Ikkinchi marta bir xil lesson_id bilan qo'shishga urinish:
# db.add(LessonSample(lesson_id=1, title="Ikkinchi namuna"))
# await db.commit()
# -> sqlalchemy.exc.IntegrityError: duplicate key value violates unique constraint
# Bu xato Python validatsiyasida emas, PostgreSQL'ning o'zida sodir bo'ladi —
# hatto kimdir Pydantic tekshiruvini chetlab o'tsa ham, baza himoyalangan.

# ============================================================
# 4) Mapped[Optional[...]] va nullable — moslik
# ============================================================
class Course(Base):
    __tablename__ = "courses"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(150), nullable=False)
    # Optional[int] + nullable=True — ikkalasi mos bo'lishi kerak, aks holda
    # runtime xatosi emas, faqat mypy ogohlantirishi beriladi:
    prerequisite_course_id: Mapped[Optional[int]] = mapped_column(
        ForeignKey("courses.id", ondelete="SET NULL"), nullable=True
    )


# ============================================================
# 5) __table_args__ — kompozit unique va indeks, 107-kursdagi
#    CREATE UNIQUE INDEX / CREATE INDEX'ning ORM ekvivalenti
# ============================================================
from sqlalchemy import Index, UniqueConstraint


class StudentNote(Base):
    __tablename__ = "student_notes"
    __table_args__ = (
        # "bitta talaba — bitta darsga — bitta eslatma" qoidasi
        UniqueConstraint("student_id", "lesson_id", name="uq_student_note_per_lesson"),
        # kurs ichidagi darslarni tezkor tartiblab olish uchun kompozit indeks
        Index("ix_note_student_created", "student_id", "created_at"),
    )

    id: Mapped[int] = mapped_column(primary_key=True)
    student_id: Mapped[int] = mapped_column(ForeignKey("students.id", ondelete="CASCADE"))
    lesson_id: Mapped[int] = mapped_column(ForeignKey("lessons.id", ondelete="CASCADE"))
    note_text: Mapped[str] = mapped_column(Text)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())

# Bu ikkalasi ham Alembic migratsiyasi orqali BAZAGA yozilishi kerak —
# modelga qo'shish yetarli emas (8-darsda ko'ramiz):
#   op.create_unique_constraint("uq_student_note_per_lesson", "student_notes", ["student_id", "lesson_id"])
#   op.create_index("ix_note_student_created", "student_notes", ["student_id", "created_at"])

# ============================================================
# 6) Ushbu platformaning haqiqiy modelidan misol — Student.username/email
# ============================================================
class StudentReal(Base):
    __tablename__ = "students"
    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    # unique=True + index=True birga — ustun bir vaqtda ham noyob, ham
    # tezkor qidiruv uchun o'z indeksiga ega (app/models/user.py'dagi
    # haqiqiy qator):
    username: Mapped[str] = mapped_column(String(50), unique=True, nullable=False, index=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, nullable=False, index=True)
    hashed_password: Mapped[str] = mapped_column(String(255), nullable=False)
# unique=True o'zi allaqachon PostgreSQL'da indeks yaratadi — lekin aniq
# index=True aniqlik uchun qoldirilgan va Alembic autogenerate bilan mos
# keladi, aks holda u kod kutayotgan indeksni "ko'rmasligi" mumkin.
