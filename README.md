1. Konfiguratsiya (.env o‘rniga config.js)
Maxfiylik uchun tokenni alohida faylda saqlash tavsiya etiladi.

JavaScript
// config.js
export const CONFIG = {
  GITHUB_TOKEN: 'your_github_token_here', // Yoki process.env.GITHUB_TOKEN
  BASE_URL: 'https://api.github.com'
};
2. Universal API Wrapper
AbortController va retry mexanizmi bilan birlashtirilgan fetch funksiyasi.

JavaScript
// api.js
import { CONFIG } from './config.js';

export async function api(url, options = {}, retries = 3, timeout = 5000) {
  const controller = new AbortController();
  const id = setTimeout(() => controller.abort(), timeout);

  try {
    const response = await fetch(`${CONFIG.BASE_URL}${url}`, {
      ...options,
      headers: {
        'Authorization': `token ${CONFIG.GITHUB_TOKEN}`,
        'Content-Type': 'application/json',
        ...options.headers
      },
      signal: controller.signal
    });

    clearTimeout(id);
    
    // Rate limit tahlili
    const remaining = response.headers.get('x-ratelimit-remaining');
    console.log(`[Rate Limit] Qolgan so'rovlar: ${remaining}`);

    if (!response.ok) throw new Error(`HTTP Error: ${response.status}`);
    return await response.json();
  } catch (error) {
    if (retries > 0) return api(url, options, retries - 1, timeout);
    throw error;
  }
}
3. Async Generator (Pagination uchun)
Barcha repolarni sahifalar bo‘yicha avtomatik yuklab oluvchi generator.

JavaScript
// githubService.js
import { api } from './api.js';

export async function* getUserRepos(username) {
  let page = 1;
  while (true) {
    const repos = await api(`/users/${username}/repos?per_page=100&page=${page}`);
    if (repos.length === 0) break;
    yield repos;
    page++;
  }
}
4. Asosiy mantiq va Jadval ko'rinishi
Foydalanuvchi ma'lumotlarini olish va natijani chiroyli formatda chiqarish.

JavaScript
// main.js
import { api } from './api.js';
import { getUserRepos } from './githubService.js';

async function fetchGitHubData(username) {
  try {
    // 1. User profile
    const user = await api(`/users/${username}`);
    console.log(`User: ${user.login} (${user.name})`);

    // 2. Repositories
    const tableData = [];
    for await (const page of getUserRepos(username)) {
      page.forEach(repo => {
        tableData.push({
          Name: repo.name,
          Stars: repo.stargazers_count,
          Forks: repo.forks_count,
          Language: repo.language || 'N/A'
        });
      });
    }

    // 3. Natijani jadval ko'rinishida chiqarish
    console.table(tableData);
    
  } catch (err) {
    console.error("Xatolik yuz berdi:", err.message);
  }
}

fetchGitHubData('username');
Asosiy funksionalliklar tushuntirishi:
AbortController: timeout tugashi bilan so‘rovni to‘xtatadi, bu resurslarni tejashga yordam beradi.

Retry: api funksiyasi xatolik yuz berganda rekursiv tarzda yana 3 marta urinib ko‘radi.

Async Generator: yield orqali xotirani to‘ldirmasdan, sahifama-sahifa katta hajmdagi repolarni yuklash imkonini beradi.

Rate Limit: GitHub API javob sarlavhalaridagi x-ratelimit-remaining qiymatini konsolga chiqarib boradi.

console.table: JavaScript-ning o‘rnatilgan funksiyasi yordamida ma’lumotlarni toza va tushunarli jadval shaklida vizuallashtiradi.
