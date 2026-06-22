<template>
  <div class="app-shell">
    <header class="hero-panel">
      <div>
        <p class="eyebrow">TaskBoard</p>
        <h1>ระบบจัดการงานด้วย CRUD</h1>
        <p class="hero-copy">สร้าง แก้ไข ลบ และติดตามสถานะงานได้ในหน้าเดียว พร้อม UI สดใสใช้งานง่าย</p>
      </div>
      <div class="hero-meta">
        <div>
          <span>งานทั้งหมด</span>
          <strong>{{ tasks.length }}</strong>
        </div>
        <div>
          <span>งานค้าง</span>
          <strong>{{ pendingCount }}</strong>
        </div>
      </div>
    </header>

    <section class="panel form-panel">
      <div class="panel-title">
        <div>
          <h2>{{ isEditing ? 'แก้ไขงาน' : 'เพิ่มงานใหม่' }}</h2>
          <p>กรอกข้อมูลงานเพื่อบันทึกหรือแก้ไขรายการงาน</p>
        </div>
        <span class="badge">{{ isEditing ? 'แก้ไข' : 'ใหม่' }}</span>
      </div>

      <form @submit.prevent="saveTask" class="task-form">
        <div class="form-grid">
          <label>
            ชื่องาน
            <input v-model="title" placeholder="เช่น อัปเดตหน้า dashboard" />
          </label>

          <label>
            ความสำคัญ
            <select v-model="priority">
              <option value="high">High</option>
              <option value="medium">Medium</option>
              <option value="low">Low</option>
            </select>
          </label>
        </div>

        <label>
          รายละเอียดงาน
          <textarea v-model="description" rows="3" placeholder="อธิบายเนื้องานเพิ่มเติม"></textarea>
        </label>

        <div class="form-grid">
          <label>
            สถานะงาน
            <select v-model="status" :disabled="!isEditing">
              <option value="todo">Todo</option>
              <option value="inprogress">In Progress</option>
              <option value="done">Done</option>
            </select>
          </label>

          <div class="form-actions">
            <button type="submit" class="btn-primary">{{ isEditing ? 'บันทึกงาน' : 'เพิ่มงาน' }}</button>
            <button type="button" class="btn-secondary" v-if="isEditing" @click="resetForm">ยกเลิก</button>
          </div>
        </div>

        <p v-if="errorMessage" class="error-message">{{ errorMessage }}</p>
      </form>
    </section>

    <section class="panel tasks-panel">
      <div class="panel-title small">
        <div>
          <h2>รายการงาน</h2>
          <p>กดแก้ไขเพื่อปรับข้อมูลงานหรือกดลบเพื่อลบรายการ</p>
        </div>
        <div class="filter-group">
          <select v-model="filterStatus" @change="loadTasks">
            <option value="">ทุกสถานะ</option>
            <option value="todo">Todo</option>
            <option value="inprogress">In Progress</option>
            <option value="done">Done</option>
          </select>
          <select v-model="filterPriority" @change="loadTasks">
            <option value="">ทุกความสำคัญ</option>
            <option value="high">High</option>
            <option value="medium">Medium</option>
            <option value="low">Low</option>
          </select>
        </div>
      </div>

      <div class="loading-state" v-if="loading">กำลังโหลดข้อมูล...</div>

      <div v-else>
        <div class="task-grid">
          <article v-for="task in tasks" :key="task.id" class="task-card" :class="task.priority">
            <div class="task-card__tag">
              <span class="status-pill" :data-status="task.status">{{ task.status }}</span>
              <span class="priority-pill">{{ task.priority }}</span>
            </div>
            <h3>{{ task.title }}</h3>
            <p v-if="task.description">{{ task.description }}</p>
            <div class="task-card__footer">
              <small>สร้าง: {{ formatDate(task.created_at) }}</small>
              <small v-if="task.updated_at">แก้ไข: {{ formatDate(task.updated_at) }}</small>
            </div>
            <div class="task-card__actions">
              <button class="btn-ghost" @click="editTask(task)">แก้ไข</button>
              <button class="btn-danger" @click="deleteTask(task.id)">ลบ</button>
            </div>
          </article>
        </div>

        <p v-if="tasks.length === 0" class="empty-state">ไม่มีงานในระบบตามเงื่อนไขนี้</p>
      </div>
    </section>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'

