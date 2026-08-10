# ============================================================
# 1) Core darajasi — jadval va so'rovni QO'LDA, klasssiz qurish
# ============================================================
from sqlalchemy import MetaData, Table, Column, Integer, String, select

metadata = MetaData()

courses_table = Table(
    "courses", metadata,
    Column("id", Integer, primary_key=True),
    Column("title", String(150), nullable=False),
)

# Core so'rovi — natija Python klassi emas, "Row" obyektlari qaytadi:
core_stmt = select(courses_table.c.id, courses_table.c.title).where(courses_table.c.id == 41)
# result = await conn.execute(core_stmt)
# row = result.first()  # row.id, row.title — lekin bu Course obyekti EMAS

# ============================================================
# 2) Bog'lovchi (association) jadval — bu platformada Core darajasida
#    e'lon qilingan haqiqiy misol (app/models/course.py'dan soddalashtirilgan)
# ============================================================
from sqlalchemy import ForeignKey, Table as CoreTable

student_courses = CoreTable(
    "student_courses", metadata,
    Column("student_id", ForeignKey("students.id", ondelete="CASCADE"), primary_key=True),
    Column("course_id", ForeignKey("courses.id", ondelete="CASCADE"), primary_key=True),
)
# Nega Core? Chunki bu jadvalning o'ziga xos xatti-harakati yo'q — u faqat
# ikkita ID juftligini saqlaydi. Alohida Python klassi keraksiz murakkablik
# qo'shgan bo'lardi.

# ============================================================
# 3) ORM darajasi — xuddi shu Course, endi to'liq deklarativ klass sifatida
# ============================================================
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship
from typing import List


class Base(DeclarativeBase):
    pass


class Course(Base):
    __tablename__ = "courses"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(150))


orm_stmt = select(Course).where(Course.id == 41)
# result = await db.execute(orm_stmt)
# course = result.scalar_one()   # course — bu HAQIQIY Course obyekti, course.title ishlaydi

# ============================================================
# 4) Engine — app/db/database.py'dagi HAQIQIY sozlash (soddalashtirilgan)
# ============================================================
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/student_platform"

engine = create_async_engine(
    DATABASE_URL,
    echo=False,      # True bo'lsa — har bir hosil bo'lgan SQL konsolga chiqadi
    future=True,
)

# Session — "fabrika", har bir so'rov uchun YANGI instansiya yaratadi:
AsyncSessionLocal = async_sessionmaker(
    bind=engine,
    class_=AsyncSession,
    expire_on_commit=False,   # commit'dan keyin ham obyekt atributlariga kirish mumkin
    autoflush=True,
)


# ============================================================
# 5) FastAPI dependency — har bir HTTP so'rovi uchun bitta Session
# ============================================================
async def get_db():
    async with AsyncSessionLocal() as session:
        yield session
        # `async with` blokidan chiqishda Session avtomatik yopiladi —
        # ochiq qolgan Session'lar connection pool'ni tugatib qo'yadi (12-dars).


# Endpoint misoli:
# @router.get("/courses/{course_id}")
# async def get_course(course_id: int, db: AsyncSession = Depends(get_db)):
#     result = await db.execute(select(Course).where(Course.id == course_id))
#     return result.scalar_one_or_none()

# ============================================================
# 6) Ommaviy (bulk) operatsiyalar uchun Core tanlanadi, ORM emas
# ============================================================
# 5000 ta talaba uchun "eslatma yuborildi" belgisini yangilash kerak bo'lsa,
# ORM orqali 5000 ta obyektni yuklab, birma-bir o'zgartirib, keyin commit
# qilish — 5000 ta obyektni xotiraga yuklaydi va identity map'ni to'ldiradi.
# Core orqali esa BITTA UPDATE bayonoti yetarli:
from sqlalchemy import update

bulk_stmt = (
    update(student_courses)
    .where(student_courses.c.course_id == 41)
    .values(reminder_sent=True)
)
# await db.execute(bulk_stmt)
# await db.commit()
# Bu — ORM'ning "har doim yaxshi" emasligining aniq misoli: yozish
# ko'lami katta bo'lsa, Core to'g'ridan-to'g'ri bitta SQL bayonoti hosil
# qiladi, ORM esa har bir qatorni Python obyektiga aylantirish narxini
# to'laydi.

# ============================================================
# 7) echo=True qanday ko'rinishda ishlaydi — ORM hech narsani yashirmaydi
# ============================================================
debug_engine = create_async_engine(DATABASE_URL, echo=True)
# Shu Engine bilan yuqoridagi orm_stmt bajarilsa, konsolga aynan shunday
# chiqadi:
#
# INFO sqlalchemy.engine.Engine SELECT courses.id, courses.title
# INFO sqlalchemy.engine.Engine FROM courses
# INFO sqlalchemy.engine.Engine WHERE courses.id = $1::INTEGER
# INFO sqlalchemy.engine.Engine [generated in 0.00021s] (41,)
#
# Bu — production'da DEBUG rejimida (settings.DEBUG orqali) yoqiladigan
# aynan shu chiqish; har qanday "sirli" ORM xatti-harakatini shu orqali
# tekshirish mumkin.

# ============================================================
# 8) Mini-solishtiruv: bir xil vazifa uchun qancha kod kerak
# ============================================================
# Xom SQL (psycopg2, ORM'siz):
#   cur.execute("SELECT id, title FROM courses WHERE id = %s", (41,))
#   row = cur.fetchone()
#   course = {"id": row[0], "title": row[1]}   # dict'ga qo'lda aylantirish
#
# SQLAlchemy Core:
#   row = (await conn.execute(select(courses_table).where(courses_table.c.id == 41))).first()
#   # baribir Row, lekin so'rov qurilishi turdosh (type-safe) va kompozitsiyalanadigan
#
# SQLAlchemy ORM:
#   course = (await db.execute(select(Course).where(Course.id == 41))).scalar_one()
#   # course.title darhol mavjud, munosabatlar course.lessons ham shunday
#
# Farq "sehr"da emas — balki bazaning qatorini Python obyektiga
# aylantirish rutinasini kim o'z zimmasiga olishida: siz qo'lda, Core
# qisman, ORM to'liq.
