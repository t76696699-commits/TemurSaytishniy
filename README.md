<div class="calculator">
    <input type="text" id="display" disabled>
    <div class="buttons">
        <button onclick="clearDisplay()" class="btn-c">C</button>
        <button onclick="appendToDisplay('/')">/</button>
        <button onclick="appendToDisplay('*')">*</button>
        <button onclick="appendToDisplay('7')">7</button>
        <button onclick="appendToDisplay('8')">8</button>
        <button onclick="appendToDisplay('9')">9</button>
        <button onclick="appendToDisplay('-')">-</button>
        <button onclick="appendToDisplay('4')">4</button>
        <button onclick="appendToDisplay('5')">5</button>
        <button onclick="appendToDisplay('6')">6</button>
        <button onclick="appendToDisplay('+')">+</button>
        <button onclick="appendToDisplay('1')">1</button>
        <button onclick="appendToDisplay('2')">2</button>
        <button onclick="appendToDisplay('3')">3</button>
        <button onclick="calculate()" class="btn-equal">=</button>
        <button onclick="appendToDisplay('0')" class="btn-zero">0</button>
    </div>
</div>

.calculator { max-width: 300px; margin: auto; padding: 20px; border: 1px solid #ccc; }
#display { width: 100%; height: 40px; font-size: 20px; text-align: right; }
.buttons { display: grid; grid-template-columns: repeat(4, 1fr); gap: 10px; margin-top: 10px; }
button { padding: 15px; cursor: pointer; }
@media (max-width: 400px) { .calculator { width: 90%; } } 

const display = document.getElementById("display");

function appendToDisplay(input) { display.value += input; }

function clearDisplay() { display.value = ""; }

function calculate() {
    try {
        // Nolga bo'lishni tekshirish
        if (display.value.includes("/0")) {
            display.value = "Xato: Nolga bo'lish";
        } else {
            display.value = eval(display.value);
        }
    } catch {
        display.value = "Xato";
    }
}

// Klaviatura bilan ishlash
document.addEventListener("keydown", (event) => {
    if (event.key >= 0 && event.key <= 9 || event.key === '+' || event.key === '-' || event.key === '*' || event.key === '/') {
        appendToDisplay(event.key);
    } else if (event.key === 'Enter') {
        calculate();
    } else if (event.key === 'Escape') {
        clearDisplay();
    }
});
