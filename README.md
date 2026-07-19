Shahar transport tarmog'ini modellashtirish uchun Adjacency List (qo'shnilik ro'yxati) yordamida Graph klassini va unga oid qidiruv algoritmlarini quyidagicha implementatsiya qilamiz.

1-qism: Graph Klassi
Bu klassda bekatlar Map orqali saqlanadi, har bir bekatning qirralari esa massiv ko'rinishida bo'ladi.

JavaScript
class Graph {
  constructor() {
    this.adjacencyList = new Map();
  }

  addVertex(station) {
    if (!this.adjacencyList.has(station)) {
      this.adjacencyList.set(station, []);
    }
  }

  addEdge(v1, v2, weight) {
    // Ikki yo'nalishli bog'lanish (Weight: masofa)
    this.adjacencyList.get(v1).push({ node: v2, weight });
    this.adjacencyList.get(v2).push({ node: v1, weight });
  }

  removeEdge(v1, v2) {
    this.adjacencyList.set(v1, this.adjacencyList.get(v1).filter(v => v.node !== v2));
    this.adjacencyList.set(v2, this.adjacencyList.get(v2).filter(v => v.node !== v1));
  }

  removeVertex(station) {
    while (this.adjacencyList.get(station).length) {
      const adjacentVertex = this.adjacencyList.get(station).pop().node;
      this.removeEdge(station, adjacentVertex);
    }
    this.adjacencyList.delete(station);
  }

  // --- Qidiruv va Traversal ---

  dfs(start, visited = new Set()) {
    console.log(start);
    visited.add(start);
    for (let neighbor of this.adjacencyList.get(start)) {
      if (!visited.has(neighbor.node)) this.dfs(neighbor.node, visited);
    }
  }

  bfs(start) {
    const queue = [start];
    const visited = new Set([start]);
    while (queue.length > 0) {
      const current = queue.shift();
      console.log(current);
      for (let neighbor of this.adjacencyList.get(current)) {
        if (!visited.has(neighbor.node)) {
          visited.add(neighbor.node);
          queue.push(neighbor.node);
        }
      }
    }
  }

  hasPath(start, end, visited = new Set()) {
    if (start === end) return true;
    visited.add(start);
    for (let neighbor of this.adjacencyList.get(start)) {
      if (!visited.has(neighbor.node)) {
        if (this.hasPath(neighbor.node, end, visited)) return true;
      }
    }
    return false;
  }
}
2-qism: Transport Tarmog'ini Qurish
10 ta shahar va ular orasidagi masofalarni (vazn) qo'shamiz:

JavaScript
const transport = new Graph();
const stations = ["Toshkent", "Samarqand", "Buxoro", "Namangan", "Andijon", 
                  "Xiva", "Termiz", "Navoiy", "Jizzax", "Qarshi"];

stations.forEach(s => transport.addVertex(s));

transport.addEdge("Toshkent", "Samarqand", 300);
transport.addEdge("Toshkent", "Namangan", 250);
transport.addEdge("Samarqand", "Buxoro", 270);
transport.addEdge("Samarqand", "Jizzax", 100);
transport.addEdge("Buxoro", "Navoiy", 110);
transport.addEdge("Navoiy", "Qarshi", 200);
transport.addEdge("Namangan", "Andijon", 70);
transport.addEdge("Termiz", "Qarshi", 350);
transport.addEdge("Xiva", "Buxoro", 450);
transport.addEdge("Jizzax", "Toshkent", 200);
3-qism: Tahlil va Sinov
JavaScript
console.log("--- DFS (Chuqurlik bo'yicha) ---");
transport.dfs("Toshkent");

console.log("\n--- BFS (Kenglik bo'yicha) ---");
transport.bfs("Toshkent");

console.log("\n--- Toshkentdan Xivagacha yo'l bormi? ---");
console.log(transport.hasPath("Toshkent", "Xiva")); // true
