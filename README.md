💻 password_generator.py
Python
from datetime import datetime
import random
import string


def get_user_preferences():
  """Foydalanuvchidan parol uzunligi va qaysi belgilar qatnashishini so'raydi."""
  print("=== Tasodifiy Parol Generatoriga Xush Kelibsiz! ===")

  # Uzunlikni tekshirib olish (8 dan 32 gacha)
  while True:
    try:
      length = int(input("Parol uzunligini kiriting (8-32): "))
      if 8 <= length <= 32:
        break
      print("Xato! Uzunlik 8 va 32 oralig'ida bo'lishi kerak.")
    except ValueError:
      print("Iltimos, faqat raqam kiriting!")

  # Belgilar turini tanlash
  print("\nParol tarkibini tanlang (ha/yo'q):")
  use_upper = (
      input("Katta harflar ishlatilsinmi? (ha/yo'q): ").strip().lower() == "ha"
  )
  use_digits = (
      input("Raqamlar ishlatilsinmi? (ha/yo'q): ").strip().lower() == "ha"
  )
  use_punctuation = (
      input("Maxsus belgilar (!@#$...) ishlatilsinmi? (ha/yo'q): ").strip()
      == "ha"
  )

  return length, use_upper, use_digits, use_punctuation


def generate_password(length, use_upper, use_digits, use_punctuation):
  """Talablarga mos ravishda tasodifiy parol generatsiya qiladi."""
  # Har doim kichik harflar bazaviy mavjud bo'ladi
  characters = string.ascii_lowercase

  if use_upper:
    characters += string.ascii_letters[
        26:
    ]  # Yoki string.ascii_uppercase
  if use_digits:
    characters += string.digits
  if use_punctuation:
    characters += string.punctuation

  # random.choices yordamida belgilar massividan tasodifiy tanlash
  password_list = random.choices(characters, k=length)
  return "".join(password_list)


def evaluate_strength(password, use_upper, use_digits, use_punctuation):
  """Parolning murakkablik darajasini baholaydi."""
  score = 0
  if len(password) >= 12:
    score += 1
  if use_upper:
    score += 1
  if use_digits:
    score += 1
  if use_punctuation:
    score += 1

  if score <= 1:
    return "Kuchsiz ❌"
  elif score <= 3:
    return "O'rta ⚠️"
  else:
    return "Kuchli 🔥"


def main():
  # 1. Foydalanuvchi sozlamalarini olish
  length, use_upper, use_digits, use_punctuation = get_user_preferences()

  # 2. Parolni yaratish
  password = generate_password(length, use_upper, use_digits, use_punctuation)

  # 3. Kuchini baholash
  strength = evaluate_strength(password, use_upper, use_digits, use_punctuation)

  # 4. Generatsiya vaqtini datetime orqali olish
  generation_time = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

  # 5. Natijalarni ekranga chiqarish
  print("\n" + "=" * 40)
  print(f"🔑 Yaratilgan parol: {password}")
  print(f"💪 Parol kuchi:     {strength}")
  print(f"🕒 Vaqt:            {generation_time}")
  print("=" * 40)


if __name__ == "__main__":
  main()
