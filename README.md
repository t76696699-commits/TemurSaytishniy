<!DOCTYPE html>
<html>
<body>

    <h1 id="sarlavha">Sayt sarlavhasi</h1>
    <p id="matn">Bu matn qismi, tugmani bosib rangini o'zgartiring.</p>
    <button id="tugma">Tungi rejimga o'tish</button>

    <script src="script.js"></script>
</body>
</html>

// Elementlarni tanlab olish
let tugma = document.getElementById("tugma");
let sarlavha = document.getElementById("sarlavha");
let matn = document.getElementById("matn");

// Tugmaga bosish hodisasini biriktirish
tugma.addEventListener("click", function() {
    
    // 1. Sahifa fonini qora qilish
    document.body.style.backgroundColor = "black";
    
    // 2. Sarlavha va matn rangini oq qilish
    sarlavha.style.color = "white";
    matn.style.color = "white";
    
    // 3. Tugma matnini yangilash
    tugma.innerText = "Kunduzgi rejimga o'tish";
});
