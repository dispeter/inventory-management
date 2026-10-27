<template>
  <div class="app" :class="{ 'is-collapsed': collapsed, 'drawer-open': drawerOpen }">
    <aside id="app-sidebar" class="sidebar">
      <div class="sidebar-header">
        <div class="brand">
          <svg class="brand-mark" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
            <path d="M21 8l-9-5-9 5v8l9 5 9-5V8z" />
            <path d="M3 8l9 5 9-5M12 13v8" />
          </svg>
          <div class="brand-text">
            <span class="brand-name">{{ t('nav.companyName') }}</span>
            <span class="brand-subtitle">{{ t('nav.subtitle') }}</span>
          </div>
        </div>
      </div>

      <!-- Desktop-only: sits on the sidebar's right edge so it works in both expanded and rail states -->
      <button
        type="button"
        class="sidebar-collapse-btn"
        :aria-label="collapsed ? 'Expand sidebar' : 'Collapse sidebar'"
        :title="collapsed ? 'Expand sidebar' : 'Collapse sidebar'"
        :aria-expanded="!collapsed"
        aria-controls="app-sidebar"
        @click="toggleCollapsed"
      >
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
          <path d="M15 6l-6 6 6 6" />
        </svg>
      </button>

      <nav class="sidebar-nav" aria-label="Main">
        <!-- Active state comes from the router (exact match, so "/" is not active everywhere);
             router-link also sets aria-current="page" on the exact-active link. -->
        <router-link
          v-for="item in navItems"
          :key="item.to"
          :to="item.to"
          class="nav-link"
          exact-active-class="is-active"
          :title="item.label"
          :aria-label="item.label"
        >
          <svg class="nav-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
            <path :d="item.icon" />
          </svg>
          <span class="nav-label">{{ item.label }}</span>
        </router-link>
      </nav>

      <div class="sidebar-footer">
        <LanguageSwitcher />
        <ProfileMenu
          @show-profile-details="showProfileDetails = true"
          @show-tasks="showTasks = true"
        />
      </div>
    </aside>

    <div class="backdrop" aria-hidden="true" @click="closeDrawer"></div>

    <div class="content-column">
      <header class="top-bar">
        <button
          type="button"
          class="drawer-toggle"
          aria-label="Open navigation menu"
          aria-controls="app-sidebar"
          :aria-expanded="drawerOpen"
          @click="toggleDrawer"
        >
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" aria-hidden="true">
            <path d="M4 6h16M4 12h16M4 18h16" />
          </svg>
        </button>
        <FilterBar />
      </header>
      <main class="main-content">
        <router-view />
      </main>
    </div>

    <ProfileDetailsModal
      :is-open="showProfileDetails"
      @close="showProfileDetails = false"
    />

    <TasksModal
      :is-open="showTasks"
      :tasks="tasks"
      @close="showTasks = false"
      @add-task="addTask"
      @delete-task="deleteTask"
      @toggle-task="toggleTask"
    />
  </div>
</template>

<script>
import { ref, onMounted, onBeforeUnmount, computed, watch } from 'vue'
import { useRoute } from 'vue-router'
import { api } from './api'
import { useAuth } from './composables/useAuth'
import { useI18n } from './composables/useI18n'
import FilterBar from './components/FilterBar.vue'
import ProfileMenu from './components/ProfileMenu.vue'
import ProfileDetailsModal from './components/ProfileDetailsModal.vue'
import TasksModal from './components/TasksModal.vue'
import LanguageSwitcher from './components/LanguageSwitcher.vue'

