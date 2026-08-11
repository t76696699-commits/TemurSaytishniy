Capstone: yangi funksiya uchun ORM sxema va migratsiya rejasi
Урок 14 из 14
· 3 раздела
✓ Пройден
📝
Matn
Matn
#1
Capstone vazifasi: "Kurs sharhlari" (Course Reviews) funksiyasi
Butun kurs davomida siz LessonFeedback (bitta darsga fikr) tizimini qurdingiz. Capstone'da undan bir daraja yuqoriga chiqamiz: talaba butun KURSNI tugatgandan keyin unga umumiy sharh va bahoQ qoldiradigan Course Review tizimini — modeldan production'gacha, to'liq — loyihalaysiz. Bu vazifa ataylab kattaroq: unda moderatsiya holati (yangi sharh avval "kutilmoqda", keyin "tasdiqlangan" bo'ladi), bitta kursga ko'p sharh, bitta sharhga ko'p "foydali ovoz" kabi bir necha munosabat qatlami bor — bu 0-12-darslarning HAMMASINI talab qiladigan minimal, lekin real domen.

1-qadam: talablarni modelga aylantirish (0-3-darslar)
Har qanday real loyiha talablardan boshlanadi: "talaba faqat TUGATGAN kursiga sharh yoza oladi" (bu — Enrollment bilan bog'liqlik, constraint emas, business logic); "bitta talaba bitta kursga faqat bitta sharh yozadi" (bu — UniqueConstraint, 2-darsdagi kabi); "sharh 1-5 baho va matn ega" (oddiy ustunlar); "moderatorlar sharhni tasdiqlashi yoki rad etishi kerak" (status ustuni + enum); "boshqa talabalar sharhni foydali deb belgilashi mumkin" (bu ALOHIDA jadval — many-to-many, 3-darsdagi kabi, chunki "kim qaysi sharhni foydali deb belgiladi" ma'lumoti kerak, shunchaki son emas).

Course Review sxemasining vizual rejasi
one-to-many

one-to-many
(muallif)

many-to-many
(kim foydali deb belgiladi)

many-to-many

courses
id PK, title

course_reviews
id PK, course_id FK, student_id FK,
rating, text, status (enum)

students
id PK

review_helpful_votes
review_id FK, student_id FK

Diagrammada capstone vazifasida loyihalanadigan Course Review sxemasi ko'rsatilgan: course_reviews — courses va students bilan ikkita one-to-many munosabatda (composite UniqueConstraint(student_id, course_id) bilan), review_helpful_votes esa "kim qaysi sharhni foydali deb belgiladi" ma'lumotini saqlovchi alohida many-to-many bog'lovchi jadval.

