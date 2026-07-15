<!DOCTYPE html>
<html lang="uz">
<head>
    <meta charset="UTF-8">
    <title>Moliya Menejeri</title>
    <style>
        body { font-family: sans-serif; padding: 20px; max-width: 500px; margin: auto; }
        .balance-card { display: flex; justify-content: space-between; margin-bottom: 20px; }
        .income { color: green; }
        .expense { color: red; }
        ul { list-style: none; padding: 0; }
        li { display: flex; justify-content: space-between; padding: 5px; border-bottom: 1px solid #ddd; }
    </style>
</head>
<body>
    <h2>Jami Balans: <span id="balance">0</span></h2>
    <div class="balance-card">
        <div>Daromad: <span class="income" id="totalIncome">0</span></div>
        <div>Xarajat: <span class="expense" id="totalExpense">0</span></div>
    </div>

    <input type="text" id="desc" placeholder="Tavsif (masalan: Ish haqi)">
    <input type="number" id="amount" placeholder="Summa">
    <select id="type">
        <option value="income">Daromad</option>
        <option value="expense">Xarajat</option>
    </select>
    <button onclick="addTransaction()">Qo'shish</button>

    <h3>Tranzaksiyalar</h3>
    <ul id="list"></ul>

    <script src="app.js"></script>
</body>
</html>
