<!DOCTYPE html>
<html lang="uz">
<head>
    <meta charset="UTF-8">
    <title>To-Do List App</title>
    <style>
        body { font-family: sans-serif; padding: 20px; }
        .completed { text-decoration: line-through; color: gray; }
        ul { list-style: none; padding: 0; }
        li { display: flex; align-items: center; gap: 10px; margin-bottom: 5px; }
    </style>
</head>
<body>

    <h2>Vazifalar ro'yxati</h2>
    <input type="text" id="taskInput" placeholder="Yangi vazifa...">
    <button id="addBtn">Qo'shish</button>

    <div>
        <button onclick="filterTasks('all')">Barchasi</button>
        <button onclick="filterTasks('completed')">Bajarilgan</button>
        <button onclick="filterTasks('pending')">Kutilayotgan</button>
    </div>

    <ul id="taskList"></ul>
    <p id="emptyMsg">Hozircha vazifalar yo'q, dam olish vaqti! 😊</p>

    <script>
        let tasks = JSON.parse(localStorage.getItem("tasks")) || [];
        let currentFilter = 'all';

        const input = document.getElementById("taskInput");
        const addBtn = document.getElementById("addBtn");
        const list = document.getElementById("taskList");
        const emptyMsg = document.getElementById("emptyMsg");

        function saveAndRender() {
            localStorage.setItem("tasks", JSON.stringify(tasks));
            render();
        }

        function render() {
            list.innerHTML = "";
            let filtered = tasks.filter(t => {
                if (currentFilter === 'completed') return t.done;
                if (currentFilter === 'pending') return !t.done;
                return true;
            });

            emptyMsg.style.display = tasks.length === 0 ? "block" : "none";

            filtered.forEach((t, index) => {
                let li = document.createElement("li");
                li.innerHTML = `
                    <input type="checkbox" ${t.done ? "checked" : ""} onchange="toggle(${tasks.indexOf(t)})">
                    <span class="${t.done ? 'completed' : ''}">${t.text}</span>
                    <button onclick="remove(${tasks.indexOf(t)})">O'chirish</button>
                `;
                list.appendChild(li);
            });
        }

        addBtn.onclick = () => {
            if (input.value.trim()) {
                tasks.push({ text: input.value, done: false });
                input.value = "";
                saveAndRender();
            }
        };

        input.onkeypress = (e) => { if (e.key === "Enter") addBtn.onclick(); };

        function toggle(i) { tasks[i].done = !tasks[i].done; saveAndRender(); }
        function remove(i) { tasks.splice(i, 1); saveAndRender(); }
        function filterTasks(f) { currentFilter = f; render(); }

        render();
    </script>
</body>
</html>
