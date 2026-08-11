# ============================================================
# 1) Over-fetching — kerakmagan ustunlarni ham yuklash
# ============================================================
from sqlalchemy.orm import load_only

# XATO NAQSH — kurslar katalogi uchun BARCHA ustunlar yuklanadi:
all_columns_stmt = select(Course).order_by(Course.display_order).limit(20)
# Har bir Course obyektida description, text_content kabi katta ustunlar
# ham keladi — garchi katalog sahifasi faqat title/thumbnail ko'rsatsa ham.

# TO'G'RI NAQSH — faqat kerakli ustunlar:
catalog_stmt = (
    select(Course)
    .options(load_only(Course.id, Course.title, Course.thumbnail_url, Course.difficulty_level))
    .order_by(Course.display_order)
    .limit(20)
)
# Yoki umuman ORM obyektisiz, Core-uslubidagi so'rov (Row qaytadi):
lightweight_stmt = (
    select(Course.id, Course.title, Course.thumbnail_url)
    .order_by(Course.display_order)
    .limit(20)
)

# ============================================================
# 2) Connection pool — sozlash va monitoring
# ============================================================
from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine(
    DATABASE_URL,
    pool_size=10,        # doimiy ulanishlar soni
    max_overflow=5,       # vaqtinchalik qo'shimcha (yuqori yuklamada)
    pool_timeout=30,      # bo'sh ulanishni kutish vaqti (soniya)
    pool_pre_ping=True,   # ulanish ishlatishdan oldin "tirikligini" tekshiradi
)

# ============================================================
# 3) XATO NAQSH — Session yopilmay qolishi (pool tugashiga olib keladi)
# ============================================================
async def leaky_endpoint_BAD(course_id: int):
    session = AsyncSessionLocal()          # context manager ISHLATILMAGAN!
    result = await session.execute(select(Course).where(Course.id == course_id))
    course = result.scalar_one_or_none()
    return course
    # session HECH QACHON yopilmaydi — ulanish pool'da abadiy band bo'lib qoladi.
    # Xato yoki erta return bo'lsa ham xuddi shu muammo takrorlanadi.

# TO'G'RI NAQSH — 6-darsdagi context manager qoidasi:
async def safe_endpoint_GOOD(course_id: int):
    async with AsyncSessionLocal() as session:
        result = await session.execute(select(Course).where(Course.id == course_id))
        return result.scalar_one_or_none()
    # `async with` blokidan chiqishda — xato bo'lsa ham — ulanish pool'ga qaytadi.

# ============================================================
# 4) Uzoq tranzaksiya — pool'ni band qilib qo'yish
# ============================================================
async def slow_transaction_BAD(db):
    async with db.begin():
        courses = (await db.execute(select(Course))).scalars().all()
        for c in courses:
            await slow_external_api_call(c)   # HAR BIR kurs uchun tashqi API — SEKIN!
            c.last_synced_at = datetime.utcnow()
        # Tranzaksiya BUTUN sikl davomida ochiq qoladi — bu vaqtda ulanish
        # boshqa so'rovlar uchun bo'shamaydi.

# TO'G'RI: tashqi chaqiruvlarni tranzaksiyadan TASHQARIDA bajarish
async def fast_transaction_GOOD(db):
    courses = (await db.execute(select(Course))).scalars().all()
    results = [await slow_external_api_call(c) for c in courses]   # tranzaksiyasiz
    async with db.begin():
        for c, synced_at in zip(courses, results):
            c.last_synced_at = synced_at
        # Endi tranzaksiya QISQA — faqat yozish uchun ochiladi.

# ============================================================
# 5) Pool holatini monitoring qilish — muammoni oldindan sezish
# ============================================================
def log_pool_status(engine) -> None:
    pool = engine.pool
    print(
        f"pool size={pool.size()} "
        f"checked_out={pool.checkedout()} "   # hozir band bo'lgan ulanishlar
        f"overflow={pool.overflow()} "         # vaqtinchalik qo'shimcha ulanishlar
        f"checked_in={pool.checkedin()}"        # bo'sh, qayta ishlatishga tayyor
    )
# Agar checked_out doimiy ravishda pool_size'ga teng bo'lib qolsa (hech
# qachon pastga tushmasa) — bu Session'lar yopilmayotganining aniq belgisi.

# ============================================================
# 6) load_only() bilan avval/keyin — taxminiy trafik solishtiruvi
# ============================================================
# Faraz qilaylik: description ~800 belgi, text_content ~4500 belgi.
# 20 ta kurs uchun:
#   load_only() SIZ: 20 * (800 + 4500 + boshqa ustunlar) ≈ 110 000+ belgi
#   load_only() BILAN (faqat id, title, thumbnail_url, difficulty_level):
#       20 * ~150 belgi ≈ 3 000 belgi
# Bu — ro'yxat sahifasida 30 baravargacha kamroq trafik va xotira degani,
# hech qanday funksional yo'qotishsiz (chunki text_content baribir shu
# sahifada ko'rsatilmaydi).

# ============================================================
# 7) Haqiqiy production sozlashiga yaqinlashtirilgan Engine namunasi
# ============================================================
production_engine = create_async_engine(
    DATABASE_URL,
    pool_size=20,           # ko'proq uvicorn worker — ko'proq ulanish kerak
    max_overflow=10,
    pool_timeout=10,        # sukut bo'yichadan qisqaroq — muammoni tezroq bildiradi
    pool_recycle=1800,      # ulanishlarni har 30 daqiqada qayta ochish (baza tomonidan uzilishdan himoya)
    pool_pre_ping=True,
)
# pool_size uvicorn/gunicorn worker soni va PostgreSQL'ning o'zining
# max_connections chegarasi bilan mos kelishi kerak — agar 4 worker *
# pool_size=20 bazaning max_connections'idan oshib ketsa, ilova muammo
# kodga yetib bormasdan turib ulanish xatolarini olishni boshlaydi.

# ============================================================
# 8) OFFSET o'rniga kursor bilan sahifalash — yuklamani kamaytirishning yana bir yo'li
# ============================================================
# Katta jadvalda .offset(10000) PostgreSQL'ga kerakli sahifani qaytarishdan
# oldin 10000 qatorni o'tkazib yuborishga (o'qib, tashlab yuborishga)
# majbur qiladi — bu ham over-fetching'ga qarindosh ortiqcha ish turi.
async def get_courses_after_cursor(db, last_id: Optional[int], page_size: int = 20):
    stmt = select(Course).order_by(Course.id).limit(page_size)
    if last_id is not None:
        stmt = stmt.where(Course.id > last_id)   # OFFSET'siz — to'g'ridan-to'g'ri kerakli joydan
    return (await db.execute(stmt)).scalars().all()
# Kursor bilan sahifalash ayniqsa "cheksiz skroll" API'lari uchun foydali
# — 500-sahifa 1-sahifa kabi tez ishlaydi.
