<!DOCTYPE html>
<html lang="uz">
<head>
    <meta charset="UTF-8">
    <title>Pagination Loyihasi</title>
    <style>
        .cards { display: flex; gap: 10px; margin-bottom: 20px; }
        .card { padding: 20px; border: 1px solid #333; border-radius: 5px; }
        .nav { margin-top: 10px; }
        button:disabled { opacity: 0.5; cursor: not-allowed; }
    </style>
</head>
<body>

    <div id="cardsContainer" class="cards"></div>
    
    <div class="nav">
        <button id="prevBtn">Oldinga</button>
        <span id="pageNumbers"></span>
        <button id="nextBtn">Orqaga</button>
    </div>

    <script>
        const data = ["1-Ma'lumot", "2-Ma'lumot", "3-Ma'lumot", "4-Ma'lumot", "5-Ma'lumot", "6-Ma'lumot", "7-Ma'lumot"];
        let currentPage = 1;
        const itemsPerPage = 3;

        function render() {
            const container = document.getElementById("cardsContainer");
            const pageNumbers = document.getElementById("pageNumbers");
            container.innerHTML = "";
            
            // 1. Kartochkalarni render qilish (for tsikli)
            let start = (currentPage - 1) * itemsPerPage;
            let end = start + itemsPerPage;
            for (let i = start; i < end && i < data.length; i++) {
                let div = document.createElement("div");
                div.className = "card";
                div.innerText = data[i];
                container.appendChild(div);
            }

            // 2. Sahifa raqamlarini render qilish
            pageNumbers.innerText = ` Sahifa: ${currentPage} `;
            document.getElementById("prevBtn").disabled = currentPage === 1;
            document.getElementById("nextBtn").disabled = currentPage >= Math.ceil(data.length / itemsPerPage);
        }

        document.getElementById("prevBtn").addEventListener("click", () => { currentPage--; render(); });
        document.getElementById("nextBtn").addEventListener("click", () => { currentPage++; render(); });

        render();
    </script>
</body>
</html>