2-qadam: so'rov naqshlarini oldindan loyihalash (4-6-darslar)
Model yozishdan oldin "bu ma'lumot qanday o'qiladi" savolini berish kerak: kurs sahifasida so'nggi tasdiqlangan sharhlar ro'yxati (sahifalash bilan, 4-dars), har bir sharh muallifi ismi bilan birga (N+1'siz selectinload, 5-dars), yangi sharh qo'shish va moderatsiya holatini o'zgartirish (tranzaksiya xavfsizligi, 6-dars). Bu savollarga oldindan javob berish — keyinchalik "modelni to'g'ri yozdim, lekin so'rov yozish qiyin" degan holatning oldini oladi.

3-qadam: migratsiya rejasini bosqichlarga bo'lish (8-10-darslar)
Yangi funksiya odatda BITTA emas, bir NECHTA migratsiyani talab qiladi: (1) asosiy course_reviews jadvalini yaratish (yangi jadval — xavfsiz, mavjud ma'lumotga ta'sir qilmaydi); (2) review_helpful_votes bog'lovchi jadvalini yaratish; (3) agar keyinchalik helpful_count kabi hisoblangan ustun qo'shilsa — 9-darsdagi 3 bosqichli naqsh. Har bir migratsiya downgrade()ga ega bo'lishi va round-trip sinovidan (8-dars) o'tishi kerak.

4-qadam: performance xavfsizlik chegaralarini belgilash (11-dars)
Loyihalash bosqichidayoq performance qoidalarini yozib qo'yish kerak: kurs sahifasidagi sharhlar ro'yxati load_only() bilan faqat kerakli ustunlarni oladi (to'liq matn emas, qisqa preview); muallif ma'lumoti selectinload() bilan eager yuklanadi; moderatsiya paneli (ko'p sharhni ko'radigan joy) uchun alohida, kattaroq sahifalash chegarasi qo'yiladi. Bu qoidalar — kodni yozishdan OLDIN qog'ozda belgilangan bo'lishi kerak, keyin emas.

Nega bu "capstone" — nima uni maxsus qiladi
Bu loyiha boshqa darslardan farqli o'laroq bitta tushunchani emas, BUTUN JARAYONNI sinaydi: talabdan modelgacha, modeldan migratsiyagacha, migratsiyadan xavfsiz so'rovgacha. Real ish joyida "ORM'ni bilaman" kamdan-kam alohida talab qilinadi — o'rniga "yangi funksiyani boshidan oxirigacha, xavfsiz va samarali qura olaman" talab qilinadi. Shu — aynan ushbu darsning maqsadi.

Nega Course Review, LessonFeedback emas — domenlarni ataylab farqlash
R1/R2'da siz LessonFeedback (bitta darsga, oddiy) tizimini qurdingiz. Capstone'da esa ataylab murakkabroq domen tanlangan: Course Review'da moderatsiya holati (uch xil qiymat — statik ikki qiymat emas) va IKKINCHI darajali munosabat (kim qaysi sharhni foydali deb belgiladi) bor. Bu farq ataylab — capstone shunchaki oldingi loyihani takrorlash emas, balki undan bir necha qadam murakkabroq real vaziyatga tayyorlaydi.

Yakuniy baholash mezoni — nima "yaxshi yechim"ni belgilaydi
Bu loyihada eng muhim narsa — kodning "ishlashi" emas (garchi bu ham zarur), balki HAR BIR QARORNING asoslanganligi: nega aynan shu constraint, nega aynan shu yuklash strategiyasi, nega migratsiya aynan shu tartibda bo'lingan. Yakuniy hisobot — bu texnik bilimni og'zaki tushuntira olish qobiliyatini sinovdan o'tkazadi, bu esa haqiqiy jamoada ishlashning ajralmas qismi.

💻
Kod
Kod
#2
python
 Nusxalash
# ============================================================
# 1-qadam: modellar — talablardan kelib chiqqan holda (0-3-darslar)
# ============================================================
import enum
from datetime import datetime
from typing import Optional, List
from sqlalchemy import (
    String, Text, Integer, Boolean, DateTime, ForeignKey, UniqueConstraint,
    CheckConstraint, Index, func, select, update,
)
from sqlalchemy.orm import Mapped, mapped_column, relationship, selectinload, load_only


class ReviewStatus(str, enum.Enum):
    pending = "pending"      # yangi sharh — moderatsiya kutmoqda
    approved = "approved"    # tasdiqlangan — jamoat ko'radi
    rejected = "rejected"    # rad etilgan


class CourseReview(Base):
    __tablename__ = "course_reviews"
    __table_args__ = (
        # "bitta talaba — bitta kurs — bitta sharh" (talab #2)
        UniqueConstraint("student_id", "course_id", name="uq_review_per_student_course"),
        CheckConstraint("rating BETWEEN 1 AND 5", name="ck_review_rating_range"),
        Index("ix_review_course_status", "course_id", "status"),
    )

    id: Mapped[int] = mapped_column(primary_key=True)
    student_id: Mapped[int] = mapped_column(ForeignKey("students.id", ondelete="CASCADE"))
    course_id: Mapped[int] = mapped_column(ForeignKey("courses.id", ondelete="CASCADE"))
    rating: Mapped[int] = mapped_column(Integer)
    review_text: Mapped[Optional[str]] = mapped_column(Text, nullable=True)
    status: Mapped[str] = mapped_column(String(20), default=ReviewStatus.pending, server_default="pending")
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())

    student: Mapped["Student"] = relationship(back_populates="course_reviews")
    course: Mapped["Course"] = relationship(back_populates="reviews")


# Ko'p-ko'p munosabat: "kim qaysi sharhni foydali deb belgiladi" (talab #5)
review_helpful_votes = Table(
    "review_helpful_votes", Base.metadata,
    Column("review_id", ForeignKey("course_reviews.id", ondelete="CASCADE"), primary_key=True),
    Column("student_id", ForeignKey("students.id", ondelete="CASCADE"), primary_key=True),
)


# ============================================================
# 2-qadam: so'rov naqshlari — oldindan loyihalangan (4-6-darslar)
# ============================================================
async def get_approved_reviews(db, course_id: int, page: int = 1, page_size: int = 10):
    stmt = (
        select(CourseReview)
        .where(CourseReview.course_id == course_id, CourseReview.status == ReviewStatus.approved)
        .order_by(CourseReview.created_at.desc())
        .options(
            load_only(CourseReview.id, CourseReview.rating, CourseReview.review_text, CourseReview.created_at),
            selectinload(CourseReview.student).load_only(Student.username, Student.avatar_url),
        )
        .limit(page_size)
        .offset((page - 1) * page_size)
    )
    return (await db.execute(stmt)).scalars().all()


async def submit_review(db, student_id: int, course_id: int, rating: int, text: str) -> bool:
    from sqlalchemy.exc import IntegrityError
    db.add(CourseReview(student_id=student_id, course_id=course_id, rating=rating, review_text=text))
    try:
        await db.commit()
        return True
    except IntegrityError:
        await db.rollback()   # UniqueConstraint buzilgan — allaqachon sharh bor
        return False


async def moderate_review(db, review_id: int, approve: bool) -> None:
    new_status = ReviewStatus.approved if approve else ReviewStatus.rejected
    await db.execute(update(CourseReview).where(CourseReview.id == review_id).values(status=new_status))
    await db.commit()

# ============================================================
# 3-qadam: migratsiya rejasi (8-10-darslar) — bosqichlar ro'yxati
# ============================================================
# Migratsiya A: course_reviews jadvalini yaratish (yangi jadval — xavfsiz)
# Migratsiya B: review_helpful_votes bog'lovchi jadvalini yaratish
# Migratsiya C (kelajakda, agar kerak bo'lsa): helpful_count ustuni —
#   9-darsdagi 3 bosqichli naqsh (nullable -> backfill -> NOT NULL)
#
# Har biri: alohida revision, downgrade() bilan, round-trip sinovidan
# o'tgan (8-dars).

# ============================================================
# 4-qadam: performance chegaralari (11-dars) — kod yozishdan OLDIN qaror
# ============================================================
MAX_REVIEWS_PAGE_SIZE = 20          # kurs sahifasi uchun
MAX_MODERATION_PAGE_SIZE = 100      # moderatsiya paneli uchun (ko'proq ma'lumot kerak)
# review_text to'liq matn sifatida FAQAT bitta sharh ochilganda yuklanadi,
# ro'yxat ko'rinishida emas (over-fetching'dan qochish, 11-dars).

# ============================================================
# Yakuniy qarorlar xaritasi — qaysi loyihaviy qaror qaysi darsga tegishli
# ============================================================
# UniqueConstraint(student_id, course_id)      -> 2-dars (baza darajasidagi cheklov)
# CheckConstraint(rating BETWEEN 1 AND 5)      -> 2-dars (baza darajasidagi validatsiya)
# review_helpful_votes alohida jadval sifatida -> 3-dars (many-to-many, Integer emas)
# selectinload(CourseReview.student)           -> 5-dars (N+1'dan himoya)
# load_only(...) get_approved_reviews ichida   -> 11-dars (over-fetching'dan himoya)
# submit_review'da try/except IntegrityError   -> 6-dars (tranzaksiya xavfsizligi)
# 2 ta alohida migratsiya (A va B)             -> 8-9-dars (tartib va xavfsizlik)
# async with AsyncSessionLocal() hamma joyda   -> 11-dars (pool tugashining oldini olish)
#
# Bu xarita shunchaki rasmiyat emas — aynan shu darsning topshirig'i talab
# qiladigan yakuniy yozma hisobotning asosini tashkil qiladi.
