# ============================================================
# 1) flush() vs commit() — ID kerak, lekin hali commit qilmoqchi emassiz
# ============================================================
new_course = Course(title="Yangi kurs", instructor_id=2, difficulty_level="Advanced",
                     duration_weeks=4, max_points=100)
db.add(new_course)
await db.flush()          # INSERT bajarildi, ID mavjud, lekin hali ROLLBACK mumkin
print(new_course.id)       # allaqachon mavjud — masalan 501

new_lesson = Lesson(course_id=new_course.id, title="1-dars", order=0)
db.add(new_lesson)
await db.commit()          # ENDI hammasi (course + lesson) BITTA tranzaksiyada yakunlanadi

# ============================================================
# 2) Tranzaksiya atomikligi — hammasi yoki hech nima
# ============================================================
try:
    db.add(Submission(student_id=7, project_id=3, status="pending"))
    await db.flush()
    student = await db.get(Student, 7)
    student.total_points += 50   # xato shu yerda bo'lsa...
    await db.commit()            # ...bu qator HECH QACHON bajarilmaydi
except Exception:
    await db.rollback()          # Submission ham, ball ham bazaga yozilmaydi

# ============================================================
# 3) HAQIQIY misol: exercise_service.py'dagi mustaqil ish birliklari
# ============================================================
async def submit_exercise(db, student_id: int, exercise_id: int, answer: str):
    submission = Submission(student_id=student_id, exercise_id=exercise_id, answer=answer)
    db.add(submission)
    await db.commit()   # asosiy submission — o'z ish birligi, mustaqil commit

    try:
        # streak yangilash — ALOHIDA, kichikroq ish birligi
        await bump_streak(db, student_id)
        await db.commit()
    except Exception:
        await db.rollback()   # faqat streak urinishi bekor bo'ladi, submission qoladi

    return submission

# ============================================================
# 4) IntegrityError — poyga holati (race condition) kutilgan xato sifatida
# ============================================================
from sqlalchemy.exc import IntegrityError

async def award_completion_points(db, student_id: int, lesson_id: int, points: int):
    db.add(LessonCompletion(student_id=student_id, lesson_id=lesson_id))  # UniqueConstraint bor
    student = await db.get(Student, student_id)
    student.total_points += points
    try:
        await db.commit()
    except IntegrityError:
        # Boshqa parallel so'rov bu yerni BIRINCHI bo'lib to'ldirgan —
        # bu XATO emas, kutilgan poyga natijasi.
        await db.rollback()

# ============================================================
# 5) expire_on_commit=False — commit'dan keyin ham atributlar o'qiladi
# ============================================================
from sqlalchemy.ext.asyncio import async_sessionmaker, AsyncSession

AsyncSessionLocal = async_sessionmaker(
    bind=engine, class_=AsyncSession, expire_on_commit=False
)
# expire_on_commit=False BO'LMASA:
#   await db.commit()
#   print(new_course.title)   # -> yana bir SELECT yuboradi (obyekt "eskirgan")
# expire_on_commit=False BILAN:
#   await db.commit()
#   print(new_course.title)   # -> xotiradan, qo'shimcha so'rovsiz

# ============================================================
# 6) Context manager — Session har doim yopilishini kafolatlash
# ============================================================
async def get_db():
    async with AsyncSessionLocal() as session:
        try:
            yield session
        finally:
            await session.close()   # xato bo'lsa ham — ulanish pool'ga qaytadi

# ============================================================
# 7) begin_nested() — faqat bir qismini bekor qilish (SAVEPOINT)
# ============================================================
async def submit_with_optional_bonus_check(db, student_id: int, exercise_id: int):
    submission = Submission(student_id=student_id, exercise_id=exercise_id)
    db.add(submission)
    await db.flush()   # asosiy submission tashqi tranzaksiyada

    try:
        async with db.begin_nested():   # SAVEPOINT ochiladi
            bonus = await check_bonus_eligibility(db, student_id)   # xato berishi mumkin
            if bonus:
                db.add(BonusAward(student_id=student_id, amount=bonus))
    except Exception:
        pass   # faqat SAVEPOINT ichidagi qism bekor bo'ladi — submission qoladi

    await db.commit()   # submission (va muvaffaqiyatli bo'lsa BonusAward) saqlanadi

# ============================================================
# 8) db.get() — identity map orqali qisqa yo'l
# ============================================================
student = await db.get(Student, 7)          # identity map'da bo'lsa — SQL YO'Q
same_student = (await db.execute(
    select(Student).where(Student.id == 7)
)).scalar_one()                              # bu HAR DOIM SQL yuboradi
assert student is same_student               # ikkalasi ham bir xil Python obyekti

# ============================================================
# 9) FastAPI endpoint ichida to'liq tranzaksiya hayotiy tsikli
# ============================================================
@router.post("/exercises/{exercise_id}/submit")
async def submit_exercise_endpoint(
    exercise_id: int, answer: str, student_id: int, db: AsyncSession = Depends(get_db)
):
    is_correct = check_answer(exercise_id, answer)
    submission = ExerciseAttempt(student_id=student_id, exercise_id=exercise_id, is_correct=is_correct)
    db.add(submission)
    await db.commit()   # birinchi ish birligi — javob urinishining o'zi

    if is_correct:
        try:
            await bump_streak(db, student_id)
            await db.commit()   # ikkinchi, mustaqil ish birligi
        except Exception:
            await db.rollback()  # streak muhim emas — javob urinishi allaqachon saqlangan

    return {"correct": is_correct}
# E'tibor bering: agar ikkalasi BITTA tranzaksiyada bo'lganida va
# bump_streak() xato bersa, javob urinishining o'zi ham bekor bo'lardi —
# talaba haqiqiy xatosi bo'lmagan joyda xato ko'rgan bo'lardi.
