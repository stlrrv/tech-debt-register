<template>
  <div style="min-height:100vh;background:#f9fafb;padding:1.5rem 1rem;">
    <div style="max-width:960px;margin:0 auto;">

      <!-- ── Header ── -->
      <div style="background:linear-gradient(135deg,#2563eb 0%,#7c3aed 100%);border-radius:14px;padding:1.5rem 2rem;margin-bottom:1.5rem;color:#fff;display:flex;align-items:center;justify-content:space-between;flex-wrap:wrap;gap:1rem;">
        <div>
          <h1 style="margin:0;font-size:1.5rem;font-weight:700;">Реестр задач</h1>
          <p style="margin:0.25rem 0 0;font-size:0.875rem;opacity:0.8;">Управление техническим долгом и бизнес-задачами</p>
        </div>
        <button v-if="isReadOnly" class="btn btn-light btn-sm" @click="enterEditMode">Войти в режим редактирования</button>
        <button v-else class="btn btn-light btn-sm" @click="exitEditMode">Выйти из режима редактирования</button>
      </div>

      <!-- ── Form ── -->
      <div v-if="!isReadOnly" class="card" style="margin-bottom:1.5rem;">
        <h2 class="section-title">{{ editingTaskId ? 'Редактировать задачу' : 'Добавить задачу' }}</h2>

        <h3 class="group-label">Основная информация</h3>
        <div class="form-grid">
          <div class="field">
            <label class="field-label">Тип задачи</label>
            <select class="field-input" v-model="form.type" @change="updatePreview">
              <option value="business">Бизнес-задача</option>
              <option value="tech">Техническая задача</option>
            </select>
          </div>
          <div class="field">
            <label class="field-label">Название</label>
            <input class="field-input" v-model="form.title" type="text" placeholder="Короткое название" />
          </div>
          <div class="field">
            <label class="field-label">Описание / контекст</label>
            <textarea class="field-input" v-model="form.description" placeholder="Короткое пояснение" rows="3"></textarea>
          </div>
          <div class="field">
            <label class="field-label">Направление</label>
            <select class="field-input" v-model="form.direction">
              <option value="web">Web</option>
              <option value="onec">1С</option>
              <option value="qa">QA</option>
              <option value="common">Общее</option>
            </select>
          </div>
        </div>

        <h3 class="group-label" style="margin-top:1.5rem;">Оценки приоритезации</h3>
        <div class="form-grid">
          <div class="field">
            <label class="field-label">Размер</label>
            <select class="field-input" v-model="form.size" @change="updatePreview">
              <option value="S">S — до 4 ч</option>
              <option value="M">M — 1–2 дня</option>
              <option value="L">L — 3–5 дней</option>
              <option value="XL">XL — неделя+</option>
            </select>
          </div>
          <div class="field">
            <label class="field-label">Процесс <span class="score-weight">×3</span></label>
            <select class="field-input" v-model="form.processScore" @change="updatePreview">
              <option value="1">1 — косметика / вспомогательное</option>
              <option value="2">2 — важный, но не критичный</option>
              <option value="3">3 — критичный для работы</option>
            </select>
          </div>
          <div class="field">
            <label class="field-label">Откладывание <span class="score-weight">×2</span></label>
            <select class="field-input" v-model="form.delayScore" @change="updatePreview">
              <option value="1">1 — ничего не изменится</option>
              <option value="2">2 — накопится долг</option>
              <option value="3">3 — проблема уже есть</option>
            </select>
          </div>
          <div class="field">
            <label class="field-label">Охват <span class="score-weight">×1</span></label>
            <select class="field-input" v-model="form.impactScore" @change="updatePreview">
              <option value="1">1 — 1 человек</option>
              <option value="2">2 — небольшая группа</option>
              <option value="3">3 — вся команда / система</option>
            </select>
          </div>
          <div class="field">
            <label class="field-label">Дедлайн <span class="score-weight">×1</span></label>
            <select class="field-input" v-model="form.deadlineScore" @change="updatePreview">
              <option value="3">Жёсткий — в этом спринте</option>
              <option value="2">Мягкий — позже</option>
              <option value="1">Без срока</option>
            </select>
          </div>
        </div>

        <!-- Preview -->
        <div v-if="preview" style="margin-top:1rem;padding:1rem 1.25rem;background:#eff6ff;border:1.5px solid #bfdbfe;border-radius:10px;display:flex;flex-wrap:wrap;gap:1.5rem;align-items:center;">
          <div>
            <span style="font-size:0.75rem;color:#6b7280;font-weight:600;text-transform:uppercase;letter-spacing:.05em;">Ценность</span>
            <p style="margin:0.125rem 0 0;font-size:1.5rem;font-weight:700;color:#1d4ed8;">{{ preview.value }}<span style="font-size:0.875rem;color:#6b7280;">/21</span></p>
          </div>
          <div>
            <span style="font-size:0.75rem;color:#6b7280;font-weight:600;text-transform:uppercase;letter-spacing:.05em;">Квадрант</span>
            <p style="margin:0.125rem 0 0;font-size:0.9375rem;font-weight:600;color:#1e3a8a;">{{ preview.quadrant }}</p>
          </div>
          <div>
            <span style="font-size:0.75rem;color:#6b7280;font-weight:600;text-transform:uppercase;letter-spacing:.05em;">Рекомендация</span>
            <p style="margin:0.125rem 0 0;font-size:0.875rem;color:#1e3a8a;">{{ preview.recommendation }}</p>
          </div>
        </div>

        <h3 class="group-label" style="margin-top:1.5rem;">Организационная информация</h3>
        <div class="form-grid">
          <div class="field">
            <label class="field-label">Кто добавил</label>
            <input class="field-input" v-model="form.author" type="text" placeholder="Тимлид или техлид" />
          </div>
          <div class="field">
            <label class="field-label">Статус в реестре</label>
            <select class="field-input" v-model="form.registryStatus">
              <option value="new">Новая</option>
              <option value="candidate">Кандидат</option>
              <option value="planned">Запланирована</option>
              <option value="in_progress">В работе</option>
              <option value="done">Закрыта</option>
              <option value="deferred">Отложена</option>
            </select>
          </div>
        </div>

        <!-- Tech fields -->
        <template v-if="form.type === 'tech'">
          <h3 class="group-label" style="margin-top:1.5rem;">Технический контекст</h3>
          <div class="form-grid">
            <div class="field">
              <label class="field-label">Последствие</label>
              <textarea class="field-input" v-model="form.consequence" placeholder="Если не сделаем, то…" rows="3"></textarea>
            </div>
            <div class="field">
              <label class="field-label">Уровень риска</label>
              <select class="field-input" v-model="form.riskLevel">
                <option value="critical">Критичный</option>
                <option value="high">Высокий</option>
                <option value="medium">Средний</option>
                <option value="low">Низкий</option>
              </select>
            </div>
            <div class="field">
              <label class="field-label">Горит через</label>
              <select class="field-input" v-model="form.burnHorizon">
                <option value="already">Уже горит</option>
                <option value="1_3m">1–3 месяца</option>
                <option value="3_6m">3–6 месяцев</option>
                <option value="6_12m">6–12 месяцев</option>
                <option value="someday">Когда-нибудь</option>
              </select>
            </div>
            <div class="field">
              <label class="field-label">Тип проблемы</label>
              <select class="field-input" v-model="form.techCategory">
                <option value="архитектура">Архитектура</option>
                <option value="производительность">Производительность</option>
                <option value="тесты">Тесты</option>
                <option value="зависимости">Зависимости</option>
                <option value="кодовая база">Кодовая база</option>
                <option value="инфраструктура">Инфраструктура</option>
                <option value="безопасность">Безопасность</option>
                <option value="другое">Другое</option>
              </select>
            </div>
            <div class="field">
              <label class="field-label">Нужна декомпозиция</label>
              <select class="field-input" v-model="form.needsDecomposition">
                <option value="false">Нет</option>
                <option value="true">Да</option>
              </select>
            </div>
            <div class="field">
              <label class="field-label">Нужен техлид</label>
              <select class="field-input" v-model="form.needsTechLead">
                <option value="false">Нет</option>
                <option value="true">Да</option>
              </select>
            </div>
          </div>
        </template>

        <!-- Business fields -->
        <template v-if="form.type === 'business'">
          <h3 class="group-label" style="margin-top:1.5rem;">Бизнес-контекст</h3>
          <div class="form-grid">
            <div class="field">
              <label class="field-label">Бизнес-цель</label>
              <textarea class="field-input" v-model="form.businessGoal" placeholder="Что хотим получить для бизнеса" rows="3"></textarea>
            </div>
            <div class="field">
              <label class="field-label">Ожидаемый результат</label>
              <textarea class="field-input" v-model="form.expectedOutcome" placeholder="Что изменится после выполнения" rows="3"></textarea>
            </div>
            <div class="field">
              <label class="field-label">Инициатор</label>
              <select class="field-input" v-model="form.initiator">
                <option value="Руководитель">Руководитель</option>
                <option value="Продажи">Продажи</option>
                <option value="Поддержка">Поддержка</option>
                <option value="Клиент">Клиент</option>
                <option value="Внутренняя команда">Внутренняя команда</option>
              </select>
            </div>
            <div class="field">
              <label class="field-label">Внешняя зависимость</label>
              <textarea class="field-input" v-model="form.externalDependencyNote" placeholder="Короткий текст или комментарий" rows="3"></textarea>
            </div>
            <div class="field">
              <label class="field-label">Критерий готовности (DoD)</label>
              <textarea class="field-input" v-model="form.definitionOfDone" placeholder="Как понять, что задача завершена" rows="3"></textarea>
            </div>
          </div>
        </template>

        <div style="margin-top:1.5rem;display:flex;gap:0.75rem;">
          <button class="btn btn-primary" @click="submitTask">{{ editingTaskId ? 'Сохранить изменения' : 'Добавить задачу' }}</button>
          <button v-if="editingTaskId" class="btn btn-light" @click="cancelEdit">Отмена</button>
        </div>
      </div>

      <!-- ── Quadrant Matrix ── -->
      <div class="card" style="margin-bottom:1.5rem;">
        <div style="display:flex;align-items:center;justify-content:space-between;margin-bottom:1rem;">
          <h2 class="section-title" style="margin:0;">Матрица ценности и усилий</h2>
          <button class="btn btn-light btn-xs" @click="quadrantCollapsed = !quadrantCollapsed">
            {{ quadrantCollapsed ? 'Развернуть' : 'Свернуть' }}
          </button>
        </div>
        <div v-if="!quadrantCollapsed" style="display:grid;grid-template-columns:1fr 1fr;gap:0.75rem;">
          <div v-for="q in quadrantDefs" :key="q.key"
               :style="`background:${q.bg};border:1.5px solid ${q.border};border-radius:10px;padding:1rem;`">
            <p :style="`margin:0 0 0.125rem;font-weight:700;font-size:0.875rem;color:${q.color};`">{{ q.label }}</p>
            <p :style="`margin:0 0 0.625rem;font-size:0.75rem;color:${q.color};opacity:0.7;`">{{ q.desc }}</p>
            <div style="display:flex;flex-wrap:wrap;gap:0.375rem;">
              <span v-for="task in quadrantTasks[q.key]" :key="task.id"
                    :style="`background:rgba(255,255,255,0.7);border:1px solid ${q.border};border-radius:9999px;padding:0.25rem 0.625rem;font-size:0.75rem;font-weight:500;color:${q.color};`">
                {{ task.title }}
              </span>
              <span v-if="!quadrantTasks[q.key]?.length"
                    :style="`font-size:0.75rem;color:${q.color};opacity:0.4;font-style:italic;`">Нет задач</span>
            </div>
          </div>
        </div>
      </div>

      <!-- ── Task List ── -->
      <div class="card">
        <h2 class="section-title">Список задач <span style="font-size:0.875rem;font-weight:400;color:#6b7280;">({{ tasks.length }})</span></h2>
        <div v-if="tasks.length === 0" style="padding:3rem 0;text-align:center;color:#9ca3af;">
          Задач пока нет{{ !isReadOnly ? '. Добавьте первую задачу выше.' : '.' }}
        </div>
        <div v-else style="display:flex;flex-direction:column;gap:0.75rem;">
          <div v-for="task in tasks" :key="task.id"
               style="border:1.5px solid #e5e7eb;border-radius:10px;padding:1rem 1.25rem;background:#fff;transition:border-color .15s;">
            <!-- Task header row -->
            <div style="display:flex;align-items:flex-start;justify-content:space-between;gap:0.75rem;margin-bottom:0.5rem;">
              <p style="margin:0;font-weight:700;font-size:0.9375rem;color:#111827;line-height:1.4;">{{ task.title }}</p>
              <div style="display:flex;gap:0.375rem;flex-shrink:0;flex-wrap:wrap;align-items:center;">
                <span class="badge" :style="typeBadgeStyle(task.type)">{{ task.type === 'tech' ? 'Тех. долг' : 'Бизнес' }}</span>
                <span class="badge" :style="sizeBadgeStyle(task.size)">{{ task.size }}</span>
                <span class="badge" :style="dirBadgeStyle(task.direction)">{{ dirLabel(task.direction) }}</span>
                <span class="badge" :style="statusBadgeStyle(task.registryStatus)">{{ statusLabel(task.registryStatus) }}</span>
              </div>
            </div>
            <!-- Score row -->
            <div style="display:flex;align-items:center;gap:1rem;margin-bottom:0.5rem;">
              <span style="font-weight:700;color:#2563eb;font-size:0.9375rem;">{{ calcValue(task) }}<span style="font-size:0.8rem;color:#9ca3af;font-weight:400;">/21</span></span>
              <span style="font-size:0.8125rem;color:#6b7280;background:#f3f4f6;padding:0.125rem 0.625rem;border-radius:9999px;">{{ getQuadrant(calcValue(task), task.size) }}</span>
            </div>
            <!-- Description -->
            <p v-if="task.description" style="margin:0 0 0.5rem;font-size:0.875rem;color:#4b5563;line-height:1.5;">{{ task.description }}</p>
            <!-- Context row -->
            <p style="margin:0 0 0.75rem;font-size:0.8125rem;color:#9ca3af;">
              <template v-if="task.type === 'tech'">
                <span :style="`color:${riskColor(task.riskLevel)};font-weight:600;`">{{ riskLabel(task.riskLevel) }}</span>
                · Горит: {{ burnLabel(task.burnHorizon) }} · {{ task.techCategory }}
                <template v-if="task.needsDecomposition"> · ⚠ Нужна декомпозиция</template>
                <template v-if="task.needsTechLead"> · 👤 Нужен техлид</template>
              </template>
              <template v-else>
                Инициатор: {{ task.initiator }}
                <template v-if="task.externalDependencyNote"> · {{ task.externalDependencyNote }}</template>
              </template>
            </p>
            <!-- Footer row -->
            <div style="display:flex;align-items:center;justify-content:space-between;flex-wrap:wrap;gap:0.5rem;">
              <span style="font-size:0.75rem;color:#9ca3af;">{{ task.author }} · {{ formatDate(task.createdAt) }}</span>
              <button v-if="!isReadOnly" class="btn btn-light btn-xs" @click="startEdit(task)">Редактировать</button>
            </div>
          </div>
        </div>
      </div>

      <!-- ── Export / Import ── -->
      <div v-if="!isReadOnly" style="margin-top:1rem;display:flex;justify-content:flex-end;gap:0.75rem;">
        <button class="btn btn-light btn-sm" @click="exportJSON">⬇ Экспорт в JSON</button>
        <button class="btn btn-light btn-sm" @click="triggerImport">⬆ Импорт из файла</button>
        <input ref="importInput" type="file" accept=".json" style="display:none;" @change="importJSON" />
      </div>

    </div>
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted } from 'vue'

