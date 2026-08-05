Mana barcha talablar va cheklovlarga (for siklisiz, forEach + push ishlatmasdan, faqat map, filter, reduce zanjirlari va boshqa massiv metodlari orqali) to'liq javob beradigan talabalar dashboard moduli uchun toza JavaScript kodi.

💻 students-dashboard.js
JavaScript
// 1. 15+ talabadan iborat boshlang'ich ma'lumotlar massivi
const students = [
    { id: 1, name: "Jasur", age: 21, profession: "Frontend", score: 88, tags: ["js", "react", "css"] },
    { id: 2, name: "Madina", age: 22, profession: "Backend", score: 95, tags: ["python", "flask", "sql"] },
    { id: 3, name: "Aziz", age: 20, profession: "Frontend", score: 76, tags: ["html", "css", "js"] },
    { id: 4, name: "Zaynab", age: 23, profession: "Fullstack", score: 92, tags: ["js", "node", "react"] },
    { id: 5, name: "Sardor", age: 19, profession: "Backend", score: 64, tags: ["python", "sql"] },
    { id: 6, name: "Malika", age: 24, profession: "UI/UX", score: 89, tags: ["figma", "css"] },
    { id: 7, name: "Bekzod", age: 21, profession: "Frontend", score: 82, tags: ["js", "vue"] },
    { id: 8, name: "Gulnora", age: 25, profession: "Backend", score: 98, tags: ["python", "django", "sql"] },
    { id: 9, name: "Bobur", age: 22, profession: "Fullstack", score: 71, tags: ["js", "node"] },
    { id: 10, name: "Sevara", age: 20, profession: "UI/UX", score: 85, tags: ["figma"] },
    { id: 11, name: "Timur", age: 23, profession: "Frontend", score: 90, tags: ["js", "react"] },
    { id: 12, name: "Dildora", age: 21, profession: "Backend", score: 79, tags: ["python", "flask"] },
    { id: 13, name: "Otabek", age: 26, profession: "Fullstack", score: 94, tags: ["js", "node", "react"] },
    { id: 14, name: "Nargiza", age: 22, profession: "UI/UX", score: 68, tags: ["figma", "html"] },
    { id: 15, name: "Javohir", age: 20, profession: "Frontend", score: 81, tags: ["js", "vue", "css"] }
];


// --- 1. KAMIDA 3 TA MAP + FILTER ZANJIRI ---

// 1-zanjir: Balli 80 dan yuqori bo'lgan talabalarning ismlarini katta harflarda olish
const highScorerNames = students
    .filter(student => (student?.score ?? 0) > 80)
    .map(student => (student?.name ?? "").toUpperCase());

// 2-zanjir: Backend kasbidagilarning yoshini aniqlab, ularga 1 yosh qo'shib yangi obyektlar massivini qaytarish
const updatedBackendStudents = students
    .filter(student => student?.profession === "Backend")
    .map(student => ({
        ...student,
        age: (student?.age ?? 0) + 1,
        status: "Senior-track"
    }));

// 3-zanjir: 'react' tegi mavjud bo'lgan talabalarning ballaridan iborat massiv yaratish va ularni tartiblash
const reactStudentScores = students
    .filter(student => Array.isArray(student?.tags) && student.tags.includes("react"))
    .map(student => student?.score ?? 0);


// --- 2. KAMIDA 2 TA REDUCE (Group by, Jami, Count) ---

// 1-reduce: Kasblar bo'yicha guruhlash (Group by profession) va har bir guruhdagi talabalar sonini (count) hisoblash
const studentsByProfession = students.reduce((acc, student) => {
    const prof = student?.profession || "Unknown";
    acc[prof] = (acc[prof] || 0) + 1;
    return acc;
}, {});

// 2-reduce: Talabalarning umumiy ballari yig'indisi (Jami - Total score) va o'rtacha qiymatini topish
const scoreSummary = students.reduce((acc, student) => {
    return {
        totalScore: acc.totalScore + (student?.score ?? 0),
        count: acc.count + 1
    };
}, { totalScore: 0, count: 0 });

const averageScore = scoreSummary.count > 0 ? scoreSummary.totalScore / scoreSummary.count : 0;


// --- 3. FIND, SOME, EVERY METODLARI ---

// find: Bali aniq 98 ga teng bo'lgan birinchi talabani topish
const topStudent = students.find(student => (student?.score ?? 0) === 98) ?? null;

// some: Yosh 25 dan katta bo'lgan talaba bormi yoki yo'qligini tekshirish
const hasOlderStudents = students.some(student => (student?.age ?? 0) > 25);

// every: Barcha talabalarning yoshi 18 dan katta yoki teng ekanligini tekshirish
const isEveryoneAdult = students.every(student => (student?.age ?? 0) >= 18);


// --- 4. BIR NECHTA MEZON BILAN SORT (Key tuple-pattern) ---
// Birinchi navbatda ballar bo'yicha kamayish tartibida (desc), 
// agar ballar teng bo'lsa, ismlar bo'yicha alifbo tartibida (asc) saralash.

const sortedStudents = [...students].sort((a, b) => {
    const scoreA = a?.score ?? 0;
    const scoreB = b?.score ?? 0;
    const nameA = a?.name ?? "";
    const nameB = b?.name ?? "";

    // Tuple pattern orqali solishtirish zanjiri
    return (scoreB - scoreA) || nameA.localeCompare(nameB);
});


// --- NATIJALARNI TEKSHIRISH (DEMO) ---
console.log("--- 1. Map + Filter Zanjirlari ---");
console.log("80+ ballargan ismlar:", highScorerNames);
console.log("Backend yangilangan:", updatedBackendStudents);
console.log("Reactistlar ballari:", reactStudentScores);

console.log("\n--- 2. Reduce Statistikasi ---");
console.log("Kasblar bo'yicha guruh:", studentsByProfession);
console.log("Jami ball va O'rtacha:", scoreSummary, "O'rtacha:", averageScore);

console.log("\n--- 3. Find, Some, Every ---");
console.log("Top talaba (98 ball):", topStudent);
console.log("25 yoshdan kattalar mavjudmi?:", hasOlderStudents);
console.log("Hamma voyaga yetganmi?:", isEveryoneAdult);

console.log("\n--- 4. Ko'p mezonli Sort (Score desc, Name asc) ---");
console.log(sortedStudents.map(s => `${s.name} (${s.score} ball)`));
