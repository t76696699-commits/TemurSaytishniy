<!DOCTYPE html>
<html>
<body>

    <h1 id="hisob">0</h1>
    <button id="oshirishBtn">Oshirish (+1)</button>

    <script src="script.js"></script>
</body>
</html>

let hisobMatn = document.getElementById("hisob");
let tugma = document.getElementById("oshirishBtn");

// 1. Sahifa yuklanganda LocalStorage-dan eski qiymatni olish
// Agar qiymat bo'lmasa, 0 dan boshlaymiz
let ochko = localStorage.getItem("hisobQiymati") ? parseInt(localStorage.getItem("hisobQiymati")) : 0;

// Sahifa yuklanishi bilan ekranga oxirgi saqlangan qiymatni chiqarish
hisobMatn.innerText = ochko;

// 2. Tugmaga bosilganda amalga oshiriladigan ishlar
tugma.addEventListener("click", function() {
    ochko++; // Ochkoni oshirish
    
    // Ekranni yangilash
    hisobMatn.innerText = ochko;
    
    // LocalStorage-ga yangi qiymatni saqlash
    localStorage.setItem("hisobQiymati", ochko);
});
