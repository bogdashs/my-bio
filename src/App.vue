<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'

const GITHUB_USERNAME = 'bogdashs'

const currentSection = ref('projects')
const selectedArticle = ref(null)
const isCommandOpen = ref(false)
const commandSearch = ref('')
const copyNotice = ref(null)

// 1. СВЕЧЕНИЕ КУРСОРА
const mouseX = ref(-1000)
const mouseY = ref(-1000)

function handleMouseMove(e) {
  mouseX.value = e.clientX
  mouseY.value = e.clientY
}

// 2. РАЗДЕЛЫ
const sections = [
  { id: 'whoami', label: 'whoami' },
  { id: 'projects', label: 'projects' },
  { id: 'researches', label: 'researches' },
  { id: 'notes', label: 'notes' }
]

function switchSection(id) {
  currentSection.value = id
  selectedArticle.value = null
  isCommandOpen.value = false
}

// Цвета языков GitHub
const LANG_COLORS = {
  'Python': '#3572A5',
  'C++': '#f34b7d',
  'C': '#555555',
  'Shell': '#89e051',
  'CSS': '#563d7c',
  'JavaScript': '#f1e05a',
  'TypeScript': '#3178c6',
  'Vue': '#41b883',
  'HTML': '#e34c26',
  'Java': '#b07219'
}

function getLangColor(lang) {
  return LANG_COLORS[lang] || '#8b949e'
}

const repos = ref([])
const isLoadingRepos = ref(true)
const repoError = ref(null)

async function fetchGithubRepos() {
  try {
    const res = await fetch(`https://api.github.com/users/${GITHUB_USERNAME}/repos?sort=updated&per_page=14`)
    if (!res.ok) throw new Error(`HTTP ${res.status}`)
    const data = await res.json()

    repos.value = data
        .filter(repo => !repo.fork)
        .map(repo => ({
          id: repo.id,
          name: repo.name,
          desc: repo.description || 'Описание проекта пока отсутствует',
          lang: repo.language || 'Code',
          url: repo.html_url,
          updated: new Date(repo.updated_at).toLocaleDateString('ru-RU', {
            day: 'numeric',
            month: 'short'
          })
        }))
  } catch (err) {
    console.error(err)
    repoError.value = 'Не удалось загрузить репозитории с GitHub'
  } finally {
    isLoadingRepos.value = false
  }
}

// 4. СТАТЬИ И РАССЛЕДОВАНИЯ
const articles = [
  // {
  //   id: 'anti-ddos',
  //   title: '«Anti-DDoS 2 Тбит/с»: почему L7-атаки все равно кладут игровой процесс',
  //   date: '4 Авг 2026',
  //   readTime: '5 мин',
  //   tag: 'Network',
  //   excerpt: 'Разбор специфики игрового трафика: разница между защитой инфраструктуры на уровнях L3/L4 и специфическими пакетами прикладного уровня.',
  //   content: `
  //     <p>Красивые баннеры про емкость фильтрации в 2 Тбит/с говорят лишь об устойчивости сетевого оборудования провайдера к примитивному объемному флуду (L3/L4).</p>
  //     <p>Игровой протокол Minecraft работает на уровне L7. Чтобы перегрузить сетевой поток Netty, не требуется гигабитный поток данных — достаточно узконаправленной генерации некорректных handshake-запросов, которые стандартные магистральные экраны пропускают как легитимный трафик.</p>
  //   `
  // }
]

// 5. ЗАМЕТКИ
const notes = [
  {
    date: '10/04/2026',
    text: 'Выложил biov2 сайт! (bio.axven.ru)'
  }
]

// Копирование
function copyText(val, label) {
  navigator.clipboard.writeText(val)
  copyNotice.value = `${label} скопирован`
  setTimeout(() => { copyNotice.value = null }, 2000)
}