const tasks = ref([])
const isReadOnly = ref(false)
const editingTaskId = ref(null)
const quadrantCollapsed = ref(false)
const importInput = ref(null)
const preview = ref(null)

const defaultForm = () => ({
  type: 'business', title: '', description: '', direction: 'web',
  size: 'S', processScore: '1', delayScore: '1', impactScore: '1', deadlineScore: '1',
  author: '', registryStatus: 'new',
  // tech
  consequence: '', riskLevel: 'critical', burnHorizon: '1_3m',
  techCategory: 'архитектура', needsDecomposition: 'false', needsTechLead: 'false',
  // business
  businessGoal: '', expectedOutcome: '', initiator: 'Руководитель',
  externalDependencyNote: '', definitionOfDone: ''
})

const form = reactive(defaultForm())

// ─── quadrant config ──────────────────────────────────────────────────────────
const quadrantDefs = [
  { key: 'first',    label: 'Делаем первыми',  desc: 'Высокая ценность · Малые усилия',   bg: '#f0fdf4', border: '#86efac', color: '#166534' },
  { key: 'plan',     label: 'Планируем',        desc: 'Высокая ценность · Большие усилия', bg: '#eff6ff', border: '#93c5fd', color: '#1e3a8a' },
  { key: 'residual', label: 'По остаточному',   desc: 'Низкая ценность · Малые усилия',   bg: '#f9fafb', border: '#d1d5db', color: '#374151' },
  { key: 'no',       label: 'Пока не делаем',  desc: 'Низкая ценность · Большие усилия', bg: '#fff7ed', border: '#fdba74', color: '#9a3412' }
]

