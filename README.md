from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.future import select
from sqlalchemy.orm import load_only
from models import Course  # Kurs modeli

# ==========================================
# 1. TO'LIQ OBYEKT YUKLOVCHI VERSIYA (Over-fetching)
# ==========================================
async def get_course_catalog_full(db: AsyncSession, page: int, page_size: int):
    """
    Barcha ustunlarni (shu jumladan og'ir description va text_content'ni) yuklaydi.
    """
    offset = (page - 1) * page_size
    stmt = (
        select(Course)
        .offset(offset)
        .limit(page_size)
    )
    result = await db.execute(stmt)
    return result.scalars().all()


# ==========================================
# 2. OPTIMALLASHTIRILGAN VERSIYA (load_only)
# ==========================================
async def get_course_catalog(db: AsyncSession, page: int, page_size: int):
    """
    Faqat katalog kartochkasi uchun kerakli ustunlarni yuklaydi.
    """
    offset = (page - 1) * page_size
    stmt = (
        select(Course)
        .options(
            load_only(
                Course.id,
                Course.title,
                Course.thumbnail_url,
                Course.difficulty_level,
                Course.duration_weeks
            )
        )
        .offset(offset)
        .limit(page_size)
    )
    result = await db.execute(stmt)
    return result.scalars().all()
