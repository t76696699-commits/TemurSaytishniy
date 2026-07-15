<style>
    body { transition: background-color 0.5s ease; display: flex; flex-direction: column; align-items: center; justify-content: center; height: 100vh; margin: 0; }
    .container { text-align: center; background: white; padding: 20px; border-radius: 10px; box-shadow: 0 4px 10px rgba(0,0,0,0.1); }
    .buttons { display: grid; grid-template-columns: repeat(3, 1fr); gap: 10px; margin-top: 20px; }
    button { padding: 10px; cursor: pointer; border: none; border-radius: 5px; }
    #hex-code { font-size: 24px; font-weight: bold; margin-bottom: 10px; }
</style>

<div class="container">
    <div id="hex-code">#FFFFFF</div>
    <div class="buttons">
        <button onclick="changeColor('#FF5733')" style="background: #FF5733;">Qizil</button>
        <button onclick="changeColor('#33FF57')" style="background: #33FF57;">Yashil</button>
        <button onclick="changeColor('#3357FF')" style="background: #3357FF;">Ko'k</button>
        <button onclick="changeColor('#F1C40F')" style="background: #F1C40F;">Sariq</button>
        <button onclick="changeColor('#8E44AD')" style="background: #8E44AD;">Binafsha</button>
        <button onclick="changeColor('#E67E22')" style="background: #E67E22;">To'q sariq</button>
        <button onclick="randomColor()" style="grid-column: span 3; background: #333; color: white;">Tasodifiy rang</button>
    </div>
</div>

const hexDisplay = document.getElementById("hex-code");

// Rangni o'zgartirish va saqlash funksiyasi
function changeColor(color) {
    document.body.style.backgroundColor = color;
    hexDisplay.innerText = color;
    localStorage.setItem("oxirgiRang", color);
}

// Tasodifiy rang yaratish funksiyasi
function randomColor() {
    const randomHex = '#' + Math.floor(Math.random()*16777215).toString(16).padStart(6, '0');
    changeColor(randomHex);
}

// Sahifa yuklanganda saqlangan rangni tiklash
window.onload = () => {
    const savedColor = localStorage.getItem("oxirgiRang");
    if (savedColor) changeColor(savedColor);
};