const quadrantTasks = computed(() => {
  const map = { first: [], plan: [], residual: [], no: [] }
  tasks.value.forEach(task => {
    const v = calcValue(task)
    const q = getQuadrant(v, task.size)
    if (q === 'Делаем первыми') map.first.push(task)
    else if (q === 'Планируем') map.plan.push(task)
    else if (q === 'По остаточному') map.residual.push(task)
    else map.no.push(task)
  })
  return map
})

// ─── logic ────────────────────────────────────────────────────────────────────
function calcValue(task) {
  return Number(task.processScore) * 3 + Number(task.delayScore) * 2 + Number(task.impactScore) + Number(task.deadlineScore)
}

function getQuadrant(value, size) {
  const highValue = value >= 14
  const highEffort = size === 'L' || size === 'XL'
  if (highValue && !highEffort) return 'Делаем первыми'
  if (highValue && highEffort) return 'Планируем'
  if (!highValue && !highEffort) return 'По остаточному'
  return 'Пока не делаем'
}

function getRecommendation(q) {
  return { 'Делаем первыми': 'Берём в ближайший спринт', 'Планируем': 'Декомпозируем и планируем', 'По остаточному': 'Берём когда нет важных задач', 'Пока не делаем': 'Отказываемся или переформулируем' }[q] || ''
}

