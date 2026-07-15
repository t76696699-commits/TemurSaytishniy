<!DOCTYPE html>
<html lang="uz">
<head>
    <meta charset="UTF-8">
    <title>Interaktiv Kalkulyator va Rejimlar</title>
    <style>
        body { transition: 0.5s; font-family: sans-serif; padding: 20px; }
        .box { border: 1px solid #ccc; padding: 20px; margin-bottom: 20px; border-radius: 8px; background: rgba(255,255,255,0.1); }
        /* Kalkulyator dizayni */
        .calculator { max-width: 300px; }
        #display { width: 100%; height: 40px; font-size: 20px; text-align: right; margin-bottom: 10px; }
        .buttons { display: grid; grid-template-columns: repeat(4, 1fr); gap: 5px; }
        button { padding: 10px; cursor: pointer; }
        #history { font-size: 12px; color: #555; }
    </style>
</head>
<body>

    <div class="box calculator">
        <h3>Kalkulyator</h3>
        <input type="text" id="display" placeholder="0">
        <div class="buttons">
            <button onclick="clearDisplay()">C</button>
            <button onclick="input('/')">/</button>
            <button onclick="input('*')">*</button>
            <button onclick="input('%')">%</button>
            <button onclick="input('7')">7</button>
            <button onclick="input('8')">8</button>
            <button onclick="input('9')">9</button>
            <button onclick="input('-')">-</button>
            <button onclick="input('4')">4</button>
            <button onclick="input('5')">5</button>
            <button onclick="input('6')">6</button>
            <button onclick="input('+')">+</button>
            <button onclick="input('1')">1</button>
            <button onclick="input('2')">2</button>
            <button onclick="input('3')">3</button>
            <button onclick="calculate()" style="grid-row: span 2;">=</button>
            <button onclick="input('0')" style="grid-column: span 2;">0</button>
        </div>
        <div id="history">Oxirgi 5 ta hisoblash:</div>
    </div>

    <script>
        let display = document.getElementById("display");
        let history = [];

        function input(val) { display.value += val; }
        function clearDisplay() { display.value = ""; }

        function calculate() {
            let expression = display.value;
            // Validatsiya: && va || operatorlari yordamida bo'shlik yoki noto'g'ri belgini tekshirish
            if (expression === "" || expression.includes("/0")) {
                display.value = "Xato!";
                return;
            }

            let result;
            // switch-case orqali amalni aniqlash (oddiy misollar uchun)
            // Kengaytirilgan hisoblash uchun eval() ishlatamiz, lekin % ni alohida ajratamiz
            try {
                if (expression.includes("%")) {
                    result = eval(expression.replace("%", "/100"));
                } else {
                    result = eval(expression);
                }
                
                display.value = result;
                updateHistory(expression + " = " + result);
            } catch {
                display.value = "Xato!";
            }
        }

        function updateHistory(entry) {
            history.unshift(entry);
            if (history.length > 5) history.pop();
            document.getElementById("history").innerHTML = "Oxirgi 5 ta: <br>" + history.join("<br>");
        }

        // Klaviatura qisqa yo'llari
        document.addEventListener("keydown", (e) => {
            if (e.key === "Enter") calculate();
            else if (e.key === "Escape") clearDisplay();
            else if ("0123456789+-*/%".includes(e.key)) input(e.key);
        });
    </script>
</body>
</html>
