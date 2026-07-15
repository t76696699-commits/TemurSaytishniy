# TemurSaytishniy
// ── Modul 1 takrorlash: uchta darsni birlashtirgan kichik dastur ────────
//
// Ssenariy: kafe buyurtmasi. Foydalanuvchi ovqat turi va to'lov usulini
// tanlaydi. Dastur yetarli pul borligini tekshiradi va xabar chiqaradi.

const SOLIH_NARX = 25000;      // const — o'zgarmaydigan narx
let buyurtmaTuri = "lavash";   // string
let toLovUsuli  = "naqd";      // string
let pulMiqdori  = 30000;       // number — hamyondagi pul
let bonusKarta  = false;       // boolean — chegirma kartasi bormi?

// 1) if / else if / else + mantiqiy operatorlar
if (buyurtmaTuri === "lavash" && (pulMiqdori >= SOLIH_NARX || bonusKarta)) {
    console.log("Buyurtmangiz qabul qilindi.");
} else if (!bonusKarta && pulMiqdori < SOLIH_NARX) {
    console.log("Mablag' yetarli emas.");
} else {
    console.log("Boshqa taom tanlang.");
}

// 2) switch-case — to'lov usuli bo'yicha xabar
switch (toLovUsuli) {
    case "naqd":
        console.log("Naqd to'lov qabul qilindi.");
        break;
    case "karta":
    case "click":
    case "payme":
        // Bir nechta case birlashtirilgan — barchasi onlayn to'lov
        console.log("Onlayn to'lov qayta ishlanmoqda...");
        break;
    default:
        console.log("Noma'lum to'lov usuli.");
}
// Kutilgan natija:
//   "Buyurtmangiz qabul qilindi."
//   "Naqd to'lov qabul qilindi."