function updatePreview() {
  const v = Number(form.processScore) * 3 + Number(form.delayScore) * 2 + Number(form.impactScore) + Number(form.deadlineScore)
  const q = getQuadrant(v, form.size)
  preview.value = { value: v, quadrant: q, recommendation: getRecommendation(q) }
}

// ─── badges ───────────────────────────────────────────────────────────────────
function typeBadgeStyle(type) {
  return type === 'tech'
    ? 'background:#f3e8ff;color:#6d28d9;border:1px solid #d8b4fe;'
    : 'background:#dbeafe;color:#1d4ed8;border:1px solid #93c5fd;'
}
function sizeBadgeStyle(size) {
  const m = { S: 'background:#dcfce7;color:#166534;border:1px solid #86efac;', M: 'background:#fef9c3;color:#854d0e;border:1px solid #fde047;', L: 'background:#fee2e2;color:#991b1b;border:1px solid #fca5a5;', XL: 'background:#fce7f3;color:#9d174d;border:1px solid #f9a8d4;' }
  return m[size] || ''
}
function dirBadgeStyle() { return 'background:#f1f5f9;color:#475569;border:1px solid #e2e8f0;' }
function statusBadgeStyle(s) {
  const m = { new: 'background:#f8fafc;color:#64748b;border:1px solid #cbd5e1;', candidate: 'background:#eff6ff;color:#2563eb;border:1px solid #93c5fd;', planned: 'background:#f0fdf4;color:#16a34a;border:1px solid #86efac;', in_progress: 'background:#fffbeb;color:#d97706;border:1px solid #fcd34d;', done: 'background:#dcfce7;color:#15803d;border:1px solid #4ade80;', deferred: 'background:#fafafa;color:#a3a3a3;border:1px solid #e5e5e5;' }
  return m[s] || ''
}
function riskColor(r) { return { critical: '#dc2626', high: '#ea580c', medium: '#d97706', low: '#16a34a' }[r] || '#6b7280' }
function riskLabel(r) { return { critical: 'Критичный', high: 'Высокий', medium: 'Средний', low: 'Низкий' }[r] || r }
function burnLabel(b) { return { already: 'Уже горит', '1_3m': '1–3 мес', '3_6m': '3–6 мес', '6_12m': '6–12 мес', someday: 'Когда-нибудь' }[b] || b }
function statusLabel(s) { return { new: 'Новая', candidate: 'Кандидат', planned: 'Запланирована', in_progress: 'В работе', done: 'Закрыта', deferred: 'Отложена' }[s] || s }
function dirLabel(d) { return { web: 'Web', onec: '1С', qa: 'QA', common: 'Общее' }[d] || d }
function formatDate(iso) { return new Date(iso).toLocaleDateString('ru-RU') }

