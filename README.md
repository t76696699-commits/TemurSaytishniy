<!DOCTYPE html>
<html lang="uz">
<body>

    <input type="text" id="ismInput" placeholder="Ismingizni kiriting...">
    <button id="qoshishBtn">Qo'shish</button>
    
    <ul id="royxat"></ul>

    <script src="script.js"></script>
</body>
</html>

// 1. Elementlarni o'zgaruvchilarga tutib olish
let input = document.getElementById("ismInput");
let qoshishBtn = document.getElementById("qoshishBtn");
let ul = document.getElementById("royxat");

// 2. Tugmaga hodisa (event) biriktirish
qoshishBtn.addEventListener("click", function() {
    let matn = input.value;

    // Agar input bo'sh bo'lmasa, element yaratamiz
    if (matn !== "") {
        // Yangi <li> elementi yaratish
        let li = document.createElement("li");
        li.innerText = matn + " "; // Ismni qo'shish

        // O'chirish tugmasini yaratish
        let ochirishBtn = document.createElement("button");
        ochirishBtn.innerText = "X";

        // O'chirish tugmasiga bosilganda elementni olib tashlash
        ochirishBtn.addEventListener("click", function() {
            li.remove();
        });

        // Tugmani <li> ichiga, <li> ni esa <ul> ichiga joylash
        li.appendChild(ochirishBtn);
        ul.appendChild(li);

        // 3. Inputni tozalash
        input.value = "";
    }
});
