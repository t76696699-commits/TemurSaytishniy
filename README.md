# ════════════════════════════════════════════════════════════════════
# DARS 2: Message handler'lar
# ════════════════════════════════════════════════════════════════════

import asyncio
import os
import logging
import re
from dotenv import load_dotenv

from aiogram import Bot, Dispatcher, F, Router
from aiogram.enums import ParseMode
from aiogram.client.default import DefaultBotProperties
from aiogram.filters import Command, CommandStart, CommandObject
from aiogram.types import Message

load_dotenv()
logging.basicConfig(level=logging.INFO)

bot = Bot(
    token=os.getenv("BOT_TOKEN"),
    default=DefaultBotProperties(parse_mode=ParseMode.HTML),
)
dp = Dispatcher()

# Admin user ID'lar
ADMINS = {111111111, 222222222}   # haqiqiy ID'lar bilan almashtiring


# ─────────────────────────────────────────────────────────────────────
# 1) Buyruqlar
# ─────────────────────────────────────────────────────────────────────

@dp.message(CommandStart())
async def cmd_start(m: Message):
    await m.answer(
        f"Salom, <b>{m.from_user.first_name}</b>!\n\n"
        f"Buyruqlar:\n"
        f"/help — yordam\n"
        f"/echo &lt;matn&gt; — echo\n"
        f"/calc &lt;ifoda&gt; — kalkulyator\n"
        f"/ban &lt;user&gt; — admin only"
    )


# Bitta handler — bir nechta buyruq uchun
@dp.message(Command("help", "yordam", "info"))
async def cmd_help(m: Message):
    await m.answer(
        "<b>Yordam</b>\n\n"
        "Quyidagi xabarlarni ham yuborib ko'ring:\n"
        "• Salom — javob keladi\n"
        "• Rahmat — yana javob\n"
        "• Matn ichida 'pizza' — alohida\n"
        "• Sonni yuboring — kvadrat hisoblanadi"
    )


# Arg'lar bilan
@dp.message(Command("echo"))
async def cmd_echo(m: Message, command: CommandObject):
    if not command.args:
        await m.answer("Foydalanish: /echo &lt;matn&gt;")
        return
    await m.answer(f"📣 <b>{command.args}</b>")


# Kalkulyator
@dp.message(Command("calc"))
async def cmd_calc(m: Message, command: CommandObject):
    if not command.args:
        await m.answer("Misol: /calc 2 + 2")
        return
    try:
        # XAVFLI — production'da bunday eval qilmang!
        # Bu faqat demo. Production: ast.literal_eval yoki sympy
        result = eval(command.args, {"__builtins__": {}}, {})
        await m.answer(f"= <code>{result}</code>")
    except Exception as e:
        await m.answer(f"Xato: {e}")


# Admin only
@dp.message(Command("ban"), F.from_user.id.in_(ADMINS))
async def cmd_ban(m: Message, command: CommandObject):
    if not command.args:
        await m.answer("Foydalanish: /ban &lt;user&gt;")
        return
    await m.answer(f"🔨 Ban: {command.args}")


# ─────────────────────────────────────────────────────────────────────
# 2) Matn bilan ishlash — F filter
# ─────────────────────────────────────────────────────────────────────

# Aniq matn
@dp.message(F.text == "Salom")
async def salom_yuborildi(m: Message):
    await m.answer(f"Va alaykum salom, {m.from_user.first_name}! 👋")


# Case-insensitive (lower bilan)
@dp.message(F.text.lower() == "rahmat")
async def rahmat(m: Message):
    await m.answer("Arzimaydi! 🤗")


# Ichida bo'lsa
@dp.message(F.text.lower().contains("pizza"))
async def pizza(m: Message):
    await m.answer("🍕 Pizza haqida gapiryapsizmi?")


# To'plamdan biri
@dp.message(F.text.in_({"ha", "ok", "tasdiqlayman"}))
async def tasdiq(m: Message):
    await m.answer("✅ Tasdiqlandi")


# Boshlangan
@dp.message(F.text.startswith("salom"))
async def salom_boshlangan(m: Message):
    await m.answer("Salom bilan boshlangan xabar 👋")


# ─────────────────────────────────────────────────────────────────────
# 3) Regex
# ─────────────────────────────────────────────────────────────────────

# Faqat son
@dp.message(F.text.regexp(r"^-?\d+$"))
async def son(m: Message):
    n = int(m.text)
    await m.answer(f"Son: {n}\nKvadrat: {n*n}\nKub: {n*n*n}")


# Telefon raqami
@dp.message(F.text.regexp(r"^\+998\d{9}$"))
async def telefon(m: Message):
    await m.answer(f"📞 O'zbekiston telefoni: {m.text}")


# Email
@dp.message(F.text.regexp(r"^[\w._-]+@[\w.-]+\.\w+$"))
async def email(m: Message):
    await m.answer(f"📧 Email: {m.text}")


# ─────────────────────────────────────────────────────────────────────
# 4) Content turlar
# ─────────────────────────────────────────────────────────────────────

@dp.message(F.sticker)
async def sticker(m: Message):
    await m.answer(f"😄 Sticker oldim! ID: <code>{m.sticker.file_id}</code>")


@dp.message(F.photo)
async def photo(m: Message):
    # m.photo — har xil o'lchamdagi versiyalar (eng katta — oxiri)
    photo = m.photo[-1]
    await m.answer(
        f"📷 Rasm oldim!\n"
        f"Eni × bo'yi: {photo.width} × {photo.height}\n"
        f"Hajmi: {photo.file_size} bayt"
    )


@dp.message(F.voice)
async def voice(m: Message):
    await m.answer(f"🎤 Voice: {m.voice.duration} soniya")


@dp.message(F.document)
async def document(m: Message):
    doc = m.document
    await m.answer(
        f"📄 Hujjat: <b>{doc.file_name}</b>\n"
        f"Hajmi: {doc.file_size:,} bayt\n"
        f"Tur: {doc.mime_type}"
    )


@dp.message(F.location)
async def location(m: Message):
    loc = m.location
    await m.answer(
        f"📍 Joylashuv:\n"
        f"Latitude: <code>{loc.latitude}</code>\n"
        f"Longitude: <code>{loc.longitude}</code>"
    )


@dp.message(F.contact)
async def contact(m: Message):
    c = m.contact
    await m.answer(
        f"📞 Kontakt:\n"
        f"Ism: {c.first_name}\n"
        f"Tel: {c.phone_number}"
    )


# ─────────────────────────────────────────────────────────────────────
# 5) Catch-all — ENG OXIRDA
# ─────────────────────────────────────────────────────────────────────

@dp.message()
async def catch_all(m: Message):
    await m.answer(
        f"🤔 Bunaqa xabarni qanday qaytarishni bilmayman.\n"
        f"<i>/help</i> bosing yoki matn yozing."
    )


# ─────────────────────────────────────────────────────────────────────
# 6) Error handler
# ─────────────────────────────────────────────────────────────────────

@dp.error()
async def error_handler(event):
    logging.error(f"Exception: {event.exception}", exc_info=True)
    return True


# ─────────────────────────────────────────────────────────────────────
async def main():
    await bot.delete_webhook(drop_pending_updates=True)
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())
