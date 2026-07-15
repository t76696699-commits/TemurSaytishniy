# TemurSaytishniy
// ── 1. Mantiqiy operatorlar ─────────────────────────────────────────────

let pul = true;
let karta = false;

// YOKI (||) — bittasi to'g'ri bo'lsa yetarli
if (pul || karta) {
    console.log("Xarid qilishingiz mumkin.");
}

// VA (&&) — ikkalasi ham true bo'lishi kerak
let bilet = true;
let pasport = true;
if (bilet && pasport) {
    console.log("Samolyotga chiqing.");
}

// EMAS (!) — shartni teskariga aylantiradi
let yomgir = false;
if (!yomgir) {
    console.log("Ko'chaga chiqishimiz mumkin.");
}

// Qat'iy taqqoslash (===) qiymat + turni birga tekshiradi
console.log("10" == 10);   // true  (== — turini o'zgartiradi)
console.log("10" === 10);  // false (=== — turi ham muhim)


// ── 2. Switch-case ─────────────────────────────────────────────────────

let kunRaqami = 3;

switch (kunRaqami) {
    case 1:
        console.log("Dushanba");
        break;
    case 2:
        console.log("Seshanba");
        break;
    case 3:
        console.log("Chorshanba");
        break;
    case 4:
        console.log("Payshanba");
        break;
    case 5:
        console.log("Juma");
        break;
    case 6:
    case 7:
        // Bir nechta case'ni birlashtirish — break yo'q, pastga "tushadi"
        console.log("Dam olish kuni");
        break;
    default:
        // Hech qaysi case mos kelmasa shu blok ishlaydi
        console.log("Noto'g'ri kun raqami (1-7 bo'lishi kerak)");
}
// Natija: "Chorshanba"