// Поиск (Ctrl+K)
const searchResults = computed(() => {
  const q = commandSearch.value.toLowerCase().trim()
  if (!q) return []
  const list = []
  repos.value.forEach(r => {
    if (r.name.toLowerCase().includes(q) || r.desc.toLowerCase().includes(q)) {
      list.push({ title: r.name, type: 'GitHub', action: () => window.open(r.url, '_blank') })
    }
  })
  articles.forEach(a => {
    if (a.title.toLowerCase().includes(q)) {
      list.push({ title: a.title, type: 'Research', action: () => { selectedArticle.value = a; currentSection.value = 'researches' } })
    }
  })
  sections.forEach(s => {
    if (s.label.includes(q)) {
      list.push({ title: `Раздел ${s.label}`, type: 'Nav', action: () => switchSection(s.id) })
    }
  })
  return list
})

function handleKeyDown(e) {
  if ((e.ctrlKey || e.metaKey) && e.key.toLowerCase() === 'k') {
    e.preventDefault()
    isCommandOpen.value = !isCommandOpen.value
  } else if (e.key === 'Escape') {
    isCommandOpen.value = false
  }
}

onMounted(() => {
  window.addEventListener('keydown', handleKeyDown)
  window.addEventListener('mousemove', handleMouseMove)
  fetchGithubRepos()
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeyDown)
  window.removeEventListener('mousemove', handleMouseMove)
})
</script>

