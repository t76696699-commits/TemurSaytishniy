Massivlarning qidiruv va o'zgarish metodlari
Урок 14 из 15
· 3 раздела
✓ Пройден
📝
Текст
Текст
#1
indexOf

includes

find

filter

map

reduce

sort

array

index yoki -1

true false

birinchi mos

yangi array

transform

bitta qiymat

saralash

Massivlarning qidiruv va o'zgarish metodlari — map, filter, find
5-darsda biz massivlar bilan ishlashning oddiy usullarini (push, pop) ko'rgan edik. Real loyihalarda esa massiv ichidagi yuzlab ma'lumotlar orasidan keraklisini qidirib topish, saralash yoki ularni o'zgartirish talab etiladi. Buning uchun JavaScript-da juda kuchli va zamonaviy metodlar mavjud.

💻
Код
Код
#2
javascript
 Копировать
let eskiNarxlar = [1000, 2000, 3000];

let yangiNarxlar = eskiNarxlar.map(function(narx) {
    return narx * 2; 
});

console.log(yangiNarxlar); // Natija: [2000, 4000, 6000]
