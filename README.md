# ============================================================
# 1) One-to-many — Course <-> Lesson, ikki tomonlama back_populates
# ============================================================
from typing import List, Optional
from sqlalchemy import String, Integer, ForeignKey, Table, Column
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class Course(Base):
    __tablename__ = "courses"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(150))

    lessons: Mapped[List["Lesson"]] = relationship(back_populates="course")


class Lesson(Base):
    __tablename__ = "lessons"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(500))
    course_id: Mapped[int] = mapped_column(ForeignKey("courses.id", ondelete="CASCADE"))

    course: Mapped["Course"] = relationship(back_populates="lessons")


# back_populates ikki tomonni xotirada sinxronlaydi — hali commit qilinmasdan:
new_lesson = Lesson(title="Yangi dars")
course = Course(title="Test kurs")
new_lesson.course = course
assert new_lesson in course.lessons  # avtomatik — Python darajasida, bazaga tegmasdan

# ============================================================
# 2) Many-to-many — Student <-> Course, bog'lovchi jadval orqali
#    (app/models/user.py'dagi HAQIQIY yondashuv, soddalashtirilgan)
# ============================================================
student_courses = Table(
    "student_courses", Base.metadata,
    Column("student_id", ForeignKey("students.id", ondelete="CASCADE"), primary_key=True),
    Column("course_id", ForeignKey("courses.id", ondelete="CASCADE"), primary_key=True),
)


class Student(Base):
    __tablename__ = "students"
    id: Mapped[int] = mapped_column(primary_key=True)
    username: Mapped[str] = mapped_column(String(50), unique=True)

    # HAQIQIY misol: app/models/user.py'dagi Student.enrolled_courses
    enrolled_courses: Mapped[List["Course"]] = relationship(
        "Course", secondary=student_courses, back_populates="students", lazy="selectin"
    )


# Course klassiga mos qarshi tomonni qo'shamiz:
Course.students = relationship(
    "Student", secondary=student_courses, back_populates="enrolled_courses", lazy="selectin"
)

# Foydalanish — hech qanday JOIN yozilmaydi, faqat navigatsiya:
# student = (await db.execute(select(Student).where(Student.id == 7))).scalar_one()
# for c in student.enrolled_courses:
#     print(c.title)

# ============================================================
# 3) cascade="all, delete-orphan" — Student.projects HAQIQIY misoli
# ============================================================
class Project(Base):
    __tablename__ = "projects"
    id: Mapped[int] = mapped_column(primary_key=True)
    student_id: Mapped[int] = mapped_column(ForeignKey("students.id", ondelete="CASCADE"))
    title: Mapped[str] = mapped_column(String(200))


Student.projects = relationship(
    "Project", back_populates="student", cascade="all, delete-orphan"
)
# Talaba o'chirilsa (yoki student.projects.remove(p) qilinsa va commit
# bo'lsa) — bog'liq Project qatorlari ORM darajasida ham o'chadi.
# ondelete="CASCADE" — bazaning o'zida (ORM'siz ham) himoya beradi.
# cascade="all, delete-orphan" — Python Session darajasida, "yetim" obyekt
# qolib ketmasligini kafolatlaydi (masalan ro'yxatdan olib tashlanganda).

# ============================================================
# 4) uselist=False — one-to-one, Student.ranking HAQIQIY misoli
# ============================================================
class Ranking(Base):
    __tablename__ = "rankings"
    id: Mapped[int] = mapped_column(primary_key=True)
    student_id: Mapped[int] = mapped_column(ForeignKey("students.id", ondelete="CASCADE"), unique=True)
    position: Mapped[int] = mapped_column(Integer)


Student.ranking = relationship(
    "Ranking", back_populates="student", uselist=False, cascade="all, delete-orphan"
)
# student.ranking -> bitta Ranking obyekti yoki None (RO'YXAT emas)

# ============================================================
# 5) Self-referential munosabat — Course.prerequisite_course_id
#    (41 <- 98 <- 107 <- bu kurs zanjiri, haqiqiy misol)
# ============================================================
class CourseWithPrereq(Base):
    __tablename__ = "courses_v2"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(150))
    prerequisite_course_id: Mapped[Optional[int]] = mapped_column(
        ForeignKey("courses_v2.id", ondelete="SET NULL"), nullable=True
    )
    # foreign_keys= — SQLAlchemy'ga AYNAN qaysi ustun ishlatilishini aytadi,
    # bir xil jadvalga bir nechta FOREIGN KEY bo'lganda bu shart bo'lib qoladi:
    prerequisite: Mapped[Optional["CourseWithPrereq"]] = relationship(
        "CourseWithPrereq", remote_side=[id], foreign_keys=[prerequisite_course_id]
    )

# 107-kursning prerequisite'i 98, 98-ning prerequisite'i 41:
# course_107.prerequisite  -> course_98  (obyekt, ID emas)
# course_107.prerequisite.prerequisite -> course_41

# ============================================================
# 6) secondary= yetarli bo'lmagan holat — bog'lovchi jadvalda qo'shimcha
#    ma'lumot kerak bo'lganda
# ============================================================
# Agar review_helpful_votes uchun (capstone'da ko'ramiz) ovozning
# created_at vaqtini ham saqlash kerak bo'lsa, oddiy secondary= jadval
# yetarli emas — u bog'lovchi jadvalning o'z ustunlariga kirish imkonini
# bermaydi. Bunday holda bog'lovchi jadval TO'LIQ model sifatida, ikkita
# relationship() bilan yoziladi:
class ReviewHelpfulVote(Base):
    __tablename__ = "review_helpful_votes_v2"
    review_id: Mapped[int] = mapped_column(ForeignKey("course_reviews.id"), primary_key=True)
    student_id: Mapped[int] = mapped_column(ForeignKey("students.id"), primary_key=True)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    # Endi nafaqat "kim ovoz bergani", balki "qachon" ekanini ham bilish
    # mumkin — bog'lovchi Table() bilan secondary= bunday imkoniyat bermaydi.