// ─── CRUD ─────────────────────────────────────────────────────────────────────
function formToTask() {
  const base = {
    type: form.type, title: form.title, description: form.description,
    direction: form.direction, size: form.size,
    processScore: Number(form.processScore), delayScore: Number(form.delayScore),
    impactScore: Number(form.impactScore), deadlineScore: Number(form.deadlineScore),
    author: form.author, registryStatus: form.registryStatus
  }
  if (form.type === 'tech') return { ...base, consequence: form.consequence, riskLevel: form.riskLevel, burnHorizon: form.burnHorizon, techCategory: form.techCategory, needsDecomposition: form.needsDecomposition === 'true', needsTechLead: form.needsTechLead === 'true' }
  return { ...base, businessGoal: form.businessGoal, expectedOutcome: form.expectedOutcome, initiator: form.initiator, externalDependencyNote: form.externalDependencyNote, definitionOfDone: form.definitionOfDone }
}

function submitTask() {
  if (!form.title.trim()) { alert('Введите название задачи'); return }
  if (editingTaskId.value) {
    const idx = tasks.value.findIndex(t => t.id === editingTaskId.value)
    if (idx !== -1) tasks.value[idx] = { ...tasks.value[idx], ...formToTask(), updatedAt: new Date().toISOString() }
    editingTaskId.value = null
  } else {
    tasks.value.push({ id: Date.now(), ...formToTask(), createdAt: new Date().toISOString(), updatedAt: new Date().toISOString() })
  }
  saveToLocalStorage()
  resetForm()
}