export default {
  name: 'App',
  components: {
    FilterBar,
    ProfileMenu,
    ProfileDetailsModal,
    TasksModal,
    LanguageSwitcher
  },
  setup() {
    const { currentUser } = useAuth()
    const { t } = useI18n()
    const showProfileDetails = ref(false)
    const showTasks = ref(false)
    const apiTasks = ref([])

    // Sidebar navigation
    const navItems = computed(() => [
      { to: '/', label: t('nav.overview'), icon: 'M3 12l9-8 9 8M5 10v10h5v-6h4v6h5V10' },
      { to: '/inventory', label: t('nav.inventory'), icon: 'M21 8l-9-5-9 5v8l9 5 9-5V8zM3 8l9 5 9-5M12 13v8' },
      { to: '/orders', label: t('nav.orders'), icon: 'M9 5H7a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2V7a2 2 0 00-2-2h-2M9 5a2 2 0 012-2h2a2 2 0 012 2v0a2 2 0 01-2 2h-2a2 2 0 01-2-2zM9 12h6M9 16h4' },
      { to: '/spending', label: t('nav.finance'), icon: 'M12 2v20M17 6H9.5a3.5 3.5 0 000 7h5a3.5 3.5 0 010 7H6' },
      { to: '/demand', label: t('nav.demandForecast'), icon: 'M3 17l6-6 4 4 8-8M15 7h6v6' },
      { to: '/reports', label: t('nav.reports'), icon: 'M4 20V10M10 20V4M16 20v-7M22 20H2' }
    ])

    // ---- Sidebar state ----
    const STORAGE_KEY = 'sidebar-collapsed'
    const route = useRoute()
    const drawerOpen = ref(false)
    const isMobile = ref(false) // < 1024px: off-canvas drawer
    const isNarrowDesktop = ref(false) // 1024-1279px: icon rail by default

    // localStorage can throw (blocked storage, some private modes), so every access is guarded.
    const readPref = () => {
      try {
        const v = localStorage.getItem(STORAGE_KEY)
        return v === null ? null : v === 'true'
      } catch (e) {
        return null
      }
    }
    const writePref = (value) => {
      try {
        localStorage.setItem(STORAGE_KEY, String(value))
      } catch (e) {
        // Ignore: the choice simply won't persist across reloads.
      }
    }

    // null means the user never chose, so the viewport width decides.
    const collapsePref = ref(readPref())
    // The drawer is always fully expanded; the rail only exists at desktop widths.
    const collapsed = computed(() => {
      if (isMobile.value) return false
      return collapsePref.value === null ? isNarrowDesktop.value : collapsePref.value
    })
    const toggleCollapsed = () => {
      collapsePref.value = !collapsed.value
      writePref(collapsePref.value)
    }

    const closeDrawer = () => { drawerOpen.value = false }
    const toggleDrawer = () => { drawerOpen.value = !drawerOpen.value }

    // Picking a destination from the drawer should reveal the page, so close it on every route change.
    watch(() => route.fullPath, closeDrawer)

    const onKeydown = (e) => {
      if (e.key === 'Escape' && drawerOpen.value) closeDrawer()
    }

    const mobileMq = window.matchMedia('(max-width: 1023.98px)')
    const narrowMq = window.matchMedia('(min-width: 1024px) and (max-width: 1279.98px)')
    const syncMedia = () => {
      isMobile.value = mobileMq.matches
      isNarrowDesktop.value = narrowMq.matches
      // Don't leave the drawer flagged open after resizing up to desktop (it would reappear on resize down).
      if (!isMobile.value) drawerOpen.value = false
    }
    syncMedia()
    mobileMq.addEventListener('change', syncMedia)
    narrowMq.addEventListener('change', syncMedia)
    window.addEventListener('keydown', onKeydown)
    onBeforeUnmount(() => {
      mobileMq.removeEventListener('change', syncMedia)
      narrowMq.removeEventListener('change', syncMedia)
      window.removeEventListener('keydown', onKeydown)
    })

    // Merge mock tasks from currentUser with API tasks
    const tasks = computed(() => {
      return [...currentUser.value.tasks, ...apiTasks.value]
    })

    const loadTasks = async () => {
      try {
        apiTasks.value = await api.getTasks()
      } catch (err) {
        console.error('Failed to load tasks:', err)
      }
    }

    const addTask = async (taskData) => {
      try {
        const newTask = await api.createTask(taskData)
        // Add new task to the beginning of the array
        apiTasks.value.unshift(newTask)
      } catch (err) {
        console.error('Failed to add task:', err)
      }
    }

    const deleteTask = async (taskId) => {
      try {
        // Check if it's a mock task (from currentUser)
        const isMockTask = currentUser.value.tasks.some(t => t.id === taskId)

        if (isMockTask) {
          // Remove from mock tasks
          const index = currentUser.value.tasks.findIndex(t => t.id === taskId)
          if (index !== -1) {
            currentUser.value.tasks.splice(index, 1)
          }
        } else {
          // Remove from API tasks
          await api.deleteTask(taskId)
          apiTasks.value = apiTasks.value.filter(t => t.id !== taskId)
        }
      } catch (err) {
        console.error('Failed to delete task:', err)
      }
    }

    const toggleTask = async (taskId) => {
      try {
        // Check if it's a mock task (from currentUser)
        const mockTask = currentUser.value.tasks.find(t => t.id === taskId)

        if (mockTask) {
          // Toggle mock task status
          mockTask.status = mockTask.status === 'pending' ? 'completed' : 'pending'
        } else {
          // Toggle API task
          const updatedTask = await api.toggleTask(taskId)
          const index = apiTasks.value.findIndex(t => t.id === taskId)
          if (index !== -1) {
            apiTasks.value[index] = updatedTask
          }
        }
      } catch (err) {
        console.error('Failed to toggle task:', err)
      }
    }

    onMounted(loadTasks)

    return {
      t,
      navItems,
      collapsed,
      drawerOpen,
      toggleCollapsed,
      toggleDrawer,
      closeDrawer,
      showProfileDetails,
      showTasks,
      tasks,
      addTask,
      deleteTask,
      toggleTask
    }
  }
}
</script>

