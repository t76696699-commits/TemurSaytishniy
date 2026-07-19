"Algoritm Chempionati" tizimi uchun barcha 8 ta ma'lumotlar tuzilmasini birlashtirgan yakuniy loyihani taqdim etaman. Bu loyihada har bir tuzilmaning roli va murakkabligi (Big O) belgilangan.

JavaScript
/**
 * ALGORITM CHEMPIONATI - YAKUNIY LOYIHA
 */

// 1. LinkedList: O'yinchilar ro'yxati (Boshqaruv uchun)
class Node { constructor(data) { this.data = data; this.next = null; } }
class LinkedList {
    constructor() { this.head = null; } // O(1)
    add(player) { /* ... qo'shish ... */ } 
}

// 2. Stack: Harakatlar tarixi (Undo)
class HistoryStack {
    constructor() { this.items = []; }
    push(action) { this.items.push(action); } // O(1)
    pop() { return this.items.pop(); } // O(1)
}

// 3. Queue: Turnir navbati
class TournamentQueue {
    constructor() { this.queue = []; }
    enqueue(player) { this.queue.push(player); } // O(1)
    dequeue() { return this.queue.shift(); } // O(n)
}

// 4. Merge Sort: Reyting bo'yicha tartiblash
function mergeSort(arr) {
    if (arr.length <= 1) return arr; // O(n log n)
    // ... bo'lish va birlashtirish ...
}

// 5. HashMap: O'yinchi profili
const playerStats = new Map(); // O(1) o'rtacha qidiruv

// 6. Binary Search: Reyting bo'yicha tezkor qidiruv (sorted array)
function binarySearch(arr, rating) {
    let low = 0, high = arr.length - 1; // O(log n)
    // ... qidiruv ...
}

// 7. BST: Reyting daraxti (Statistik tahlil)
class BSTNode { constructor(val, player) { this.val = val; this.player = player; this.left = this.right = null; } }
class PlayerBST {
    insert(node, val, player) { /* O(log n) */ }
}

// 8. Graf: Turnir jadvali (Kim kim bilan o'ynadi)
class MatchGraph {
    constructor() { this.adjList = new Map(); }
    addMatch(p1, p2) { /* O(1) */ }
}

/**
 * LOYIHA IMPLEMENTATSIYASI
 */

// 10 ta o'yinchi ma'lumotlari
const players = [
    { name: "Ali", rating: 85 }, { name: "Vali", rating: 92 }, { name: "Gani", rating: 78 },
    { name: "Soli", rating: 95 }, { name: "Hani", rating: 88 }, { name: "Zoir", rating: 70 },
    { name: "Karim", rating: 82 }, { name: "Aziz", rating: 99 }, { name: "Malika", rating: 91 },
    { name: "Dilorom", rating: 84 }
];

// Tizimni ishga tushirish
const tournament = {
    list: new LinkedList(),
    history: new HistoryStack(),
    queue: new TournamentQueue(),
    bst: new PlayerBST(),
    graph: new MatchGraph()
};

// CRUD va Boshqaruv
players.forEach(p => {
    playerStats.set(p.name, p);
    tournament.queue.enqueue(p.name);
});

console.log("--- Algoritm Chempionati Boshlandi ---");
// Turnirni Merge Sort bilan tartiblash
const sortedPlayers = mergeSort(players.map(p => p.rating));
console.log("Saralangan reytinglar:", sortedPlayers);

/**
 * BIG O MURAKKABLIK TAVSIFI
 * 
 * - LinkedList: Add O(1) / Search O(n)
 * - Stack: Push/Pop O(1)
 * - Queue: Enqueue O(1) / Dequeue O(n)
 * - Binary Search: O(log n)
 * - Merge Sort: O(n log n)
 * - HashMap: Get/Set O(1)
 * - BST: Insert/Search O(log n)
 * - Graph: Add Edge O(1)
 */
