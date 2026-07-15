let oyinchi = {
    ism: "Ali",
    ochko: 0,

    // Ochkoni 10 taga oshiruvchi metod
    ochkoOshir: function() {
        this.ochko += 10;
        console.log(`${this.ism}ning yangi ochkosi: ${this.ochko}`);
    }
};

// Metodni chaqirib ko'ramiz
oyinchi.ochkoOshir(); // Natija: 10
oyinchi.ochkoOshir(); // Natija: 20 
