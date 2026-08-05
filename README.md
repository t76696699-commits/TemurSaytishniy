📁 1. index.html (Struktura)
HTML
<!DOCTYPE html>
<html lang="uz">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Oddiy Kalkulyator</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>

    <div class="calculator">
        <div id="display" class="calculator-screen">0</div>
        
        <div class="calculator-keys">
            <button class="operator action-btn" data-action="clear">C</button>
            <button class="operator" data-action="sign">±</button>
            <button class="operator" data-action="%">%</button>
            <button class="operator" data-action="divide">÷</button>

            <button data-number="7">7</button>
            <button data-number="8">8</button>
            <button data-number="9">9</button>
            <button class="operator" data-action="multiply">×</button>

            <button data-number="4">4</button>
            <button data-number="5">5</button>
            <button data-number="6">6</button>
            <button class="operator" data-action="subtract">-</button>

            <button data-number="1">1</button>
            <button data-number="2">2</button>
            <button data-number="3">3</button>
            <button class="operator" data-action="add">+</button>

            <button data-number="0" class="span-2">0</button>
            <button data-action="decimal">.</button>
            <button class="operator calculate" data-action="calculate">=</button>
        </div>
    </div>

    <script src="script.js"></script>
</body>
</html>
🎨 2. style.css (Dizayn va Responsive)
CSS
* {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

body {
    background: linear-gradient(135deg, #74b9ff, #0984e3);
    height: 100vh;
    display: flex;
    justify-content: center;
    align-items: center;
}

.calculator {
    background-color: #2d3436;
    border-radius: 16px;
    box-shadow: 0 10px 25px rgba(0, 0, 0, 0.3);
    width: 100%;
    max-width: 320px;
    overflow: hidden;
}

.calculator-screen {
    background-color: #1e272c;
    color: #ffffff;
    font-size: 2.5rem;
    height: 90px;
    padding: 20px;
    text-align: right;
    display: flex;
    align-items: flex-end;
    justify-content: flex-end;
    word-break: break-all;
    word-wrap: break-word;
}

.calculator-keys {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 1px;
    background-color: #b2bec3;
}

.calculator-keys button {
    background-color: #f1f2f6;
    border: none;
    color: #2d3436;
    font-size: 1.25rem;
    height: 65px;
    cursor: pointer;
    transition: background-color 0.2s;
}

.calculator-keys button:hover {
    background-color: #dfe4ea;
}

.calculator-keys button:active {
    background-color: #ced6e0;
}

.calculator-keys .operator {
    background-color: #ffa502;
    color: #fff;
}

.calculator-keys .operator:hover {
    background-color: #ff9f1a;
}

.calculator-keys .action-btn {
    background-color: #ff4757;
    color: #fff;
}

.calculator-keys .action-btn:hover {
    background-color: #ff6b81;
}

.calculator-keys .span-2 {
    grid-column: span 2;
}
⚡ 3. script.js (Mantiq, DOM va Klaviatura)
JavaScript
const display = document.getElementById('display');
const keys = document.querySelector('.calculator-keys');

let displayValue = '0';
let firstValue = null;
let operator = null;
let waitingForSecondValue = false;

function updateDisplay() {
    display.textContent = displayValue;
}

updateDisplay();

keys.addEventListener('click', (event) => {
    const { target } = event;
    if (!target.matches('button')) return;

    if (target.classList.contains('operator')) {
        handleOperator(target.dataset.action);
        updateDisplay();
        return;
    }

    if (target.dataset.number !== undefined) {
        inputNumber(target.dataset.number);
        updateDisplay();
        return;
    }

    if (target.dataset.action === 'decimal') {
        inputDecimal(target.dataset.action);
        updateDisplay();
        return;
    }
});

function inputNumber(num) {
    if (waitingForSecondValue === true) {
        displayValue = num;
        waitingForSecondValue = false;
    } else {
        displayValue = displayValue === '0' ? num : displayValue + num;
    }
}

function inputDecimal() {
    if (waitingForSecondValue) {
        displayValue = '0.';
        waitingForSecondValue = false;
        return;
    }

    if (!displayValue.includes('.')) {
        displayValue += '.';
    }
}

function handleOperator(nextOperator) {
    const inputValue = parseFloat(displayValue);

    if (operator && waitingForSecondValue) {
        operator = nextOperator;
        return;
    }

    if (firstValue === null && !isNaN(inputValue)) {
        firstValue = inputValue;
    } else if (operator) {
        const result = calculate(firstValue, inputValue, operator);
        
        if (result === 'Xato') {
            displayValue = 'Xato';
            firstValue = null;
            operator = null;
            waitingForSecondValue = false;
            return;
        }

        displayValue = `${parseFloat(result.toFixed(7))}`;
        firstValue = result;
    }

    waitingForSecondValue = true;
    operator = nextOperator;

    if (nextOperator === 'clear') {
        clearCalculator();
    }
}

function calculate(first, second, op) {
    if (op === 'add') return first + second;
    if (op === 'subtract') return first - second;
    if (op === 'multiply') return first * second;
    if (op === 'divide') {
        if (second === 0) return 'Xato';
        return first / second;
    }
    return second;
}

function clearCalculator() {
    displayValue = '0';
    firstValue = null;
    operator = null;
    waitingForSecondValue = false;
}

// Klaviatura yordamida boshqarish
window.addEventListener('keydown', (e) => {
    if (!isNaN(e.key)) {
        inputNumber(e.key);
        updateDisplay();
    } else if (e.key === '.') {
        inputDecimal();
        updateDisplay();
    } else if (e.key === '+') {
        handleOperator('add');
        updateDisplay();
    } else if (e.key === '-') {
        handleOperator('subtract');
        updateDisplay();
    } else if (e.key === '*') {
        handleOperator('multiply');
        updateDisplay();
    } else if (e.key === '/') {
        e.preventDefault(); // Brauzerda qidirishni oldini olish
        handleOperator('divide');
        updateDisplay();
    } else if (e.key === 'Enter' || e.key === '=') {
        handleOperator('calculate');
        updateDisplay();
    } else if (e.key === 'Escape' || e.key.toLowerCase() === 'c') {
        clearCalculator();
        updateDisplay();
    }
});