<template>
  <div class="app-root">
    <div
        class="cursor-glow"
        :style="{ left: mouseX + 'px', top: mouseY + 'px' }"
    ></div>

    <transition name="toast">
      <div v-if="copyNotice" class="toast-notice">
        {{ copyNotice }}
      </div>
    </transition>

    <!-- Командное меню (Ctrl+K) -->
    <div v-if="isCommandOpen" class="cmd-overlay" @click.self="isCommandOpen = false">
      <div class="cmd-palette">
        <div class="cmd-input-wrap">
          <span class="cmd-icon">⌘</span>
          <input
              v-model="commandSearch"
              placeholder="Поиск по проектам, статьям... (Esc для выхода)"
              autofocus
          />
        </div>
        <div class="cmd-list">
          <div v-if="!commandSearch" class="cmd-hint">
            Начните вводить название репозитория, статьи или раздела...
          </div>
          <div
              v-for="(res, idx) in searchResults"
              :key="idx"
              class="cmd-item"
              @click="res.action(); isCommandOpen = false"
          >
            <span class="cmd-title">{{ res.title }}</span>
            <span class="cmd-type">{{ res.type }}</span>
          </div>
        </div>
      </div>
    </div>

    <!-- ВЕРХНИЙ БАР (TOPBAR) -->
    <header class="header">
      <div class="header-inner">
        <div class="brand" @click="switchSection('whoami')">
          <div class="brand-avatar">A</div>
          <span class="brand-name">axven</span>
        </div>

        <nav class="nav">
          <button
              v-for="s in sections"
              :key="s.id"
              :class="['nav-link', { active: currentSection === s.id && !selectedArticle }]"
              @click="switchSection(s.id)"
          >
            {{ s.label }}
          </button>
        </nav>

        <button class="cmd-trigger" @click="isCommandOpen = true" title="Поиск (Ctrl+K)">
          <span class="kbd">Ctrl</span><span class="kbd">K</span>
        </button>
      </div>
    </header>

    <!-- ОСНОВНОЙ КОНТЕНТ -->
    <main class="main-content">

      <!-- 1. WHOAMI -->
      <section v-if="currentSection === 'whoami'" class="section-view">
        <h1 class="title-primary">Фронтенд / КиберБез</h1>
        <p class="text-secondary lead">
          Фронтенд различных сайтов (vue/css/html/javascript), так-же кибер безопасность (c/c++) ai (python)
        </p>

        <div class="surface-box specs-wrap">
          <div class="spec-entry" @click="copyText('Artix Linux · runit · Hyprland', 'Дистрибутив')">
            <span class="spec-name">Окружение</span>
            <span class="spec-value hover-copy">Artix Linux · runit · Hyprland 📋</span>
          </div>
          <div class="spec-entry" @click="copyText('C23, C++20, x86 ASM, Vue 3', 'Стек')">
            <span class="spec-name">Стек</span>
            <span class="spec-value hover-copy">C, C++, x86 ASM, Vue 3, CSS, HTML, Python3, JavaScript, Java 📋</span>
          </div>
          <div class="spec-entry" @click="copyText('Ghostty, Kitty, CLion, Neovim', 'Инструменты')">
            <span class="spec-name">Инструменты</span>
            <span class="spec-value hover-copy">WebStorm, Kitty, CLion, Micro 📋</span>
          </div>
        </div>

        <div class="links-cluster">
          <a :href="'https://github.com/' + GITHUB_USERNAME" target="_blank" class="glow-link">
            GitHub <span class="arrow">→</span>
          </a>
          <a href="https://steamcommunity.com/id/bogdashss/" target="_blank" class="glow-link">
            Steam <span class="arrow">→</span>
          </a>
        </div>
      </section>

      <!-- 2. PROJECTS -->
      <section v-if="currentSection === 'projects'" class="section-view">
        <div class="heading-row">
          <h2 class="title-secondary">Репозитории</h2>
          <span v-if="!isLoadingRepos && !repoError" class="count-badge">{{ repos.length }}</span>
        </div>

        <div v-if="isLoadingRepos" class="status-box">
          <span class="spinner"></span> Загрузка проектов с GitHub API...
        </div>

        <div v-else-if="repoError" class="status-box error">
          <span>{{ repoError }}</span>
          <button class="retry-btn" @click="fetchGithubRepos">Повторить</button>
        </div>

        <div v-else class="repo-grid">
          <a
              v-for="p in repos"
              :key="p.id"
              :href="p.url"
              target="_blank"
              class="repo-card"
          >
            <div class="repo-head">
              <div class="repo-title-wrap">
                <svg class="repo-icon" viewBox="0 0 16 16" width="14" height="14" fill="currentColor">
                  <path d="M2 2.5A2.5 2.5 0 0 1 4.5 0h8.75a.75.75 0 0 1 .75.75v12.5a.75.75 0 0 1-.75.75h-2.5a.75.75 0 0 1 0-1.5h1.75v-2h-8a1 1 0 0 0-.714 1.7.75.75 0 1 1-1.072 1.05A2.495 2.495 0 0 1 2 11.5Zm10.5-1h-8a1 1 0 0 0-1 1v6.708A2.486 2.486 0 0 1 4.5 9h8ZM5 12.25a.25.25 0 0 1 .25-.25H12v1.5H5.25a.25.25 0 0 1-.25-.25Z"></path>
                </svg>
                <span class="repo-name">{{ p.name }}</span>
              </div>
              <span class="repo-external">↗</span>
            </div>

            <p class="repo-desc">{{ p.desc }}</p>

            <div class="repo-footer">
              <div class="repo-lang">
                <span class="lang-dot" :style="{ backgroundColor: getLangColor(p.lang) }"></span>
                <span>{{ p.lang }}</span>
              </div>

              <div class="repo-stats">
                <span class="stat-item date">
                  {{ p.updated }}
                </span>
              </div>
            </div>
          </a>
        </div>
      </section>

      <!-- 3. RESEARCHES -->
      <section v-if="currentSection === 'researches'" class="section-view">
        <article v-if="selectedArticle" class="article-reader">
          <button class="action-back" @click="selectedArticle = null">
            <span class="arrow">←</span> назад к списку
          </button>

          <header class="reader-header">
            <div class="badge-row">
              <span class="tag-pill">{{ selectedArticle.tag }}</span>
              <span class="meta-dot"></span>
              <span class="meta-text">{{ selectedArticle.date }}</span>
              <span class="meta-dot"></span>
              <span class="meta-text">{{ selectedArticle.readTime }}</span>
            </div>
            <h1 class="reader-title">{{ selectedArticle.title }}</h1>
          </header>

          <div class="reader-body" v-html="selectedArticle.content"></div>
        </article>

        <div v-else>
          <div class="heading-row">
            <h2 class="title-secondary">Расследования</h2>
            <span class="count-badge">{{ articles.length }}</span>
          </div>

          <div class="card-stack">
            <div
                v-for="art in articles"
                :key="art.id"
                class="interactive-card research-card"
                @click="selectedArticle = art"
            >
              <div class="card-header-meta">
                <span class="tag-pill">{{ art.tag }}</span>
                <span class="meta-text">{{ art.date }}</span>
                <span class="meta-dot"></span>
                <span class="meta-text">{{ art.readTime }}</span>
              </div>
              <h3 class="card-main-title">{{ art.title }}</h3>
              <p class="card-excerpt">{{ art.excerpt }}</p>
              <div class="card-footer-action">
                <span class="action-text">Читать расследование <span class="arrow">→</span></span>
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- 4. NOTES -->
      <section v-if="currentSection === 'notes'" class="section-view">
        <div class="heading-row">
          <h2 class="title-secondary">Заметки</h2>
        </div>

        <div class="timeline-container">
          <div v-for="(n, idx) in notes" :key="idx" class="timeline-card">
            <div class="timeline-date">{{ n.date }}</div>
            <div class="timeline-text">{{ n.text }}</div>
          </div>
        </div>
      </section>

    </main>
  </div>
