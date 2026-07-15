# TemurSaytishniy
let yigindi = 0;

// 1 dan 50 gacha bo'lgan sonlar (50 ham kirishi uchun <= ishlatamiz)
for (let i = 1; i <= 50; i++) {
    // Sonning juftligini tekshirish (2 ga bo'lganda qoldiq 0 bo'lishi kerak)
    if (i % 2 === 0) {
        yigindi += i;
    }
}

console.log("1 dan 50 gacha bo'lgan juft sonlar yig'indisi: " + yigindi);