<style>
/* ===== Design tokens ===== */
:root {
  /* 4px spacing scale */
  --space-1: 4px;
  --space-2: 8px;
  --space-3: 12px;
  --space-4: 16px;
  --space-5: 20px;
  --space-6: 24px;
  --space-7: 28px;
  --space-8: 32px;

  --radius-md: 6px;  /* controls */
  --radius-lg: 10px; /* cards */

  --shadow-sm: 0 1px 2px rgba(15, 23, 42, 0.06);
  --shadow-md: 0 4px 12px rgba(15, 23, 42, 0.08);

  --color-sidebar: #0f172a;
  --color-sidebar-text: #94a3b8;
  --color-sidebar-text-hover: #e2e8f0;
  --color-sidebar-border: rgba(148, 163, 184, 0.16);
  --color-accent: #2563eb;
  --color-accent-tint: rgba(37, 99, 235, 0.16);
  --color-canvas: #f1f5f9;
  --color-surface: #ffffff;
  --color-border: #e2e8f0;
  --color-border-strong: #cbd5e1;
  --color-text: #0f172a;
  --color-text-muted: #64748b;
  --color-success: #059669;
  --color-warning: #ea580c;
  --color-danger: #dc2626;

  --sidebar-width: 240px;
  --topbar-height: 60px;

  --z-topbar: 30;
  --z-sidebar: 40;
  --z-backdrop: 50;
  --z-drawer: 60;
  /* Modals/dropdowns in components use 1000-2000 and stay above all of these. */
}

/* ===== Base ===== */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: system-ui, -apple-system, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, 'Noto Sans', 'Hiragino Sans', 'Yu Gothic UI', Meiryo, sans-serif;
  background: var(--color-canvas);
  color: var(--color-text);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

:focus-visible {
  outline: 2px solid var(--color-accent);
  outline-offset: 2px;
}

/* ===== Shell layout ===== */
.app {
  display: grid;
  grid-template-columns: var(--sidebar-width) minmax(0, 1fr);
  min-height: 100vh;
  transition: grid-template-columns 0.2s ease;
}

.app.is-collapsed {
  --sidebar-width: 64px;
}