</template>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Geist:wght@300;400;500;600;700&family=Geist+Mono:wght@400;500&display=swap');

.app-root {
  min-height: 100vh;
  background-color: #08090a;
  color: #ededed;
  font-family: 'Geist', -apple-system, BlinkMacSystemFont, sans-serif;
  position: relative;
  overflow-x: hidden;
  padding-bottom: 96px;
}

/* Свечение от курсора */
.cursor-glow {
  position: fixed;
  transform: translate(-50%, -50%);
  width: 520px;
  height: 520px;
  background: radial-gradient(circle, rgba(255, 255, 255, 0.04) 0%, transparent 65%);
  pointer-events: none;
  z-index: 0;
}

/* Toast */
.toast-notice {
  position: fixed;
  bottom: 24px;
  right: 24px;
  background: #16171b;
  border: 1px solid rgba(255, 255, 255, 0.14);
  color: #ffffff;
  padding: 8px 16px;
  border-radius: 8px;
  font-size: 0.82rem;
  font-family: 'Geist Mono', monospace;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.5);
  z-index: 100;
}
.toast-enter-active, .toast-leave-active { transition: all 0.2s ease; }
.toast-enter-from, .toast-leave-to { opacity: 0; transform: translateY(8px); }

/* Command Palette (Ctrl+K) */
.cmd-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.7);
  backdrop-filter: blur(8px);
  z-index: 200;
  display: flex;
  align-items: flex-start;
  justify-content: center;
  padding-top: 14vh;
}

.cmd-palette {
  width: 100%;
  max-width: 540px;
  background: #111215;
  border: 1px solid rgba(255, 255, 255, 0.12);
  border-radius: 12px;
  box-shadow: 0 24px 60px rgba(0, 0, 0, 0.8);
  overflow: hidden;
  animation: cmdIn 0.15s ease-out;
}

@keyframes cmdIn {
  from { opacity: 0; transform: scale(0.97); }
  to { opacity: 1; transform: scale(1); }
}

.cmd-input-wrap {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 14px 16px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.08);
}

.cmd-icon {
  color: #777;
  font-size: 0.9rem;
}

.cmd-input-wrap input {
  background: transparent;
  border: none;
  outline: none;
  color: #fff;
  font-family: inherit;
  font-size: 0.9rem;
  width: 100%;
}

.cmd-list {
  max-height: 280px;
  overflow-y: auto;
  padding: 8px;
}

