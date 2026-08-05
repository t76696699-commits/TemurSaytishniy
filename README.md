// 1. O'zgaruvchilarni e'lon qilish
let haftaningKuni = "Dushanba"; // "Dushanba", "Seshanba", "Chorshanba", "Payshanba", "Juma", "Shanba", "Yakshanba"
let imtihonBor = false;       // true yoki false
let vazifaBajarildi = true;   // true yoki false
let darslarQoldirilgan = false; // Dam olish kuni yoki darslar rasman qoldirilganligini tekshirish uchun

let isDarsKuni = false;
let isDamOlishKuni = false;

// 2. 1-qism: Switch-case yordamida kun tartibini aniqlash
switch (haftaningKuni) {
    case "Dushanba":
    case "Chorshanba":
    case "Juma":
        console.log("Bugun asosiy dars kunlari.");
        isDarsKuni = true;
        break;
        
    case "Seshanba":
    case "Payshanba":
        console.log("Bugun amaliyot va laboratoriya kuni.");
        isDarsKuni = true;
        break;
        
    case "Shanba":
    case "Yakshanba":
        console.log("Dam olish kuni.");
        isDamOlishKuni = true;
        break;
        
    default:
        console.log("Noto'g'ri kun kiritildi.");
        break;
}

// 3. 2-qism: Mantiqiy operatorlar (&&, ||) yordamida qo'shimcha shartlarni tekshirish
if (isDarsKuni && imtihonBor) {
    console.log("Darhol imtihon zaliga kiring!");
} 
else if (!imtihonBor && vazifaBajarildi && isDarsKuni) {
    console.log("Siz darsga tayyorsiz, kirishingiz mumkin.");
} 
else if (isDarsKuni && !imtihonBor && !vazifaBajarildi) {
    console.log("Vazifani bajarmaganingiz uchun darsga kiritilmaysiz!");
} 

if (isDamOlishKuni || darslarQoldirilgan) {
    console.log("Miriqib dam oling!");
}