const API = import.meta.env.VITE_API_URL || '/api'
const tasks = ref([])
const loading = ref(false)
const title = ref('')
const description = ref('')
const priority = ref('medium')
const status = ref('todo')
const filterStatus = ref('')
const filterPriority = ref('')
const editTaskId = ref(null)
const errorMessage = ref('')

const isEditing = computed(() => !!editTaskId.value)
const pendingCount = computed(() => tasks.value.filter((task) => task.status !== 'done').length)

function buildQuery() {
  const params = new URLSearchParams()
  if (filterStatus.value) params.append('status', filterStatus.value)
  if (filterPriority.value) params.append('priority', filterPriority.value)
  return params.toString() ? `?${params.toString()}` : ''
}

async function requestJSON(url, options = {}) {
  const res = await fetch(url, options)
  const body = await res.text()
  if (!res.ok) {
    let message = body
    try {
      message = JSON.parse(body).error || body
    } catch {
      message = body || `HTTP ${res.status}`
    }
    throw new Error(message)
  }
  return body ? JSON.parse(body) : null
}

async function loadTasks() {
  loading.value = true
  errorMessage.value = ''
  try {
    tasks.value = await requestJSON(`${API}/tasks${buildQuery()}`)
  } catch (err) {
    errorMessage.value = err.message
  } finally {
    loading.value = false
  }
}

function resetForm() {
  title.value = ''
  description.value = ''
  priority.value = 'medium'
  status.value = 'todo'
  editTaskId.value = null
  errorMessage.value = ''
}

async function saveTask() {
  if (!title.value.trim()) {
    errorMessage.value = 'กรุณากรอกชื่องาน'
    return
  }

  const payload = {
    title: title.value.trim(),
    description: description.value.trim(),
    priority: priority.value,
    status: status.value,
  }

  try {
    errorMessage.value = ''
    if (isEditing.value) {
      await requestJSON(`${API}/tasks/${editTaskId.value}`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload),
      })
    } else {
      await requestJSON(`${API}/tasks`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload),
      })
    }
    await loadTasks()
    resetForm()
  } catch (err) {
    errorMessage.value = err.message
  }
}

function editTask(task) {
  editTaskId.value = task.id
  title.value = task.title
  description.value = task.description || ''
  priority.value = task.priority
  status.value = task.status
  window.scrollTo({ top: 0, behavior: 'smooth' })
}

async function deleteTask(id) {
  if (!confirm('ยืนยันลบงานนี้?')) return
  try {
    await requestJSON(`${API}/tasks/${id}`, { method: 'DELETE' })
    await loadTasks()
  } catch (err) {
    errorMessage.value = err.message
  }
}

function formatDate(value) {
  if (!value) return '-'
  return new Intl.DateTimeFormat('th-TH', {
    day: '2-digit',
    month: 'short',
    year: 'numeric',
    hour: '2-digit',
    minute: '2-digit',
  }).format(new Date(value))
}

onMounted(loadTasks)
</script>

<style scoped>
:global(body) {
  background: #f7faff;
}

.app-shell {
  max-width: 1080px;
  margin: 0 auto;
  padding: 28px 24px 40px;
  font-family: Inter, system-ui, sans-serif;
  color: #1f2937;
}

