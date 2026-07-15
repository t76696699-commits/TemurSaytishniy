let mashina = {
    brend: "Chevrolet",
    model: "Gentra",
    yil: 2022,
    narx: 13000,
    tanirovka: false,

    // Malumotlarni ekranga chiqarish metodi
    malumotBering: function() {
        console.log(`Brend: ${this.brend}, Model: ${this.model}, Narxi: ${this.narx}$`);
    },

    // Tanirovka qildirish va narxni oshirish metodi
    tanirovkaQildir: function() {
        this.tanirovka = true;
        this.narx += 500;
        console.log("Tanirovka o'rnatildi va narx yangilandi.");
    }
};

// 1. Obyektga tashqaridan rang xususiyatini qo'shish
mashina.rang = "To'q kulrang";

// 2. Obyektga tashqaridan narxni yangilash
mashina.narx = 12500;

// 3. Metodlarni chaqirish va natijani tekshirish
mashina.malumotBering();      // Dastlabki holat
mashina.tanirovkaQildir();    // Tanirovka qilish
mashina.malumotBering();      // O'zgarishdan keyingi holat

// 4. Oxirgi obyektni konsolga to'liq chiqarish
console.log(mashina);
