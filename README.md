 LocalStorage
Урок 13 из 15
· 3 раздела
✓ Пройден
📝
Текст
Текст
#1
setItem k v

getItem k

removeItem k

JSON.stringify

JSON.parse

localStorage

browser save

string yoki null

ochiriladi

object array

string

object qaytadi

LocalStorage — Ma'lumotlarni brauzer xotirasida saqlash
Shu vaqtgacha biz yaratgan dasturlarda (masalan, "Mehmonlar ro'yxati" yoki "Vazifalar ro'yxati" loyihalarida) bitta muammo bor edi: agar brauzer sahifasini yangilasak (Refresh qilsak), barcha kiritilgan ma'lumotlar o'chib ketardi.

Bugun ma'lumotlarni foydalanuvchi kompyuterida (brauzerida) doimiy saqlashni o'rganamiz. Buning uchun LocalStorage (Mahalliy xotira) ishlatiladi. LocalStorage-ga yozilgan ma'lumotlar brauzer yopilsa ham, kompyuter o'chirib yoqilsa ham o'chib ketmaydi.

💻
Код
Код
#2
javascript
 Копировать
// 1. Ismni xotiraga "foydalanuvchi" kaliti bilan saqlaymiz
localStorage.setItem("foydalanuvchi", "Sardor");

// 2. Sahifa yangilanganda xotiradan o'sha ismni qayta o'qib olamiz
let ism = localStorage.getItem("foydalanuvchi");

console.log(ism); // Natija: Sardor