.hero-panel {
  display: flex;
  justify-content: space-between;
  gap: 1.5rem;
  padding: 2rem;
  border-radius: 28px;
  background: linear-gradient(135deg, #4f46e5, #2563eb);
  color: #fff;
  box-shadow: 0 28px 80px rgba(37, 99, 235, 0.18);
  margin-bottom: 1.75rem;
}

.eyebrow {
  margin: 0 0 0.75rem;
  letter-spacing: 0.24em;
  font-size: 0.85rem;
  opacity: 0.9;
  text-transform: uppercase;
}

.hero-panel h1 {
  margin: 0;
  font-size: clamp(2rem, 2.5vw, 2.5rem);
  line-height: 1.05;
}

.hero-copy {
  margin-top: 0.85rem;
  max-width: 42rem;
  line-height: 1.7;
  opacity: 0.95;
}

.hero-meta {
  display: grid;
  gap: 1rem;
  align-content: center;
  text-align: right;
}

.hero-meta div {
  background: rgba(255,255,255,0.16);
  border-radius: 18px;
  padding: 1rem 1.15rem;
}

.hero-meta span {
  display: block;
  opacity: 0.9;
  font-size: 0.9rem;
}

.hero-meta strong {
  display: block;
  margin-top: 0.3rem;
  font-size: 1.65rem;
}

.panel {
  background: #fff;
  border: 1px solid #e5e7f0;
  border-radius: 26px;
  padding: 1.75rem;
  box-shadow: 0 22px 48px rgba(15, 23, 42, 0.06);
  margin-bottom: 1.5rem;
}

.panel-title {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 1rem;
  margin-bottom: 1.5rem;
}

.panel-title.small {
  flex-wrap: wrap;
}

.panel-title h2 {
  margin: 0;
  font-size: 1.15rem;
}

.panel-title p {
  margin: 0.45rem 0 0;
  color: #4b5563;
}

.badge {
  background: #eef2ff;
  color: #3730a3;
  padding: 0.65rem 1rem;
  border-radius: 999px;
  font-weight: 700;
}

.task-form {
  display: grid;
  gap: 1.15rem;
}

.form-grid {
  display: grid;
  gap: 1rem;
  grid-template-columns: repeat(2, minmax(0, 1fr));
}

label {
  display: grid;
  gap: 0.5rem;
  font-weight: 600;
  color: #334155;
}

input,
textarea,
select {
  width: 100%;
  border: 1px solid #d1d5db;
  border-radius: 18px;
  background: #f8fafc;
  padding: 0.95rem 1rem;
  font-size: 1rem;
  color: #111827;
  outline: none;
}

textarea {
  min-height: 90px;
  resize: vertical;
}

.form-actions {
  display: flex;
  align-items: center;
  gap: 0.85rem;
  flex-wrap: wrap;
}

button {
  border: none;
  border-radius: 16px;
  padding: 0.95rem 1.4rem;
  cursor: pointer;
  font-weight: 700;
}

.btn-primary {
  background: #4338ca;
  color: #fff;
}

.btn-secondary {
  background: #eef2ff;
  color: #3730a3;
}

.error-message {
  color: #b91c1c;
  padding: 0.5rem 0.75rem;
  background: #fef2f2;
  border-radius: 14px;
  border: 1px solid #fecaca;
}

.filter-group {
  display: flex;
  gap: 0.75rem;
  flex-wrap: wrap;
  min-width: 250px;
}

.filter-group select {
  min-width: 150px;
}

.task-grid {
  display: grid;
  gap: 1rem;
}

.task-card {
  display: grid;
  gap: 1rem;
  padding: 1.35rem;
  border-radius: 24px;
  border: 1px solid #e5e7eb;
  background: #ffffff;
}

.task-card__tag {
  display: flex;
  justify-content: space-between;
  gap: 0.75rem;
  align-items: center;
}

.status-pill,
.priority-pill {
  display: inline-flex;
  align-items: center;
  padding: 0.55rem 0.9rem;
  border-radius: 999px;
  font-size: 0.85rem;
  font-weight: 700;
}

.status-pill[data-status='todo'] {
  background: #e0f2fe;
  color: #0c4a6e;
}

.status-pill[data-status='inprogress'] {
  background: #fef9c3;
  color: #713f12;
}

.status-pill[data-status='done'] {
  background: #d1fae5;
  color: #166534;
}

.priority-pill {
  background: #eff6ff;
  color: #1d4ed8;
}

.task-card h3 {
  margin: 0;
  font-size: 1.05rem;
}

.task-card p {
  margin: 0;
  color: #475569;
  line-height: 1.75;
}

.task-card__footer {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;
  font-size: 0.9rem;
  color: #64748b;
}

.task-card__actions {
  display: flex;
  gap: 0.75rem;
  flex-wrap: wrap;
}

.btn-ghost {
  background: #eef2ff;
  color: #3730a3;
}

.btn-danger {
  background: #fee2e2;
  color: #b91c1c;
}

.loading-state,
.empty-state {
  text-align: center;
  padding: 1.5rem 1rem;
  color: #475569;
}

@media (max-width: 780px) {
  .hero-panel,
  .panel {
    padding: 1.4rem;
  }

  .hero-panel {
    flex-direction: column;
    text-align: left;
  }

  .hero-meta {
    text-align: left;
  }

  .form-grid,
  .filter-group {
    grid-template-columns: 1fr;
  }
}
</style>
