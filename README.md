// 1. Umumiy summani hisoblovchi funksiya
function hisobla(mahsulotNarxi, miqdori) {
    return mahsulotNarxi * miqdori;
}

// 2. Chegirma hisoblovchi funksiya
function chegirmaBer(umumiySumma) {
    if (umumiySumma > 100000) {
        // 10% chegirma hisoblash (umumiy summaning 90% qismini qaytaradi)
        return umumiySumma * 0.9;
    } else {
        // Chegirmasiz asl narxni qaytarish
        return umumiySumma;
    }
}

// 3. Funksiyalarni ketma-ket chaqirib, natijani konsolga chiqarish
// Foydalanuvchi 3 ta 40 000 so'mlik mahsulot olgani:
let narx = 40000;
let miqdor = 3;

// Avval hisobla funksiyasi orqali umumiy summa topiladi, keyin chegirmaBer funksiyasiga uzatiladi
let umumiy = hisobla(narx, miqdor);
let yakuniyTolov = chegirmaBer(umumiy);

console.log("Yakuniy to'lov miqdori: " + yakuniyTolov + " so'm");
