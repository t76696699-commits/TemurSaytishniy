💻 dictionary.py
Python
import json
import os

FILENAME = "sozlik.json"


def load_dictionary():
  """Ilova ochilganda JSON fayldan ma'lumotlarni yuklaydi.

  Fayl topilmasa yoki xatolik bo'lsa, boshlang'ich 20+ so'z bilan qaytaradi.
  """
  default_data = {
      "apple": "olma",
      "book": "kitob",
      "computer": "kompyuter",
      "dog": "it",
      "elephant": "fil",
      "flower": "gul",
      "garden": "bog'",
      "house": "uy",
      "ice": "muz",
      "juice": "sharbat",
      "key": "kalit",
      "lamp": "chiroq",
      "mountain": "tog'",
      "notebook": "daftar",
      "orange": "apelsin",
      "pencil": "qalam",
      "queen": "qirolicha",
      "river": "daryo",
      "sun": "quyosh",
      "tree": "dрахt",
      "water": "suv",
  }

  try:
    if not os.path.exists(FILENAME):
      # Fayl mavjud bo'lmasa, dastlabki 20+ so'z bilan yaratib qo'yamiz
      save_dictionary(default_data)
      return default_data

    with open(FILENAME, "r", encoding="utf-8") as file:
      data = json.load(file)
      # Agar fayl bo'sh bo'lsa, default ma'lumotni qaytaramiz
      return data if data else default_data

  except (json.JSONDecodeError, IOError) as e:
    print(f"⚠️ Faylni o'qishda xatolik yuz berdi: {e}")
    print("Standart lug'at ishlatiladi.")
    return default_data


def save_dictionary(dictionary):
  """Lug'at ma'lumotlarini JSON faylga saqlaydi."""
  try:
    with open(FILENAME, "w", encoding="utf-8") as file:
      json.dump(dictionary, file, ensure_ascii=False, indent=4)
  except IOError as e:
    print(f"❌ Faylga yozishda xatolik yuz berdi: {e}")


def add_word(dictionary):
  """Yangi so'z va uning tarjimasini qo'shadi."""
  print("\n--- Yangi so'z qo'shish ---")
  eng = input("Inglizcha so'zni kiriting: ").strip().lower()
  uz = input("O'zbekcha tarjimasini kiriting: ").strip().lower()

  if not eng or not uz:
    print("❌ Maydonlar bo'sh bo'lishi mumkin emas!")
    return

  if eng in dictionary:
    print(
        f"⚠️ '{eng}' so'zi allaqachon mavjud. Tarjimasi: {dictionary[eng]}"
    )
    update = (
        input("Tarjimasini yangilashni xohlaysizmi? (ha/yo'q): ")
        .strip()
        .lower()
    )
    if update != "ha":
      return

  dictionary[eng] = uz
  save_dictionary(dictionary)
  print(f"✅ '{eng} -> {uz}' muvaffaqiyatli saqlandi!")


def search_word(dictionary):
  """So'zni ham kalit (inglizcha), ham qiymat (o'zbekcha) bo'yicha qidiradi."""
  print("\n--- So'z qidirish ---")
  query = input("Qidirilayotgan so'zni kiriting: ").strip().lower()

  if not query:
    print("❌ Qidiruv so'zi kiritilmadi!")
    return

  found = False

  # 1. Inglizcha (kalit) bo'yicha qidirish
  if query in dictionary:
    print(f"🔍 Topildi (Inglizcha): {query} -> {dictionary[query]}")
    found = True

  # 2. O'zbekcha (qiymat) bo'yicha qidirish
  for eng, uz in dictionary.items():
    if query in uz:
      print(f"🔍 Topildi (O'zbekcha): {eng} -> {uz}")
      found = True

  if not found:
    print(f"❌ '{query}</i> bo'yicha hech qanday so'z topilmadi.")


def delete_word(dictionary):
  """Lug'atdan so'zni o'chiradi."""
  print("\n--- So'zni o'chirish ---")
  eng = (
      input("O'chirmoqchi bo'lgan inglizcha so'zni kiriting: ")
      .strip()
      .lower()
  )

  if eng in dictionary:
    del dictionary[eng]
    save_dictionary(dictionary)
    print(f"🗑️ '{eng}' so'zi muvaffaqiyatli o'chirildi!")
  else:
    print(f"❌ '{eng}' so'zi lug'atda topilmadi.")


def show_statistics(dictionary):
  """Lug'at statistikasi (jami so'zlar soni)."""
  print("\n--- Statistikasi ---")
  total_words = len(dictionary)
  print(f"📊 Lug'atdagi jami so'zlar soni: {total_words} ta")


def display_all(dictionary):
  """Lug'atdagi barcha so'zlarni ekranga chiqaradi."""
  print("\n--- Barcha so'zlar ro'yxati ---")
  if not dictionary:
    print("Lug'at bo'sh.")
    return

  for i, (eng, uz) in enumerate(dictionary.items(), 1):
    print(f"{i}. {eng} — {uz}")


def main():
  # Ilova ochilganda fayldan yuklash
  dictionary = load_dictionary()

  while True:
    print("\n==============================")
    print("  📚 INGLIZ-O'ZBEK LUG'ATI")
    print("==============================")
    print("1. So'z qo'shish")
    print("2. So'z qidirish (Inglizcha/O'zbekcha)")
    print("3. So'zni o'chirish")
    print("4. Barcha so'zlarni ko'rish")
    print("5. Statistika")
    print("6. Chiqish")

    choice = input("Tanlovingizni kiriting (1-6): ").strip()

    if choice == "1":
      add_word(dictionary)
    elif choice == "2":
      search_word(dictionary)
    elif choice == "3":
      delete_word(dictionary)
    elif choice == "4":
      display_all(dictionary)
    elif choice == "5":
      show_statistics(dictionary)
    elif choice == "6":
      print("Dasturdan chiqildi. Xayr!")
      break
    else:
      print("❌ Noto'g'ri tanlov! Iltimos, 1 dan 6 gacha raqam kiriting.")


if __name__ == "__main__":
  main()