.cmd-hint {
  padding: 16px;
  text-align: center;
  color: #666;
  font-size: 0.82rem;
}

.cmd-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px 12px;
  border-radius: 6px;
  cursor: pointer;
  transition: background 0.15s;
}

.cmd-item:hover {
  background: rgba(255, 255, 255, 0.06);
}

.cmd-title {
  font-size: 0.85rem;
  color: #eee;
}

.cmd-type {
  font-size: 0.72rem;
  font-family: 'Geist Mono', monospace;
  color: #777;
  background: rgba(255, 255, 255, 0.04);
  padding: 2px 6px;
  border-radius: 4px;
}

/* Header */
.header {
  position: sticky;
  top: 20px;
  z-index: 50;
  display: flex;
  justify-content: center;
  padding: 0 16px;
}

.header-inner {
  width: 100%;
  max-width: 680px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 6px 12px;
  background: rgba(13, 14, 16, 0.75);
  backdrop-filter: blur(16px);
  -webkit-backdrop-filter: blur(16px);
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 14px;
  box-shadow: 0 12px 32px -8px rgba(0, 0, 0, 0.5);
}

.brand {
  display: flex;
  align-items: center;
  gap: 10px;
  cursor: pointer;
}

.brand-avatar {
  width: 24px;
  height: 24px;
  border-radius: 6px;
  background: #1f2026;
  border: 1px solid rgba(255, 255, 255, 0.12);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.75rem;
  font-weight: 600;
}

.brand-name {
  font-size: 0.88rem;
  font-weight: 500;
}

.nav {
  display: flex;
  gap: 2px;
}

.nav-link {
  background: transparent;
  border: none;
  color: #888;
  padding: 6px 12px;
  font-size: 0.82rem;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.nav-link:hover { color: #fff; }
.nav-link.active {
  color: #fff;
  background: rgba(255, 255, 255, 0.08);
}

.cmd-trigger {
  background: rgba(255, 255, 255, 0.04);
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 6px;
  padding: 3px 6px;
  cursor: pointer;
  display: flex;
  gap: 3px;
}

.kbd {
  font-family: 'Geist Mono', monospace;
  font-size: 0.7rem;
  color: #888;
}

/* Layout */
.main-content {
  position: relative;
  z-index: 1;
  max-width: 660px;
  margin: 56px auto 0;
  padding: 0 20px;
}

.title-primary {
  font-size: 1.85rem;
  font-weight: 600;
  letter-spacing: -0.03em;
  margin: 0 0 12px 0;
}

.title-secondary {
  font-size: 1.25rem;
  font-weight: 600;
  margin: 0;
}

.lead {
  font-size: 0.98rem;
  line-height: 1.6;
  margin-bottom: 28px;
  color: #929292;
}

/* Выравнивание заголовка репозиториев строго по краям сетки */
.heading-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 18px;
  text-align: left;
}

.count-badge {
  font-size: 0.72rem;
  font-family: 'Geist Mono', monospace;
  color: #777;
  padding: 2px 7px;
  background: rgba(255, 255, 255, 0.04);
  border: 1px solid rgba(255, 255, 255, 0.06);
  border-radius: 99px;
}

/* Карточки */
.interactive-card {
  background: linear-gradient(180deg, rgba(255, 255, 255, 0.02) 0%, rgba(255, 255, 255, 0.008) 100%);
  border: 1px solid rgba(255, 255, 255, 0.06);
  border-radius: 12px;
  padding: 20px;
  cursor: pointer;
  transition: all 0.2s ease;
}

.interactive-card:hover {
  background: linear-gradient(180deg, rgba(255, 255, 255, 0.04) 0%, rgba(255, 255, 255, 0.015) 100%);
  border-color: rgba(255, 255, 255, 0.14);
  transform: translateY(-1px);
}

/* WHOAMI Specs */
.specs-wrap {
  display: flex;
  flex-direction: column;
  gap: 12px;
  padding: 16px 20px;
  border-radius: 12px;
  background: rgba(255, 255, 255, 0.015);
  border: 1px solid rgba(255, 255, 255, 0.06);
  margin-bottom: 28px;
}

.spec-entry {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 0.88rem;
  cursor: pointer;
}

.spec-name { color: #6c6c6c; }
.spec-value {
  font-family: 'Geist Mono', monospace;
  font-size: 0.82rem;
  color: #ccc;
  transition: color 0.15s;
}

.spec-entry:hover .hover-copy {
  color: #fff;
  text-decoration: underline;
}

.links-cluster {
  display: flex;
  gap: 12px;
}

.glow-link {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 8px 16px;
  border-radius: 8px;
  background: rgba(255, 255, 255, 0.04);
  border: 1px solid rgba(255, 255, 255, 0.08);
  color: #ededed;
  font-size: 0.85rem;
  text-decoration: none;
  transition: all 0.15s;
}

.glow-link:hover {
  background: rgba(255, 255, 255, 0.08);
}

.arrow { transition: transform 0.15s; }
.glow-link:hover .arrow,
.interactive-card:hover .arrow {
  transform: translateX(2px);
}

/* PROJECTS (Сетка 2 колонки) */
.repo-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 12px;
}

@media (max-width: 620px) {
  .repo-grid {
    grid-template-columns: 1fr;
  }
}

.repo-card {
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  text-decoration: none;
  background: rgba(255, 255, 255, 0.015);
  border: 1px solid rgba(255, 255, 255, 0.06);
  border-radius: 10px;
  padding: 16px;
  min-height: 110px;
  transition: all 0.2s cubic-bezier(0.16, 1, 0.3, 1);
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.2);
}

