Python kodi (catalog.py)
Python
# 10+ mahsulotdan iborat lug'atlar ro'yxati
products = [
    {"nom": "Smartfon", "narx": 4500000, "soni": 10, "kategoriya": "Elektronika"},
    {"nom": "Noutbuk", "narx": 12000000, "soni": 5, "kategoriya": "Elektronika"},
    {"nom": "Quloqchin", "narx": 300000, "soni": 25, "kategoriya": "Elektronika"},
    {"nom": "Planshet", "narx": 6000000, "soni": 8, "kategoriya": "Elektronika"},
    {"nom": "Stol", "narx": 1500000, "soni": 4, "kategoriya": "Mebel"},
    {"nom": "Kreslo", "narx": 1200000, "soni": 12, "kategoriya": "Mebel"},
    {"nom": "Shkaf", "narx": 3500000, "soni": 3, "kategoriya": "Mebel"},
    {"nom": "Futbolka", "narx": 150000, "soni": 50, "kategoriya": "Kiyim"},
    {"nom": "Jinsi", "narx": 350000, "soni": 30, "kategoriya": "Kiyim"},
    {"nom": "Kurtka", "narx": 800000, "soni": 15, "kategoriya": "Kiyim"},
    {"nom": "Krossovka", "narx": 650000, "soni": 20, "kategoriya": "Oyoq kiyim"},
]

def print_catalog(title, items):
    print(f"\n--- {title} ---")
    for p in items:
        print(f"Nom: {p['nom']:<12} | Narx: {p['narx']:>10,} so'm | Soni: {p['soni']:>2} ta | Kategoriya: {p['kategoriya']}")

# 1. Narx bo'yicha o'suvchi saralash
sorted_asc = sorted(products, key=lambda x: x['narx'])
print_catalog("Narxi bo'yicha o'suvchi tartib", sorted_asc)

# 2. Narx bo'yicha kamayuvchi saralash
sorted_desc = sorted(products, key=lambda x: x['narx'], reverse=True)
print_catalog("Narxi bo'yicha kamayuvchi tartib", sorted_desc)

# 3. Bir nechta mezon bilan saralash (Masalan: narx kamayish tartibida, narxlar teng bo'lsa nom bo'yicha o'sishda)
multi_sorted = sorted(products, key=lambda p: (-p['narx'], p['nom']))
print_catalog("Ko'p mezonli saralash (-narx, nom)", multi_sorted)

# 4. min va max yordamida eng arzon va eng qimmat mahsulotni topish
cheapest = min(products, key=lambda x: x['narx'])
expensive = max(products, key=lambda x: x['narx'])

print(f"\n🏷️ Eng arzon mahsulot: {cheapest['nom']} ({cheapest['narx']:,} so'm)")
print(f"💎 Eng qimmat mahsulot: {expensive['nom']} ({expensive['narx']:,} so'm)")

# 5. filter + lambda bilan kategoriya bo'yicha izlash (Masalan: 'Elektronika' kategoriyasi)
target_category = "Elektronika"
filtered_products = list(filter(lambda x: x['kategoriya'].lower() == target_category.lower(), products))
print_catalog(f"Kategoriya bo'yicha filtr: '{target_category}'", filtered_products)

# 6. map yordamida har bir mahsulotning umumiy qiymatini (narx * soni) hisoblash
total_values = list(map(lambda x: {"nom": x['nom'], "umumiy_qiymat": x['narx'] * x['soni']}, products))

print(f"\n--- Mahsulotlarning umumiy qiymati (Ombordagi jami summa) ---")
for item in total_values:
    print(f"Nom: {item['nom']:<12} | Jami qiymat: {item['umumiy_qiymat']:>12,} so'm")

# Jami katalokdagi tovarlar qiymati
grand_total = sum(map(lambda x: x['narx'] * x['soni'], products))
print(f"\n💰 Ombordagi barcha mahsulotlarning umumiy summasi: {grand_total:,} so'm")
