Tranzaksiyalar va Sessiyalar: Unit of Work
Урок 7 из 14
· 3 раздела
✓ Пройден
📝
Matn
Matn
#1
Session — bu shunchaki "ulanish" emas, "ish birligi"
Yangi boshlovchilar ko'pincha Sessionni bazaga ulanish (connection) bilan adashtiradi. Aslida Session — Unit of Work (ish birligi) naqshini amalga oshiradi: u sizning barcha o'zgarishlaringizni (qo'shilgan, o'zgartirilgan, o'chirilgan obyektlar) xotirada kuzatib boradi va faqat commit() chaqirilganda ularning barchasini BITTA tranzaksiyada bazaga yuboradi. Bu platformada har bir HTTP so'rovi o'zining Session'iga ega — app/dependencies.py'dagi get_db() orqali.

flush() vs commit() — ikki xil "bazaga yuborish"
flush() — Session'dagi kutilayotgan o'zgarishlarni bazaga YUBORADI (INSERT/UPDATE bajaradi), lekin tranzaksiyani YAKUNLAMAYDI — hali ROLLBACK qilish mumkin. commit() esa avval avtomatik flush qiladi, SO'NGRA tranzaksiyani COMMIT bilan yakunlaydi — bundan keyin o'zgarishlarni qaytarib bo'lmaydi. Yangi qo'shilgan obyektning ID'sini commit'dan OLDIN bilish kerak bo'lsa (masalan boshqa obyektga FOREIGN KEY sifatida berish uchun), await db.flush() yetarli — butun tranzaksiyani yakunlash shart emas.

Tranzaksiya chegaralari — bitta so'rov, bitta "hammasi yoki hech nima"
107-kursda BEGIN; ... COMMIT; orqali ko'rgan atomiklik tushunchasi ORM'da ham xuddi shunday ishlaydi: agar bitta so'rovda 3 ta jadvalga yozish kerak bo'lsa (masalan yangi Submission + Student ballarini yangilash + Achievement tekshiruvi) va uchinchisi xato bersa, birinchi ikkitasi ham COMMIT bo'lmasligi kerak. Bu platformaning haqiqiy exercise_service.py'sida bu naqsh aniq ko'rinadi: agar bump_streak() xato bersa, faqat SHU qo'shimcha amal rollback() qilinadi, asosiy submission esa allaqachon alohida commit qilingan bo'ladi — bu ataylab qilingan qaror, chunki ikkalasi mustaqil ish birligi hisoblanadi.

IntegrityError va rollback — poyga holatlarini (race condition) boshqarish
Bir vaqtning o'zida ikkita so'rov bir xil UniqueConstraint'ni buzishga urinishi mumkin (masalan ikkala so'rov ham "bu darsni birinchi marta tugatdim" deb his qilib, ball qo'shishga urinadi). Bunday holatda PostgreSQL IntegrityError tashlaydi — bu platformada bu holat maxsus try/except IntegrityError: await db.rollback() orqali ushlanadi: "ikkinchi so'rov yutqazdi" degani, va bu XATO emas, kutilgan poyga holati natijasi. rollback — Session'ni "toza" holatga qaytaradi, keyingi operatsiyalar buzilgan tranzaksiya holatida davom etmaydi.

expire_on_commit=False — nega bu platformada ishlatiladi
Standart holatda, commit()dan keyin Session barcha obyektlarni "eskirgan" (expired) deb belgilaydi — keyingi murojaatda ularni QAYTA bazadan o'qiydi. Bu ba'zan kerak, lekin FastAPI'da endpoint commit()dan keyin obyekt atributlarini javobga qo'shishi kerak bo'lganda muammo tug'diradi (Session allaqachon yopilgan bo'lishi mumkin). Shuning uchun async_sessionmaker(..., expire_on_commit=False) ishlatiladi — commit'dan keyin ham obyekt atributlariga xotiradan (bazaga qayta murojaatsiz) kirish mumkin bo'ladi.

Context manager — Session har doim yopilishini kafolatlash
async with AsyncSessionLocal() as session: — bu blok tugaganda (xato bo'lsa ham) Session avtomatik yopiladi. Bu — 12-darsda ko'radigan "connection pool tugashi" muammosining oldini oluvchi eng muhim qoida: agar Session qo'lda yopilmasa (masalan try/finally'siz), u connection pool'dan bitta ulanishni abadiy egallab qoladi.

Ichma-ich tranzaksiyalar — begin_nested() va SAVEPOINT
Ba'zan katta tranzaksiya ichida faqat bir qismini bekor qilish kerak bo'ladi, hammasini emas. PostgreSQL'ning SAVEPOINT mexanizmi buni imkon beradi, ORM darajasida esa async with db.begin_nested(): orqali ishlatiladi. Masalan: asosiy submission saqlanadi, so'ngra ixtiyoriy "bonus tekshiruvi" ichki savepoint ichida ishga tushiriladi — agar bonus tekshiruvi xato bersa, faqat SHU qism bekor bo'ladi, asosiy submission esa tashqi tranzaksiyada saqlanib qoladi.

Session hayot siklining vizual sxemasi
yo'q

ha

bonus xato bersa

obyektlarni qo'shish/o'zgartirish
(xotirada)

await db.flush()
INSERT/UPDATE bajariladi,
lekin qaytarish mumkin

xato bormi?

await db.commit()
COMMIT — qaytarib bo'lmaydi

await db.rollback()
Session tozalanadi

begin_nested():
ichki SAVEPOINT
(masalan bonus tekshiruvi)

faqat SAVEPOINT rollback —
asosiy submission saqlanib qoladi

Diagramma Session'ning to'liq hayot siklini ko'rsatadi: xotiradagi o'zgarishlar avval flush() bilan bazaga yuboriladi (hali qaytarish mumkin), so'ngra commit() yoki rollback() bilan yakunlanadi; exercise_service.py'dagi haqiqiy naqshga o'xshab, ichki begin_nested() SAVEPOINT'i asosiy tranzaksiyani buzmasdan faqat qo'shimcha amalni bekor qilish imkonini beradi.

db.get() vs select().where(id ==) — bir xil natija, ikki yo'l
await db.get(Student, 7) — bu PRIMARY KEY bo'yicha qidiruv uchun maxsus qisqartma: agar shu ID Session'ning identity map'ida allaqachon bo'lsa, ORM hatto bazaga so'rov HAM yubormaydi (0-darsdagi identity map misolini eslang). select(Student).where(Student.id == 7) esa har doim so'rov yuboradi, hatto obyekt xotirada bo'lsa ham — chunki bu umumiy so'rov mexanizmi, identity map'ni "qisqa yo'l" sifatida ishlatmaydi.

💻
Kod
Kod
#2
python
 Nusxalash
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