.sidebar {
  position: sticky;
  top: 0;
  align-self: start;
  height: 100vh;
  z-index: var(--z-sidebar);
  display: flex;
  flex-direction: column;
  background: var(--color-sidebar);
  color: var(--color-sidebar-text);
  /* Not overflow:auto here: footer dropdowns must not be clipped. Only the nav scrolls. */
}

.sidebar-header {
  display: flex;
  align-items: center;
  height: var(--topbar-height);
  padding: 0 var(--space-5);
  border-bottom: 1px solid var(--color-sidebar-border);
  flex-shrink: 0;
}

.brand {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  min-width: 0;
}

.brand-mark {
  width: 24px;
  height: 24px;
  color: #60a5fa;
  flex-shrink: 0;
}

.brand-text {
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.brand-name,
.brand-subtitle {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.brand-name {
  color: #ffffff;
  font-size: 0.9375rem;
  font-weight: 700;
  letter-spacing: -0.01em;
  line-height: 1.3;
}

.brand-subtitle {
  font-size: 0.75rem;
  line-height: 1.3;
}

.sidebar-collapse-btn {
  position: absolute;
  top: calc(var(--topbar-height) / 2 - 12px);
  right: -12px;
  width: 24px;
  height: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--color-surface);
  border: 1px solid var(--color-border);
  border-radius: 50%;
  color: var(--color-text-muted);
  cursor: pointer;
  box-shadow: var(--shadow-sm);
  z-index: 2;
}

.sidebar-collapse-btn:hover {
  color: var(--color-text);
  border-color: var(--color-border-strong);
}

.sidebar-collapse-btn svg {
  width: 14px;
  height: 14px;
  transition: transform 0.2s ease;
}

.is-collapsed .sidebar-collapse-btn svg {
  transform: rotate(180deg);
}

.sidebar-nav {
  flex: 1;
  min-height: 0;
  overflow-y: auto;
  overflow-x: hidden;
  display: flex;
  flex-direction: column;
  gap: var(--space-1);
  padding: var(--space-4) 0;
}

.nav-link {
  position: relative;
  display: flex;
  align-items: center;
  gap: var(--space-3);
  height: 40px;
  padding: 0 var(--space-5);
  color: var(--color-sidebar-text);
  text-decoration: none;
  font-size: 0.875rem;
  font-weight: 500;
  transition: background-color 0.15s ease, color 0.15s ease;
}

.nav-link:hover {
  color: var(--color-sidebar-text-hover);
  background: rgba(148, 163, 184, 0.08);
}

.nav-link.is-active {
  color: #ffffff;
  background: var(--color-accent-tint);
}

/* 3px accent rail on the sidebar's left edge */
.nav-link.is-active::before {
  content: '';
  position: absolute;
  left: 0;
  top: 0;
  bottom: 0;
  width: 3px;
  background: var(--color-accent);
}

.nav-icon {
  width: 18px;
  height: 18px;
  flex-shrink: 0;
}

.nav-label {
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.sidebar :focus-visible {
  outline-color: #60a5fa;
  outline-offset: -2px;
}

.sidebar-footer {
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
  padding: var(--space-3);
  border-top: 1px solid var(--color-sidebar-border);
  flex-shrink: 0;
}

/* Restyle the footer widgets for the dark sidebar (specificity beats their scoped styles) */
.app .sidebar-footer .language-switcher,
.app .sidebar-footer .profile-menu {
  width: 100%;
}

.app .sidebar-footer .language-button,
.app .sidebar-footer .profile-button {
  width: 100%;
  height: 40px;
  padding: 0 var(--space-2);
  background: rgba(148, 163, 184, 0.08);
  border: 1px solid var(--color-sidebar-border);
  border-radius: var(--radius-md);
  color: var(--color-sidebar-text-hover);
  overflow: hidden;
}

.app .sidebar-footer .language-button:hover,
.app .sidebar-footer .profile-button:hover {
  background: rgba(148, 163, 184, 0.16);
  border-color: rgba(148, 163, 184, 0.3);
}

.app .sidebar-footer .globe-icon,
.app .sidebar-footer .chevron {
  color: var(--color-sidebar-text);
}

.app .sidebar-footer .language-label,
.app .sidebar-footer .profile-name {
  flex: 1;
  min-width: 0;
  text-align: left;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  color: var(--color-sidebar-text-hover);
}

.app .sidebar-footer .avatar {
  width: 28px;
  height: 28px;
  flex-shrink: 0;
}

/* Dropdowns open upward from the bottom of the sidebar so they stay on screen */
.app .sidebar-footer .dropdown-menu {
  top: auto;
  bottom: calc(100% + var(--space-2));
  left: 0;
  right: auto;
  min-width: 240px;
}

/* ===== Collapsed icon rail ===== */
.is-collapsed .sidebar-header {
  padding: 0;
  justify-content: center;
}

.is-collapsed .sidebar-footer .chevron {
  display: none;
}

/* Visually hidden (not display:none) so the language/profile buttons keep their accessible names in the rail */
.is-collapsed .brand-text,
.is-collapsed .nav-label,
.app.is-collapsed .sidebar-footer .language-label,
.app.is-collapsed .sidebar-footer .profile-name {
  position: absolute;
  width: 1px;
  height: 1px;
  overflow: hidden;
  clip-path: inset(50%);
  white-space: nowrap;
}

.is-collapsed .nav-link {
  justify-content: center;
  padding: 0;
}

.is-collapsed .sidebar-footer {
  padding: var(--space-3) var(--space-2);
  align-items: center;
}

.app.is-collapsed .sidebar-footer .language-button,
.app.is-collapsed .sidebar-footer .profile-button {
  justify-content: center;
  padding: 0;
}

/* ===== Backdrop / drawer toggle (hidden on desktop) ===== */
.backdrop {
  display: none;
}

.drawer-toggle {
  display: none;
  align-items: center;
  justify-content: center;
  width: 36px;
  height: 36px;
  flex-shrink: 0;
  background: var(--color-surface);
  border: 1px solid var(--color-border-strong);
  border-radius: var(--radius-md);
  color: var(--color-text);
  cursor: pointer;
}

.drawer-toggle:hover {
  background: var(--color-canvas);
}

.drawer-toggle svg {
  width: 18px;
  height: 18px;
}

/* ===== Content column ===== */
.content-column {
  min-width: 0;
  display: flex;
  flex-direction: column;
}

.top-bar {
  position: sticky;
  top: 0;
  z-index: var(--z-topbar);
  display: flex;
  align-items: center;
  gap: var(--space-3);
  min-height: var(--topbar-height);
  padding: var(--space-2) var(--space-8);
  background: var(--color-surface);
  border-bottom: 1px solid var(--color-border);
}

.main-content {
  flex: 1;
  width: 100%;
  max-width: 1600px;
  margin: 0 auto;
  padding: var(--space-6) var(--space-8);
}

/* ===== Responsive: off-canvas drawer below 1024px ===== */
@media (max-width: 1023.98px) {
  .app {
    grid-template-columns: minmax(0, 1fr);
  }

  .sidebar {
    position: fixed;
    top: 0;
    left: 0;
    bottom: 0;
    width: 240px;
    z-index: var(--z-drawer);
    transform: translateX(-100%);
    /* visibility keeps the closed drawer out of the tab order once the slide finishes */
    visibility: hidden;
    transition: transform 0.2s ease, visibility 0s linear 0.2s;
  }

  .drawer-open .sidebar {
    transform: translateX(0);
    visibility: visible;
    transition: transform 0.2s ease, visibility 0s;
  }

  .sidebar-collapse-btn {
    display: none;
  }

  .drawer-open .backdrop {
    display: block;
    position: fixed;
    inset: 0;
    z-index: var(--z-backdrop);
    background: rgba(15, 23, 42, 0.5);
  }

  .drawer-toggle {
    display: flex;
  }

  .top-bar {
    padding: var(--space-2) var(--space-4);
  }

  .main-content {
    padding: var(--space-4);
  }
}

@media (prefers-reduced-motion: reduce) {
  .app,
  .sidebar,
  .drawer-open .sidebar,
  .sidebar-collapse-btn svg,
  .nav-link {
    transition: none;
  }
}

/* ===== Shared surfaces ===== */
.page-header {
  margin-bottom: var(--space-6);
}

.page-header h2 {
  font-size: 1.75rem;
  font-weight: 700;
  color: var(--color-text);
  margin-bottom: var(--space-1);
  letter-spacing: -0.025em;
}

.page-header p {
  color: var(--color-text-muted);
  font-size: 0.875rem;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(240px, 1fr));
  gap: var(--space-5);
  margin-bottom: var(--space-6);
}

.stat-card {
  background: var(--color-surface);
  padding: var(--space-5);
  border-radius: var(--radius-lg);
  border: 1px solid var(--color-border);
  box-shadow: var(--shadow-sm);
  transition: border-color 0.2s ease, box-shadow 0.2s ease;
}

.stat-card:hover {
  border-color: var(--color-border-strong);
  box-shadow: var(--shadow-md);
}

.stat-label {
  color: var(--color-text-muted);
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: var(--space-2);
}

.stat-value {
  font-size: 2rem;
  font-weight: 700;
  color: var(--color-text);
  letter-spacing: -0.025em;
  font-variant-numeric: tabular-nums;
}

.stat-card.warning .stat-value {
  color: var(--color-warning);
}

.stat-card.success .stat-value {
  color: var(--color-success);
}

.stat-card.danger .stat-value {
  color: var(--color-danger);
}

.stat-card.info .stat-value {
  color: var(--color-accent);
}

.card {
  background: var(--color-surface);
  border-radius: var(--radius-lg);
  padding: var(--space-5);
  border: 1px solid var(--color-border);
  box-shadow: var(--shadow-sm);
  margin-bottom: var(--space-5);
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: var(--space-4);
  margin-bottom: var(--space-4);
  padding-bottom: var(--space-3);
  border-bottom: 1px solid var(--color-border);
}

.card-title {
  font-size: 1.0625rem;
  font-weight: 700;
  color: var(--color-text);
  letter-spacing: -0.02em;
}

.table-container {
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
}

thead {
  background: #f8fafc;
  border-top: 1px solid var(--color-border);
  border-bottom: 1px solid var(--color-border);
}

th {
  text-align: left;
  padding: var(--space-3);
  font-weight: 600;
  color: #475569;
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

td {
  padding: var(--space-3);
  border-top: 1px solid #f1f5f9;
  color: #334155;
  font-size: 0.875rem;
  font-variant-numeric: tabular-nums;
}

tbody tr {
  transition: background-color 0.15s ease;
}

tbody tr:hover {
  background: #f8fafc;
}

.badge {
  display: inline-block;
  padding: var(--space-1) var(--space-3);
  border-radius: var(--radius-md);
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

.badge.success {
  background: #d1fae5;
  color: #065f46;
}

.badge.warning {
  background: #fed7aa;
  color: #92400e;
}

.badge.danger {
  background: #fecaca;
  color: #991b1b;
}

.badge.info {
  background: #dbeafe;
  color: #1e40af;
}

.badge.increasing {
  background: #d1fae5;
  color: #065f46;
}

.badge.decreasing {
  background: #fecaca;
  color: #991b1b;
}

.badge.stable {
  background: #e0e7ff;
  color: #3730a3;
}

.badge.high {
  background: #fecaca;
  color: #991b1b;
}

.badge.medium {
  background: #fed7aa;
  color: #92400e;
}

.badge.low {
  background: #dbeafe;
  color: #1e40af;
}

.loading {
  text-align: center;
  padding: var(--space-8);
  color: var(--color-text-muted);
  font-size: 0.875rem;
}

.error {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
  padding: var(--space-4);
  border-radius: var(--radius-md);
  margin: var(--space-4) 0;
  font-size: 0.875rem;
}
</style>
