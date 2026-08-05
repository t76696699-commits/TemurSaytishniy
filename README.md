// 1. Svetofor rangi va mashina tezligi o'zgaruvchilari
let svetoforRangi = "qizil"; // "qizil", "sariq", "yashil" yoki boshqa qiymat berib sinab ko'rishingiz mumkin
let tezlik = 0;              // Mashina tezligi (km/soat)

// 2. Shartlarni tekshirish (If, Else If, Else)
if (svetoforRangi === "qizil") {
    if (tezlik === 0) {
        console.log("To'g'ri, to'xtab turibsiz.");
    } else {
        console.log("Taqiqlangan! Sizga jarima yozildi!");
    }
} 
else if (svetoforRangi === "sariq") {
    if (tezlik > 50) {
        console.log("Tezlikni kamaytiring va to'xtashga tayyorlaning!");
    } else {
        console.log("To'xtab kuting.");
    }
} 
else if (svetoforRangi === "yashil") {
    console.log("Yo'lingiz ochiq, xavfsiz harakatlaning!");
} 
else {
    console.log("Svetofor buzilgan, tartibga soluvchiga qarang!");
}
