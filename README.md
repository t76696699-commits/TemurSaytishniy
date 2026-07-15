# TemurSaytishniy
if / else — dastur qaror qabul qiladi
true

false

true

false

if shart

if bloki ishlaydi

else if shart

else if bloki ishlaydi

else bloki ishlaydi

davom

🏆 5 daqiqada g'alaba — svetoforni kod orqali boshqaramiz
Hech narsa o'rnatmaymiz. about:blank oching → F12 → Console. Pastdagilarni yopishtirib, Enter bosing.

BLOKA 1 — Birinchi qaror
let chiroq = "yashil";

if (chiroq === "yashil") {
    console.log("🚶 Yuring!");
} else {
    console.log("🛑 To'xtang!");
}
Natija: 🚶 Yuring!. Endi chiroq'ni o'zgartiring va qaytadan yuring:

chiroq = "qizil";
if (chiroq === "yashil") { console.log("🚶 Yuring!"); } else { console.log("🛑 To'xtang!"); }
Endi 🛑 To'xtang!. Bitta o'zgaruvchi — ikki xil natija. Dasturingiz qaror qabul qildi.

BLOKA 2 — 3 ta variant (else if)
Sariq chiroqni ham qo'shamiz:

let svetofor = "sariq";

if (svetofor === "yashil") {
    console.log("🚶 Yuring!");
} else if (svetofor === "sariq") {
    console.log("⏸ Kuting!");
} else {
    console.log("🛑 To'xtang!");
}
Natija: ⏸ Kuting!. else if zanjirini istalgancha cho'zish mumkin — 4, 5, 10 ta variant.

BLOKA 3 — Interaktiv yosh tekshiruvchi
let yosh = Number(prompt("Yoshingizni kiriting:"));

if (yosh < 0) {
    alert("Bunday yosh bo'lmaydi!");
} else if (yosh < 7) {
    alert("Bog'cha yoshi 🧒");
} else if (yosh < 18) {
    alert("Maktab yoshi 📚");
} else if (yosh < 60) {
    alert("Voyaga yetgan 💼");
} else {
    alert("Hurmatli yosh 🎩");
}
Hayrat: bitta dastur 5 xil natija qaytaradi. Sharti birinchi true bo'lganda to'xtaydi — boshqalarni tekshirmaydi. Tartib MUHIM.

🐛 Ataylab xato — `=` va `===` farqi
Bu — JavaScript'ning eng mashhur tuzog'i. Sinab ko'ring:

let kun = "shanba";

// XATO: bir tenglik = bu o'zlashtirish!
if (kun = "yakshanba") {
    console.log("Dam olish!");
} else {
    console.log("Ish kuni");
}
Natija: Dam olish! — hatto kun = "shanba" bo'lsa ham. Sababi: bitta = — o'zlashtirish ("kunga yakshanba qiymatini ber"), solishtirish emas. Natijada kun o'zgardi va shart "truthy" deb baholandi.

To'g'risi — uchta tenglik:

if (kun === "yakshanba") { ... }   // taqqoslash
Qoida: solishtirishda har doim === ishlating. Yagona = faqat qiymat berish uchun.

Endi tushuntiramiz — if / else if / else
Sintaksis
if (SHART) {
    // shart true bo'lsa — bu blok ishlaydi
} else if (BOSHQA_SHART) {
    // birinchi shart false, ikkinchisi true bo'lsa
} else {
    // hech qaysi shart true bo'lmasa
}
Taqqoslash operatorlari
Operator	Ma'no	Misol
===	Qat'iy teng (tur + qiymat)	5 === 5 → true
!==	Qat'iy teng emas	5 !== "5" → true
> <	Katta / kichik	10 > 5 → true
>= <=	Katta-teng / kichik-teng	5 >= 5 → true
Eslatma: == (ikkita tenglik) — turlarni avtomatik aylantiradi: "5" == 5 → true. Bu xavfli. Har doim === ishlating.

Mantiqiy operatorlar (qisqacha)
&& — VA (ikkalasi ham true bo'lsa)
|| — YOKI (hech bo'lmaganda bittasi true bo'lsa)
! — EMAS (true → false, false → true)
let yosh = 25;
let karta = true;

if (yosh >= 18 && karta === true) {
    console.log("✅ Sotib olishingiz mumkin");
}
Tartib — birinchi true to'xtatadi
JavaScript shartlarni yuqoridan pastga tekshiradi. Birinchi true topilganda — boshqalarini ko'rmaydi.

let ball = 85;

if (ball >= 90) { console.log("A"); }
else if (ball >= 80) { console.log("B"); }   // ← bu ishlaydi
else if (ball >= 70) { console.log("C"); }   // ← tekshirilmaydi ham
else { console.log("D"); }
Qachon if, qachon else if, qachon else?
Faqat if — bitta shart bor, "agar shu bo'lsa qil"
if + else — 2 ta variant ("yashil yoki qizil")
if + else if + else — 3+ variant ("yashil/sariq/qizil")
📌 Bu darsdan keyin siz bilasizki
if shart true bo'lsa — bloki ishlaydi
else if bilan zanjir cho'zilarli, else "boshqa hamma holatda"
Solishtirish — har doim ===, hech qachon =
Birinchi true shart bajariladi, qolganlar tekshirilmaydi
&& (VA), || (YOKI), ! (EMAS) bilan shartlarni birlashtirasiz
💻
Код
Код
#2
javascript
 Копировать
// ═══ DARS 2 — if / else / else if ═══
// about:blank → F12 → Console → pastdagilarni yopishtiring

// ─── BLOKA 1: Birinchi qaror ─────────────────────
let chiroq = "yashil";
if (chiroq === "yashil") {
    console.log("🚶 Yuring!");
} else {
    console.log("🛑 To'xtang!");
}

// O'zgaruvchini almashtiring, qaytadan yuring
chiroq = "qizil";
if (chiroq === "yashil") { console.log("🚶 Yuring!"); }
else { console.log("🛑 To'xtang!"); }

// ─── BLOKA 2: 3 ta variant — else if ─────────────
let svetofor = "sariq";
if (svetofor === "yashil") {
    console.log("🚶 Yuring!");
} else if (svetofor === "sariq") {
    console.log("⏸ Kuting!");
} else {
    console.log("🛑 To'xtang!");
}

// ─── BLOKA 3: Interaktiv yosh tekshiruvchi ───────
let yosh = Number(prompt("Yoshingizni kiriting:"));
if (yosh < 0)        alert("Bunday yosh bo'lmaydi!");
else if (yosh < 7)   alert("Bog'cha yoshi 🧒");
else if (yosh < 18)  alert("Maktab yoshi 📚");
else if (yosh < 60)  alert("Voyaga yetgan 💼");
else                 alert("Hurmatli yosh 🎩");

// ─── XATO TUZOG'I: bitta = vs uchta === ──────────
let kun = "shanba";
if (kun = "yakshanba") console.log("Dam!");       // XATO: o'zlashtirish
else                    console.log("Ish kuni");
// Tuzatish:
if (kun === "yakshanba") console.log("Dam!");     // TO'G'RI: taqqoslash
else                      console.log("Ish kuni");

// ─── Mantiqiy operator (&& va ||) ────────────────
let yoshim = 25, kartam = true;
if (yoshim >= 18 && kartam === true) {
    console.log("✅ Sotib olishingiz mumkin");
}