.repo-card:hover {
  background: rgba(255, 255, 255, 0.035);
  border-color: rgba(255, 255, 255, 0.16);
  transform: translateY(-2px);
  box-shadow: 0 8px 20px -6px rgba(0, 0, 0, 0.5);
}

.repo-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 8px;
}

.repo-title-wrap {
  display: flex;
  align-items: center;
  gap: 8px;
}

.repo-icon {
  color: #666;
  flex-shrink: 0;
}

.repo-name {
  font-size: 0.92rem;
  font-weight: 500;
  color: #ededed;
  font-family: 'Geist Mono', monospace;
  letter-spacing: -0.01em;
}

.repo-external {
  color: #555;
  font-size: 0.85rem;
  transition: all 0.15s ease;
}

.repo-card:hover .repo-external {
  color: #fff;
  transform: translate(2px, -2px);
}

.repo-desc {
  font-size: 0.82rem;
  color: #888;
  line-height: 1.45;
  margin: 0 0 14px 0;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.repo-footer {
  display: flex;
  align-items: center;
  justify-content: space-between;
  font-size: 0.74rem;
  font-family: 'Geist Mono', monospace;
  color: #666;
  border-top: 1px solid rgba(255, 255, 255, 0.04);
  padding-top: 10px;
}

.repo-lang {
  display: flex;
  align-items: center;
  gap: 6px;
  color: #aaa;
}

.lang-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  display: inline-block;
}

.repo-stats {
  display: flex;
  align-items: center;
}

.stat-item.date {
  color: #555;
}

/* Статус загрузки и ошибок */
.status-box {
  padding: 24px;
  background: rgba(255, 255, 255, 0.02);
  border: 1px solid rgba(255, 255, 255, 0.06);
  border-radius: 10px;
  color: #888;
  font-size: 0.9rem;
  display: flex;
  align-items: center;
  gap: 12px;
}

.status-box.error {
  color: #f87171;
  border-color: rgba(248, 113, 113, 0.2);
}

.retry-btn {
  background: transparent;
  border: 1px solid rgba(248, 113, 113, 0.4);
  color: #f87171;
  border-radius: 6px;
  padding: 4px 10px;
  font-size: 0.78rem;
  cursor: pointer;
  margin-left: auto;
}

