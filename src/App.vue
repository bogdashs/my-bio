<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'

const GITHUB_USERNAME = 'bogdashs'

const currentSection = ref('projects')
const selectedArticle = ref(null)
const isCommandOpen = ref(false)
const commandSearch = ref('')
const copyNotice = ref(null)
const year = new Date().getFullYear()

// РАЗДЕЛЫ
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
  isLoadingRepos.value = true
  repoError.value = null
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
  fetchGithubRepos()
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeyDown)
})
</script>

<template>
  <div class="app-root">
    <transition name="toast">
      <div v-if="copyNotice" class="toast-notice">
        {{ copyNotice }}
      </div>
    </transition>

    <!-- Командное меню (Ctrl+K) -->
    <div v-if="isCommandOpen" class="cmd-overlay" @click.self="isCommandOpen = false">
      <div class="cmd-palette">
        <div class="cmd-input-wrap">
          <span class="cmd-icon">&gt;</span>
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
        <button class="brand" @click="switchSection('whoami')">
          <span class="brand-avatar">A</span>
          <span class="brand-name">axven</span>
        </button>

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
        <p class="eyebrow">whoami</p>
        <h1 class="title-primary">Фронтенд <span class="slash">/</span> КиберБез</h1>
        <p class="text-secondary lead">
          Фронтенд различных сайтов (vue/css/html/javascript), так-же кибер безопасность (c/c++) ai (python)
        </p>

        <div class="spec-list">
          <div class="spec-row" @click="copyText('Artix Linux · runit · Hyprland', 'Дистрибутив')">
            <span class="spec-label">Окружение</span>
            <span class="spec-value">
              Artix Linux · runit · Hyprland
              <span class="copy-hint">copy</span>
            </span>
          </div>
          <div class="spec-row" @click="copyText('C23, C++20, x86 ASM, Vue 3', 'Стек')">
            <span class="spec-label">Стек</span>
            <span class="spec-value">
              C, C++, x86 ASM, Vue 3, CSS, HTML, Python3, JavaScript, Java
              <span class="copy-hint">copy</span>
            </span>
          </div>
          <div class="spec-row" @click="copyText('Ghostty, Kitty, CLion, Neovim', 'Инструменты')">
            <span class="spec-label">Инструменты</span>
            <span class="spec-value">
              WebStorm, Kitty, CLion, Micro
              <span class="copy-hint">copy</span>
            </span>
          </div>
        </div>

        <div class="links-cluster">
          <a :href="'https://github.com/' + GITHUB_USERNAME" target="_blank" class="glow-link">
            GitHub <span class="arrow">↗</span>
          </a>
          <a href="https://steamcommunity.com/id/bogdashss/" target="_blank" class="glow-link">
            Steam <span class="arrow">↗</span>
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

        <div v-else-if="!repos.length" class="empty-state">
          <p class="empty-title">Пока пусто</p>
          <p class="empty-text">Публичных репозиториев не найдено.</p>
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

          <div v-if="!articles.length" class="empty-state">
            <p class="empty-title">Пока пусто</p>
            <p class="empty-text">Расследования скоро появятся.</p>
          </div>

          <div v-else class="card-stack">
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

    <footer class="footer">
      <span>© {{ year }} axven</span>
      <span class="footer-meta">Vue · Vite</span>
    </footer>
  </div>
</template>

<style scoped>
.app-root {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  background: #fafaf8;
  color: #141414;
  font-family: 'Geist', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  position: relative;
}

/* Toast */
.toast-notice {
  position: fixed;
  bottom: 24px;
  right: 24px;
  z-index: 300;
  background: #141414;
  color: #fafaf8;
  padding: 9px 15px;
  border-radius: 8px;
  font-size: 0.78rem;
  font-family: 'Geist Mono', monospace;
  box-shadow: 0 12px 32px rgba(20, 20, 20, 0.22);
}
.toast-enter-active, .toast-leave-active { transition: all 0.2s ease; }
.toast-enter-from, .toast-leave-to { opacity: 0; transform: translateY(6px); }

/* Command Palette (Ctrl+K) */
.cmd-overlay {
  position: fixed;
  inset: 0;
  background: rgba(20, 20, 20, 0.24);
  backdrop-filter: blur(6px);
  -webkit-backdrop-filter: blur(6px);
  z-index: 200;
  display: flex;
  align-items: flex-start;
  justify-content: center;
  padding: 14vh 16px 16px;
}

