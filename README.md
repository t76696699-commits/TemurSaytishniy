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