function startEdit(task) {
  editingTaskId.value = task.id
  Object.assign(form, {
    type: task.type, title: task.title, description: task.description,
    direction: task.direction, size: task.size,
    processScore: String(task.processScore), delayScore: String(task.delayScore),
    impactScore: String(task.impactScore), deadlineScore: String(task.deadlineScore),
    author: task.author, registryStatus: task.registryStatus,
    consequence: task.consequence || '', riskLevel: task.riskLevel || 'critical',
    burnHorizon: task.burnHorizon || '1_3m', techCategory: task.techCategory || 'архитектура',
    needsDecomposition: task.needsDecomposition ? 'true' : 'false',
    needsTechLead: task.needsTechLead ? 'true' : 'false',
    businessGoal: task.businessGoal || '', expectedOutcome: task.expectedOutcome || '',
    initiator: task.initiator || 'Руководитель', externalDependencyNote: task.externalDependencyNote || '',
    definitionOfDone: task.definitionOfDone || ''
  })
  updatePreview()
  window.scrollTo({ top: 0, behavior: 'smooth' })
}

function cancelEdit() { editingTaskId.value = null; resetForm() }
function resetForm() { Object.assign(form, defaultForm()); preview.value = null; updatePreview() }
function enterEditMode() { isReadOnly.value = false }
function exitEditMode() { isReadOnly.value = true; cancelEdit() }

function saveToLocalStorage() { localStorage.setItem('taskRegistry', JSON.stringify(tasks.value)) }

function exportJSON() {
  const blob = new Blob([JSON.stringify({ registryName: 'Task Registry', publishedAt: new Date().toISOString(), version: '1.0', items: tasks.value }, null, 2)], { type: 'application/json' })
  const a = document.createElement('a'); a.href = URL.createObjectURL(blob); a.download = 'registry-data.json'; a.click()
}
function triggerImport() { importInput.value?.click() }
function importJSON(e) {
  const file = e.target.files[0]; if (!file) return
  const reader = new FileReader()
  reader.onload = ev => { try { const d = JSON.parse(ev.target.result); tasks.value = d.items || []; saveToLocalStorage() } catch { alert('Ошибка импорта') } }
  reader.readAsText(file); e.target.value = ''
}

onMounted(async () => {
  try {
    const res = await fetch('registry-data.json')
    if (res.ok) { const d = await res.json(); tasks.value = d.items || []; isReadOnly.value = true; return }
  } catch {}
  const stored = localStorage.getItem('taskRegistry')
  if (stored) tasks.value = JSON.parse(stored)
  updatePreview()
})
</script>

<style scoped>
.card {
  background: #fff;
  border: 1.5px solid #e5e7eb;
  border-radius: 14px;
  padding: 1.5rem;
  box-shadow: 0 1px 3px rgba(0,0,0,.06);
}
.section-title {
  margin: 0 0 1rem;
  font-size: 1.0625rem;
  font-weight: 700;
  color: #111827;
}
.group-label {
  margin: 0 0 0.75rem;
  font-size: 0.75rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: .06em;
  color: #6b7280;
}
.form-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
}
@media (max-width: 600px) {
  .form-grid { grid-template-columns: 1fr; }
}
.field { display: flex; flex-direction: column; }
.score-weight {
  font-size: 0.7rem;
  color: #9ca3af;
  font-weight: 400;
  background: #f3f4f6;
  padding: 0.1em 0.4em;
  border-radius: 4px;
  margin-left: 4px;
  vertical-align: middle;
}
</style>