.cmd-palette {
  width: 100%;
  max-width: 540px;
  background: #ffffff;
  border: 1px solid #e4e1db;
  border-radius: 14px;
  box-shadow: 0 24px 64px -16px rgba(20, 20, 20, 0.28);
  overflow: hidden;
  animation: cmdIn 0.16s ease-out;
}

@keyframes cmdIn {
  from { opacity: 0; transform: translateY(-6px) scale(0.99); }
  to { opacity: 1; transform: none; }
}

.cmd-input-wrap {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 16px 18px;
  border-bottom: 1px solid #eeebe5;
}

.cmd-icon {
  color: #a3a19c;
  font-family: 'Geist Mono', monospace;
  font-size: 0.9rem;
}

.cmd-input-wrap input {
  background: transparent;
  border: none;
  outline: none;
  color: #141414;
  font-family: inherit;
  font-size: 0.92rem;
  width: 100%;
}

.cmd-input-wrap input::placeholder { color: #a3a19c; }

.cmd-list {
  max-height: 300px;
  overflow-y: auto;
  padding: 8px;
}

.cmd-hint {
  padding: 20px 16px;
  text-align: center;
  color: #a3a19c;
  font-size: 0.82rem;
}

.cmd-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
  padding: 11px 12px;
  border-radius: 8px;
  cursor: pointer;
  transition: background 0.12s;
}