.spinner {
  width: 14px;
  height: 14px;
  border: 2px solid rgba(255, 255, 255, 0.1);
  border-top-color: #fff;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

/* Articles */
.card-stack {
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.card-header-meta {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 10px;
}

.tag-pill {
  font-family: 'Geist Mono', monospace;
  font-size: 0.7rem;
  padding: 2px 7px;
  border-radius: 5px;
  background: rgba(255, 255, 255, 0.06);
  color: #d1d1d1;
}

.meta-dot {
  width: 3px;
  height: 3px;
  background: #444;
  border-radius: 50%;
}

.meta-text { font-size: 0.78rem; color: #6e6e6e; }

.card-main-title {
  font-size: 1.08rem;
  font-weight: 500;
  letter-spacing: -0.015em;
  margin: 0 0 8px 0;
  color: #fff;
}

.card-excerpt {
  font-size: 0.88rem;
  color: #8a8a8a;
  line-height: 1.55;
  margin: 0 0 14px 0;
}

.action-text {
  font-size: 0.82rem;
  color: #fff;
  font-weight: 500;
}

/* Reader */
.action-back {
  background: transparent;
  border: none;
  color: #888;
  cursor: pointer;
  font-size: 0.82rem;
  margin-bottom: 24px;
  padding: 0;
  font-family: inherit;
  transition: color 0.15s;
}

.action-back:hover { color: #fff; }

.reader-header { margin-bottom: 32px; }

.badge-row {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 12px;
}

.reader-title {
  font-size: 1.85rem;
  font-weight: 600;
  letter-spacing: -0.03em;
  line-height: 1.3;
  margin: 0;
}

.reader-body {
  color: #b5b5b5;
  line-height: 1.75;
  font-size: 0.96rem;
}

:deep(.reader-body .lead) {
  font-size: 1.05rem;
  color: #e5e5e5;
  margin-bottom: 24px;
}

:deep(.reader-body h3) {
  color: #ffffff;
  font-size: 1.18rem;
  font-weight: 600;
  margin: 36px 0 12px 0;
  letter-spacing: -0.015em;
}

:deep(.reader-body code) {
  font-family: 'Geist Mono', monospace;
  font-size: 0.82rem;
  background: rgba(255, 255, 255, 0.05);
  border: 1px solid rgba(255, 255, 255, 0.08);
  padding: 2px 6px;
  border-radius: 5px;
  color: #eeeeee;
}

:deep(.table-wrap) {
  overflow-x: auto;
  margin: 24px 0;
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 10px;
}

:deep(table) {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.85rem;
  text-align: left;
}

:deep(th) {
  padding: 10px 14px;
  background: rgba(255, 255, 255, 0.02);
  color: #777;
  font-weight: 500;
  border-bottom: 1px solid rgba(255, 255, 255, 0.06);
}

:deep(td) {
  padding: 10px 14px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.04);
}

:deep(.text-danger) { color: #f87171; font-family: 'Geist Mono', monospace; }
:deep(.text-success) { color: #4ade80; font-family: 'Geist Mono', monospace; }

/* Notes */
.timeline-container {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.timeline-card {
  padding: 16px;
  border-radius: 10px;
  background: rgba(255, 255, 255, 0.015);
  border: 1px solid rgba(255, 255, 255, 0.05);
}

.timeline-date {
  font-family: 'Geist Mono', monospace;
  font-size: 0.74rem;
  color: #606060;
  margin-bottom: 6px;
}

.timeline-text {
  font-size: 0.9rem;
  color: #b5b5b5;
  line-height: 1.55;
}

@media (max-width: 580px) {
  .header { top: 10px; padding: 0 10px; }
  .nav-link { padding: 5px 8px; font-size: 0.78rem; }
  .title-primary, .reader-title { font-size: 1.5rem; }
}
</style>