.cmd-item:hover { background: #f5f3ef; }

.cmd-title {
  font-size: 0.86rem;
  color: #141414;
}

.cmd-type {
  font-size: 0.66rem;
  font-family: 'Geist Mono', monospace;
  letter-spacing: 0.05em;
  text-transform: uppercase;
  color: #8a8a8a;
  background: #f4f2ee;
  border: 1px solid #eae7e1;
  padding: 2px 7px;
  border-radius: 4px;
  flex-shrink: 0;
}

/* Header */
.header {
  position: sticky;
  top: 0;
  z-index: 50;
  background: rgba(250, 250, 248, 0.82);
  backdrop-filter: blur(14px);
  -webkit-backdrop-filter: blur(14px);
  border-bottom: 1px solid #e8e5df;
}

.header-inner {
  width: 100%;
  max-width: 760px;
  margin: 0 auto;
  padding: 0 24px;
  height: 64px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
}

.brand {
  display: flex;
  align-items: center;
  gap: 10px;
  background: none;
  border: 0;
  padding: 0;
  color: inherit;
  cursor: pointer;
}

.brand-avatar {
  width: 28px;
  height: 28px;
  border-radius: 8px;
  background: #141414;
  color: #fafaf8;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.78rem;
  font-weight: 600;
}

.brand-name {
  font-size: 0.92rem;
  font-weight: 600;
  letter-spacing: -0.02em;
}

.nav {
  display: flex;
  align-items: center;
  gap: 2px;
}

.nav-link {
  background: transparent;
  border: none;
  color: #737373;
  padding: 7px 11px;
  font-size: 0.84rem;
  border-radius: 7px;
  cursor: pointer;
  transition: color 0.15s ease, background 0.15s ease;
  font-family: inherit;
}

.nav-link:hover { color: #141414; background: rgba(20, 20, 20, 0.045); }
.nav-link.active {
  color: #141414;
  background: rgba(20, 20, 20, 0.06);
  font-weight: 500;
}

.cmd-trigger {
  background: #ffffff;
  border: 1px solid #e2dfd8;
  border-radius: 7px;
  padding: 5px 8px;
  cursor: pointer;
  display: flex;
  gap: 3px;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
}

.cmd-trigger:hover {
  border-color: #c9c5bd;
  box-shadow: 0 1px 2px rgba(20, 20, 20, 0.05);
}

.kbd {
  font-family: 'Geist Mono', monospace;
  font-size: 0.68rem;
  color: #8a8a8a;
  line-height: 1.2;
}

/* Layout */
.main-content {
  position: relative;
  z-index: 1;
  flex: 1;
  width: 100%;
  max-width: 760px;
  margin: 0 auto;
  padding: 64px 24px 72px;
}

.section-view {
  animation: sectionIn 0.28s ease both;
}

@keyframes sectionIn {
  from { opacity: 0; transform: translateY(6px); }
  to { opacity: 1; transform: none; }
}

.eyebrow {
  font-family: 'Geist Mono', monospace;
  font-size: 0.72rem;
  letter-spacing: 0.12em;
  text-transform: uppercase;
  color: #8a8a8a;
  margin: 0 0 14px;
}

.title-primary {
  font-size: clamp(2.1rem, 6vw, 3rem);
  font-weight: 600;
  letter-spacing: -0.045em;
  line-height: 1.08;
  margin: 0 0 18px 0;
  color: #141414;
}

.slash {
  color: #c6c3bc;
  font-weight: 400;
}

.title-secondary {
  font-size: 1.15rem;
  font-weight: 600;
  letter-spacing: -0.02em;
  margin: 0;
  color: #141414;
}

.lead {
  font-size: 1.02rem;
  line-height: 1.65;
  margin-bottom: 36px;
  color: #5f5f5f;
  max-width: 60ch;
}

.heading-row {
  display: flex;
  align-items: baseline;
  gap: 10px;
  margin-bottom: 20px;
  padding-bottom: 14px;
  border-bottom: 1px solid #e8e5df;
  text-align: left;
}

.count-badge {
  margin-left: auto;
  font-size: 0.7rem;
  font-family: 'Geist Mono', monospace;
  color: #8a8a8a;
  padding: 2px 8px;
  background: #ffffff;
  border: 1px solid #e8e5df;
  border-radius: 99px;
}

/* WHOAMI Specs */
.spec-list {
  border-top: 1px solid #e8e5df;
  margin-bottom: 32px;
}

.spec-row {
  display: grid;
  grid-template-columns: 150px 1fr;
  gap: 16px;
  align-items: center;
  padding: 13px 0;
  border-bottom: 1px solid #e8e5df;
  cursor: copy;
  transition: background 0.15s ease;
}

.spec-row:hover { background: rgba(20, 20, 20, 0.025); }

.spec-label {
  font-family: 'Geist Mono', monospace;
  font-size: 0.72rem;
  letter-spacing: 0.06em;
  text-transform: uppercase;
  color: #9a9893;
}

.spec-value {
  font-family: 'Geist Mono', monospace;
  font-size: 0.83rem;
  color: #3d3d3d;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
}

.copy-hint {
  font-size: 0.66rem;
  letter-spacing: 0.1em;
  text-transform: uppercase;
  color: #9a9893;
  opacity: 0;
  transition: opacity 0.15s ease;
}

.spec-row:hover .copy-hint { opacity: 1; }

.links-cluster {
  display: flex;
  gap: 28px;
}

.glow-link {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  font-size: 0.9rem;
  font-weight: 500;
  color: #141414;
  text-decoration: none;
  padding-bottom: 2px;
  border-bottom: 1px solid #cfccc5;
  transition: border-color 0.15s ease, color 0.15s ease;
}

.glow-link:hover { border-color: #141414; }

.arrow {
  transition: transform 0.15s ease;
  color: #8a8a8a;
}

.glow-link:hover .arrow { transform: translate(2px, -2px); }
.interactive-card:hover .arrow { transform: translateX(3px); }

/* PROJECTS */
.repo-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 14px;
}

.repo-card {
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  text-decoration: none;
  background: #ffffff;
  border: 1px solid #e8e5df;
  border-radius: 12px;
  padding: 18px;
  min-height: 132px;
  transition: transform 0.2s cubic-bezier(0.16, 1, 0.3, 1), box-shadow 0.2s ease, border-color 0.2s ease;
}

.repo-card:hover {
  border-color: #d5d1c9;
  transform: translateY(-2px);
  box-shadow: 0 12px 24px -14px rgba(20, 20, 20, 0.22);
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
  min-width: 0;
}

.repo-icon {
  color: #a3a19c;
  flex-shrink: 0;
}

.repo-name {
  font-size: 0.86rem;
  font-weight: 500;
  color: #141414;
  font-family: 'Geist Mono', monospace;
  letter-spacing: -0.01em;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.repo-external {
  color: #b5b2ac;
  font-size: 0.85rem;
  transition: all 0.15s ease;
  flex-shrink: 0;
}

.repo-card:hover .repo-external {
  color: #141414;
  transform: translate(2px, -2px);
}

.repo-desc {
  font-size: 0.84rem;
  color: #6b6b6b;
  line-height: 1.5;
  margin: 0 0 16px 0;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.repo-footer {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  font-size: 0.7rem;
  font-family: 'Geist Mono', monospace;
  color: #9a9893;
  border-top: 1px solid #f0eee9;
  padding-top: 12px;
}

.repo-lang {
  display: flex;
  align-items: center;
  gap: 6px;
  color: #5f5f5f;
  min-width: 0;
}

.lang-dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  display: inline-block;
  flex-shrink: 0;
}

.repo-stats {
  display: flex;
  align-items: center;
}

.stat-item.date {
  color: #a3a19c;
  font-variant-numeric: tabular-nums;
}

/* Статус загрузки и ошибок */
.status-box {
  padding: 22px 20px;
  background: #ffffff;
  border: 1px dashed #ddd9d1;
  border-radius: 12px;
  color: #737373;
  font-size: 0.88rem;
  display: flex;
  align-items: center;
  gap: 12px;
}

.status-box.error {
  color: #b42318;
  border-color: #f0cbc7;
  border-style: solid;
  background: #fffafa;
}

.retry-btn {
  background: #141414;
  border: 1px solid #141414;
  color: #fafaf8;
  border-radius: 7px;
  padding: 6px 12px;
  font-size: 0.78rem;
  cursor: pointer;
  margin-left: auto;
  transition: opacity 0.15s ease;
}

.retry-btn:hover { opacity: 0.85; }

.spinner {
  width: 14px;
  height: 14px;
  border: 2px solid #e8e5df;
  border-top-color: #141414;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.empty-state {
  padding: 48px 20px;
  text-align: center;
  border: 1px dashed #ddd9d1;
  border-radius: 12px;
  background: rgba(255, 255, 255, 0.55);
}

.empty-title {
  margin: 0 0 6px;
  font-size: 0.95rem;
  font-weight: 600;
  color: #141414;
}

.empty-text {
  margin: 0;
  font-size: 0.85rem;
  color: #8a8a8a;
}

/* Articles */
.card-stack {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.interactive-card {
  background: #ffffff;
  border: 1px solid #e8e5df;
  border-radius: 12px;
  padding: 22px;
  text-align: left;
  cursor: pointer;
  transition: transform 0.2s cubic-bezier(0.16, 1, 0.3, 1), box-shadow 0.2s ease, border-color 0.2s ease;
}

.interactive-card:hover {
  border-color: #d5d1c9;
  transform: translateY(-2px);
  box-shadow: 0 12px 24px -16px rgba(20, 20, 20, 0.2);
}

.card-header-meta {
  display: flex;
  align-items: center;
  gap: 9px;
  margin-bottom: 12px;
  flex-wrap: wrap;
}

.tag-pill {
  font-family: 'Geist Mono', monospace;
  font-size: 0.68rem;
  letter-spacing: 0.03em;
  padding: 3px 8px;
  border-radius: 5px;
  background: #f4f2ee;
  border: 1px solid #eae7e1;
  color: #5f5f5f;
}

.meta-dot {
  width: 3px;
  height: 3px;
  background: #c9c5bd;
  border-radius: 50%;
}

.meta-text {
  font-size: 0.76rem;
  color: #9a9893;
  font-family: 'Geist Mono', monospace;
}

.card-main-title {
  font-size: 1.05rem;
  font-weight: 600;
  letter-spacing: -0.02em;
  line-height: 1.35;
  margin: 0 0 8px 0;
  color: #141414;
}

.card-excerpt {
  font-size: 0.88rem;
  color: #6b6b6b;
  line-height: 1.6;
  margin: 0 0 16px 0;
}

.action-text {
  font-size: 0.82rem;
  color: #141414;
  font-weight: 500;
  display: inline-flex;
  align-items: center;
  gap: 6px;
}

/* Reader */
.action-back {
  background: transparent;
  border: none;
  color: #737373;
  cursor: pointer;
  font-size: 0.82rem;
  margin-bottom: 28px;
  padding: 0;
  font-family: inherit;
  display: inline-flex;
  align-items: center;
  gap: 6px;
  transition: color 0.15s ease;
}

.action-back:hover { color: #141414; }

.reader-header {
  margin-bottom: 32px;
  padding-bottom: 24px;
  border-bottom: 1px solid #e8e5df;
}

.badge-row {
  display: flex;
  align-items: center;
  gap: 9px;
  margin-bottom: 14px;
  flex-wrap: wrap;
}

.reader-title {
  font-size: clamp(1.6rem, 4vw, 2rem);
  font-weight: 600;
  letter-spacing: -0.035em;
  line-height: 1.25;
  margin: 0;
  color: #141414;
}

.reader-body {
  color: #3f3f3f;
  line-height: 1.8;
  font-size: 0.98rem;
}

.reader-body :deep(p) { margin: 0 0 18px 0; }
.reader-body :deep(.lead) {
  font-size: 1.05rem;
  color: #141414;
  margin-bottom: 24px;
}

.reader-body :deep(h3) {
  color: #141414;
  font-size: 1.15rem;
  font-weight: 600;
  margin: 34px 0 12px 0;
  letter-spacing: -0.015em;
}

.reader-body :deep(code) {
  font-family: 'Geist Mono', monospace;
  font-size: 0.82rem;
  background: #f4f2ee;
  border: 1px solid #eae7e1;
  padding: 2px 6px;
  border-radius: 5px;
  color: #141414;
}

.reader-body :deep(.table-wrap) {
  overflow-x: auto;
  margin: 24px 0;
  border: 1px solid #e8e5df;
  border-radius: 10px;
}

.reader-body :deep(table) {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.85rem;
  text-align: left;
}

.reader-body :deep(th) {
  padding: 10px 14px;
  background: #faf9f6;
  color: #737373;
  font-weight: 500;
  border-bottom: 1px solid #e8e5df;
}

.reader-body :deep(td) {
  padding: 10px 14px;
  border-bottom: 1px solid #f0eee9;
}

.reader-body :deep(.text-danger) { color: #b42318; font-family: 'Geist Mono', monospace; }
.reader-body :deep(.text-success) { color: #067647; font-family: 'Geist Mono', monospace; }

/* Notes */
.timeline-container {
  border-top: 1px solid #e8e5df;
}

.timeline-card {
  display: grid;
  grid-template-columns: 130px 1fr;
  gap: 20px;
  align-items: baseline;
  padding: 16px 0;
  border-bottom: 1px solid #e8e5df;
}

.timeline-date {
  font-family: 'Geist Mono', monospace;
  font-size: 0.73rem;
  color: #9a9893;
  letter-spacing: 0.02em;
}

.timeline-text {
  font-size: 0.92rem;
  color: #3f3f3f;
  line-height: 1.6;
}

/* Footer */
.footer {
  width: 100%;
  max-width: 760px;
  margin: 0 auto;
  padding: 22px 24px 32px;
  border-top: 1px solid #e8e5df;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  font-family: 'Geist Mono', monospace;
  font-size: 0.7rem;
  color: #9a9893;
}

/* Mobile */
@media (max-width: 640px) {
  .header-inner {
    height: 58px;
    padding: 0 16px;
    gap: 8px;
  }

  .brand-name { display: none; }
  .nav { gap: 0; }
  .nav-link { padding: 6px 7px; font-size: 0.78rem; }
  .cmd-trigger { padding: 5px 6px; }

  .main-content { padding: 40px 16px 56px; }

  .spec-row {
    grid-template-columns: 1fr;
    gap: 5px;
    padding: 12px 0;
  }

  .repo-grid { grid-template-columns: 1fr; }

  .timeline-card {
    grid-template-columns: 1fr;
    gap: 4px;
  }

  .footer { padding: 20px 16px 28px; }

  .toast-notice {
    right: 16px;
    bottom: 16px;
    left: 16px;
    text-align: center;
  }
}

@media (max-width: 380px) {
  .nav-link { padding: 6px 5px; font-size: 0.72rem; }
  .cmd-trigger { padding: 4px 5px; }
}

@media (prefers-reduced-motion: reduce) {
  .section-view { animation: none; }
  .repo-card, .interactive-card, .arrow, .repo-external { transition: none; }
}
</style>
