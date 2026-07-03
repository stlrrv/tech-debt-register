<template>
  <div class="page-shell">
    <div class="page-inner">
      <section class="hero">
        <div>
          <div class="hero-badge">Zezeba Registry</div>
          <h1 class="hero-title">Реестр приоритезации задач</h1>
          <p class="hero-subtitle">
            Прозрачный реестр бизнес-задач и технических задач: что уже в
            спринте, а что стоит в общей очереди.
          </p>
        </div>
        <button
          v-if="isReadOnly"
          class="btn btn-light"
          type="button"
          @click="enterEditMode"
        >
          Войти в режим редактирования
        </button>
        <button
          v-else
          class="btn btn-light"
          type="button"
          @click="exitEditMode"
        >
          Выйти из режима редактирования
        </button>
      </section>

      <section v-if="!isReadOnly" class="card">
        <div class="section-head">
          <div>
            <div class="card-kicker">Редактирование</div>
            <h2 class="section-title">
              {{ editingTaskId ? "Изменение задачи" : "Новая задача" }}
            </h2>
          </div>
        </div>

        <div class="group-block">
          <h3 class="group-title">Шаблоны быстрого старта</h3>
          <div class="template-grid">
            <button
              v-for="preset in taskTemplates"
              :key="preset.key"
              type="button"
              class="template-card"
              :class="{ active: activeTemplateKey === preset.key }"
              @click="applyTemplate(preset.key)"
            >
              <span class="template-type">{{ preset.typeLabel }}</span>
              <strong>{{ preset.label }}</strong>
              <span>{{ preset.desc }}</span>
            </button>
          </div>
        </div>

        <div class="group-block">
          <h3 class="group-title">Основная информация</h3>
          <div class="form-grid">
            <label class="field">
              <span class="field-label">Куда добавить задачу</span>
              <select
                v-model="form.placement"
                class="field-input"
                @change="syncStatusForPlacement"
              >
                <option value="queue">В очередь</option>
                <option value="sprint">В спринт</option>
              </select>
            </label>
            <label class="field">
              <span class="field-label">Тип задачи</span>
              <select v-model="form.type" class="field-input" @change="updatePreview">
                <option value="business">Бизнес-задача</option>
                <option value="tech">Техническая задача</option>
              </select>
            </label>
            <label class="field">
              <span class="field-label">Номер задачи B24</span>
              <input
                v-model="form.taskNumber"
                class="field-input"
                type="text"
                inputmode="numeric"
                placeholder="3322843"
              />
            </label>
            <label class="field field-span-2">
              <span class="field-label">Название</span>
              <input
                v-model="form.title"
                class="field-input"
                type="text"
                placeholder="Короткое название"
              />
            </label>
            <label class="field field-span-2">
              <span class="field-label">Описание / контекст</span>
              <textarea
                v-model="form.description"
                class="field-input"
                rows="3"
                placeholder="Что происходит сейчас и почему задача нужна"
              ></textarea>
            </label>
            <label class="field">
              <span class="field-label">Отдел-заказчик</span>
              <select v-model="form.customerDepartment" class="field-input">
                <option
                  v-for="department in customerDepartments"
                  :key="department"
                  :value="department"
                >
                  {{ department }}
                </option>
              </select>
            </label>
            <label class="field">
              <span class="field-label">Направление</span>
              <select v-model="form.direction" class="field-input">
                <option value="web">Web</option>
                <option value="onec">1С</option>
                <option value="common">Общее</option>
              </select>
            </label>
            <label class="field">
              <span class="field-label">Статус в реестре</span>
              <select
                v-model="form.registryStatus"
                class="field-input"
                @change="syncPlacementWithStatus"
              >
                <option
                  v-for="option in statusOptionsForForm"
                  :key="option.value"
                  :value="option.value"
                >
                  {{ option.label }}
                </option>
              </select>
            </label>
            <label v-if="form.placement === 'queue'" class="field">
              <span class="field-label">Позиция в очереди</span>
              <input
                v-model="form.queuePosition"
                class="field-input"
                type="number"
                min="1"
                placeholder="Например: 17"
              />
            </label>
          </div>
        </div>

        <div class="group-block">
          <h3 class="group-title">Приоритезация</h3>
          <div class="form-grid">
            <label class="field">
              <span class="field-label">Усилия</span>
              <select v-model="form.size" class="field-input" @change="updatePreview">
                <option value="S">S — до 4 ч</option>
                <option value="M">M — 1–2 дня</option>
                <option value="L">L — 3–5 дней</option>
                <option value="XL">XL — неделя+</option>
              </select>
            </label>
            <label class="field">
              <span class="field-label">Откладывание ×2</span>
              <select v-model="form.delayScore" class="field-input" @change="updatePreview">
                <option value="1">1 — ничего не изменится</option>
                <option value="2">2 — накопится долг</option>
                <option value="3">3 — проблема уже есть</option>
              </select>
            </label>
            <label class="field">
              <span class="field-label">Охват ×1</span>
              <select v-model="form.impactScore" class="field-input" @change="updatePreview">
                <option value="1">1 — точечная задача</option>
                <option value="2">2 — затрагивает команду / отдел</option>
                <option value="3">3 — влияет на несколько ролей / систем</option>
              </select>
            </label>
            <label class="field">
              <span class="field-label">Дедлайн ×1</span>
              <select v-model="form.deadlineScore" class="field-input" @change="updatePreview">
                <option value="3">3 — жесткий внешний срок</option>
                <option value="2">2 — желательно в ближайшее время</option>
                <option value="1">1 — без жесткого срока</option>
              </select>
            </label>
          </div>

          <div class="fact-grid">
            <label class="field">
              <span class="field-label">Операций в день</span>
              <input
                v-model="form.operationsPerDay"
                class="field-input"
                type="number"
                min="0"
                step="1"
                @input="updatePreview"
              />
            </label>
            <label class="field">
              <span class="field-label">Время на операцию, мин</span>
              <input
                v-model="form.minutesPerOperation"
                class="field-input"
                type="number"
                min="0"
                step="1"
                @input="updatePreview"
              />
            </label>
            <label class="field">
              <span class="field-label">Кол-во человек</span>
              <input
                v-model="form.peopleCount"
                class="field-input"
                type="number"
                min="0"
                step="1"
                @input="updatePreview"
              />
            </label>
            <label class="field">
              <span class="field-label">Рабочих дней в мес</span>
              <input
                v-model="form.workDaysPerMonth"
                class="field-input"
                type="number"
                min="0"
                step="1"
                @input="updatePreview"
              />
            </label>
            <label class="field">
              <span class="field-label">Средняя стоимость часа, ₽</span>
              <input
                v-model="form.hourlyRate"
                class="field-input"
                type="number"
                min="0"
                step="100"
                @input="updatePreview"
              />
            </label>
          </div>

          <div class="preview-panel">
            <div class="preview-item">
              <span>Часы / мес</span>
              <strong>{{ formatNumber(previewMetrics.hours) }}</strong>
            </div>
            <div class="preview-item">
              <span>Деньги / мес</span>
              <strong>{{ formatMoney(previewMetrics.money) }}</strong>
            </div>
            <div class="preview-item">
              <span>Процесс ×3</span>
              <strong>{{ processBandLabel(previewMetrics.processScore) }}</strong>
            </div>
            <div class="preview-item">
              <span>Ценность</span>
              <strong>{{ previewMetrics.value }}/21</strong>
            </div>
            <div class="preview-item">
              <span>Квадрант</span>
              <strong>{{ previewMetrics.quadrant }}</strong>
            </div>
          </div>
        </div>

        <template v-if="form.type === 'business'">
          <div class="group-block">
            <h3 class="group-title">Бизнес-контекст</h3>
            <div class="form-grid">
              <label class="field field-span-2">
                <span class="field-label">Бизнес-цель</span>
                <textarea
                  v-model="form.businessGoal"
                  class="field-input"
                  rows="3"
                  placeholder="Что хотим получить для бизнеса"
                ></textarea>
              </label>
              <label class="field field-span-2">
                <span class="field-label">Ожидаемый результат</span>
                <textarea
                  v-model="form.expectedOutcome"
                  class="field-input"
                  rows="3"
                  placeholder="Как изменится процесс после выполнения"
                ></textarea>
              </label>
              <label class="field field-span-2">
                <span class="field-label">Внешняя зависимость</span>
                <textarea
                  v-model="form.externalDependencyNote"
                  class="field-input"
                  rows="2"
                  placeholder="Есть ли внешняя дата, отдел или событие"
                ></textarea>
              </label>
            </div>
          </div>
        </template>

        <template v-else>
          <div class="group-block">
            <h3 class="group-title">Технический контекст</h3>
            <div class="form-grid">
              <label class="field field-span-2">
                <span class="field-label">Последствие</span>
                <textarea
                  v-model="form.consequence"
                  class="field-input"
                  rows="3"
                  placeholder="Если не сделать, то..."
                ></textarea>
              </label>
              <label class="field">
                <span class="field-label">Уровень риска</span>
                <select v-model="form.riskLevel" class="field-input">
                  <option value="critical">Критичный</option>
                  <option value="high">Высокий</option>
                  <option value="medium">Средний</option>
                  <option value="low">Низкий</option>
                </select>
              </label>
              <label class="field">
                <span class="field-label">Горит через</span>
                <select v-model="form.burnHorizon" class="field-input">
                  <option value="already">Уже горит</option>
                  <option value="1_3m">1–3 мес</option>
                  <option value="3_6m">3–6 мес</option>
                  <option value="6_12m">6–12 мес</option>
                  <option value="someday">Когда-нибудь</option>
                </select>
              </label>
              <label class="field">
                <span class="field-label">Тип проблемы</span>
                <select v-model="form.techCategory" class="field-input">
                  <option value="архитектура">Архитектура</option>
                  <option value="производительность">Производительность</option>
                  <option value="зависимости">Зависимости</option>
                  <option value="кодовая база">Кодовая база</option>
                  <option value="инфраструктура">Инфраструктура</option>
                  <option value="безопасность">Безопасность</option>
                  <option value="другое">Другое</option>
                </select>
              </label>
              <label class="field">
                <span class="field-label">Нужна декомпозиция</span>
                <select v-model="form.needsDecomposition" class="field-input">
                  <option value="false">Нет</option>
                  <option value="true">Да</option>
                </select>
              </label>
              <label class="field">
                <span class="field-label">Нужен техлид</span>
                <select v-model="form.needsTechLead" class="field-input">
                  <option value="false">Нет</option>
                  <option value="true">Да</option>
                </select>
              </label>
            </div>
          </div>
        </template>

        <div class="form-actions">
          <button class="btn btn-primary" type="button" @click="submitTask">
            {{
              editingTaskId
                ? "Сохранить изменения"
                : form.placement === "sprint"
                  ? "Добавить задачу в спринт"
                  : "Добавить задачу в очередь"
            }}
          </button>
          <button
            v-if="editingTaskId"
            class="btn btn-light"
            type="button"
            @click="cancelEdit"
          >
            Отмена
          </button>
        </div>
      </section>

      <section class="card">
        <div class="section-head">
          <div>
            <div class="card-kicker">Фильтры и просмотр</div>
            <h2 class="section-title">Текущий срез</h2>
          </div>
        </div>
        <div class="filter-grid">
          <label class="field">
            <span class="field-label">Показывать</span>
            <select v-model="typeFilter" class="field-input">
              <option value="business">Бизнес</option>
              <option value="tech">Технические</option>
              <option value="all">Все</option>
            </select>
          </label>
          <label class="field">
            <span class="field-label">Направление</span>
            <select v-model="directionFilter" class="field-input">
              <option value="all">Все направления</option>
              <option value="web">Web</option>
              <option value="onec">1С</option>
              <option value="common">Общее</option>
            </select>
          </label>
          <label class="field">
            <span class="field-label">Отдел-заказчик</span>
            <select v-model="departmentFilter" class="field-input">
              <option value="all">Все отделы</option>
              <option
                v-for="department in customerDepartments"
                :key="department"
                :value="department"
              >
                {{ department }}
              </option>
            </select>
          </label>
          <label class="field">
            <span class="field-label">Сортировка списка</span>
            <select v-model="sortBy" class="field-input">
              <option value="priority">По ценности</option>
              <option value="status">По статусу</option>
            </select>
          </label>
        </div>
      </section>

      <section class="card">
        <div class="section-head">
          <div>
            <div class="card-kicker">Спринт</div>
            <h2 class="section-title">Квадранты текущего спринта</h2>
            <p class="section-caption">
              Здесь показываются только задачи со статусами «В спринте» и
              «В работе».
            </p>
          </div>
          <button class="btn btn-light" type="button" @click="quadrantCollapsed = !quadrantCollapsed">
            {{ quadrantCollapsed ? "Показать квадранты" : "Свернуть квадранты" }}
          </button>
        </div>

        <div v-if="!quadrantCollapsed" class="quadrant-grid">
          <article
            v-for="quadrant in quadrantDefs"
            :key="quadrant.key"
            class="quadrant-card"
            :style="quadrantStyle(quadrant)"
          >
            <div class="quadrant-head">
              <div>
                <h3>{{ quadrant.label }}</h3>
                <p>{{ quadrant.desc }}</p>
              </div>
              <span class="quadrant-count">{{ sprintQuadrants[quadrant.key].length }}</span>
            </div>

            <div v-if="sprintQuadrants[quadrant.key].length" class="dot-wrap">
              <button
                v-for="task in sprintQuadrants[quadrant.key]"
                :key="task.id"
                type="button"
                class="task-dot"
                :style="taskDotStyle(task)"
                :title="task.title"
                @click="handleTaskDotClick(task)"
              >
                {{ task.title }}
              </button>
            </div>
            <p v-else class="empty-state">В этом квадранте пока нет задач.</p>
          </article>
        </div>
      </section>

      <section class="card">
        <div class="section-head">
          <div>
            <div class="card-kicker">Очередь</div>
            <h2 class="section-title">Общий бэклог</h2>
            <p class="section-caption">
              Плоский список задач вне спринта, отсортированный по позиции в
              очереди.
            </p>
          </div>
        </div>

        <div v-if="visibleQueueTasks.length" class="queue-list">
          <article v-for="task in visibleQueueTasks" :key="task.id" class="queue-item">
            <div class="queue-main">
              <div class="queue-position">
                <template v-if="!isReadOnly">
                  <input
                    class="queue-position-input"
                    :value="task.queuePosition ?? ''"
                    type="number"
                    min="1"
                    @change="updateQueuePosition(task.id, $event.target.value)"
                  />
                </template>
                <template v-else>
                  {{ task.queuePosition || "—" }}
                </template>
              </div>
              <div class="queue-body">
                <div class="queue-title-row">
                  <strong>{{ taskLabel(task) }}</strong>
                  <span class="value-chip">{{ calcValue(task) }}/21</span>
                </div>
                <div class="chip-row">
                  <span class="chip">{{ task.type === "tech" ? "Техническая" : "Бизнес" }}</span>
                  <span class="chip">{{ dirLabel(task.direction) }}</span>
                  <span class="chip">{{ task.customerDepartment }}</span>
                  <span class="chip">{{ statusLabel(task.registryStatus) }}</span>
                  <span class="chip">{{ processBandLabel(getProcessScore(task)) }}</span>
                </div>
                <p class="queue-text">{{ task.description || "Описание не заполнено." }}</p>
              </div>
            </div>
            <div v-if="!isReadOnly" class="row-actions">
              <button class="btn btn-light" type="button" @click="startEdit(task)">
                Редактировать
              </button>
              <button class="btn btn-danger" type="button" @click="deleteTask(task.id)">
                Удалить
              </button>
            </div>
          </article>
        </div>
        <p v-else class="empty-state">В очереди нет задач для текущих фильтров.</p>
      </section>

      <section class="card">
        <div class="section-head">
          <div>
            <div class="card-kicker">Все задачи</div>
            <h2 class="section-title">Полный список</h2>
          </div>
        </div>
        <div v-if="sortedTasks.length" class="list-stack">
          <article v-for="task in sortedTasks" :key="task.id" class="list-card">
            <div class="list-main">
              <div class="list-title-row">
                <strong>{{ taskLabel(task) }}</strong>
                <span class="value-chip">{{ calcValue(task) }}/21</span>
              </div>
              <div class="chip-row">
                <span class="chip">{{ task.type === "tech" ? "Техническая" : "Бизнес" }}</span>
                <span class="chip">{{ dirLabel(task.direction) }}</span>
                <span class="chip">{{ task.customerDepartment }}</span>
                <span class="chip">{{ statusLabel(task.registryStatus) }}</span>
                <span class="chip">{{ task.size }}</span>
                <span class="chip">{{ processBandLabel(getProcessScore(task)) }}</span>
              </div>
              <p class="queue-text">{{ task.description || "Описание не заполнено." }}</p>
            </div>
            <div v-if="!isReadOnly" class="row-actions">
              <button class="btn btn-light" type="button" @click="startEdit(task)">
                Редактировать
              </button>
              <button class="btn btn-danger" type="button" @click="deleteTask(task.id)">
                Удалить
              </button>
            </div>
          </article>
        </div>
        <p v-else class="empty-state">По текущим фильтрам задач не найдено.</p>
      </section>

      <section class="meta-grid meta-grid-single">
        <article class="card">
          <div class="card-kicker">Сводка по текущему срезу</div>
          <div class="summary-actions">
            <button class="btn btn-secondary" type="button" @click="copySummary">
              Скопировать сводку
            </button>
            <button
              v-if="!isReadOnly"
              class="btn btn-light"
              type="button"
              @click="exportJSON"
            >
              Экспорт JSON
            </button>
            <button
              v-if="!isReadOnly"
              class="btn btn-light"
              type="button"
              @click="triggerImport"
            >
              Импорт JSON
            </button>
            <input
              ref="importInput"
              type="file"
              accept="application/json"
              style="display: none"
              @change="importJSON"
            />
          </div>
          <pre class="summary-box">{{ summaryText }}</pre>
        </article>
      </section>
    </div>
  </div>
</template>

<script setup>
import { computed, onMounted, reactive, ref, watch } from "vue";

const customerDepartments = [
  "Логистика",
  "Склад",
  "Склад и Логистика",
  "Продажи",
  "Маркетплейсы",
  "Бухгалтерия",
  "Сервис",
  "Метрология",
  "Маркетинг",
  "Развитие",
  "Закупки",
  "Юристы",
];

const quadrantDefs = [
  {
    key: "first",
    label: "Делаем первыми",
    desc: "Высокая ценность, умеримые усилия",
    bg: "#ecfdf5",
    border: "#86efac",
    color: "#166534",
  },
  {
    key: "plan",
    label: "Планируем",
    desc: "Высокая ценность, большие усилия",
    bg: "#eff6ff",
    border: "#93c5fd",
    color: "#1d4ed8",
  },
  {
    key: "residual",
    label: "По остаточному",
    desc: "Низкая ценность, умеримые усилия",
    bg: "#f8fafc",
    border: "#cbd5e1",
    color: "#334155",
  },
  {
    key: "no",
    label: "Пока не делаем",
    desc: "Низкая ценность, большие усилия",
    bg: "#fff7ed",
    border: "#fdba74",
    color: "#9a3412",
  },
];

const taskTemplates = [
  {
    key: "business-fast",
    typeLabel: "Бизнес",
    label: "Спринт / быстрый эффект",
    desc: "Для задачи с явной выгодой и небольшими усилиями",
    values: {
      type: "business",
      size: "S",
      delayScore: "3",
      impactScore: "2",
      deadlineScore: "3",
      registryStatus: "planned",
      operationsPerDay: "10",
      minutesPerOperation: "10",
      peopleCount: "1",
      workDaysPerMonth: "22",
      hourlyRate: "700",
    },
  },
  {
    key: "business-queue",
    typeLabel: "Бизнес",
    label: "Бэклог / оценить позже",
    desc: "Для идеи без срочного окна, но с понятной выгодой",
    values: {
      type: "business",
      size: "M",
      delayScore: "2",
      impactScore: "2",
      deadlineScore: "1",
      registryStatus: "candidate",
      operationsPerDay: "5",
      minutesPerOperation: "5",
      peopleCount: "1",
      workDaysPerMonth: "22",
      hourlyRate: "600",
    },
  },
  {
    key: "tech-sprint",
    typeLabel: "Тех",
    label: "Спринт / технический риск",
    desc: "Для технической задачи, которая уже мешает работе",
    values: {
      type: "tech",
      size: "M",
      delayScore: "3",
      impactScore: "2",
      deadlineScore: "2",
      registryStatus: "planned",
      operationsPerDay: "20",
      minutesPerOperation: "15",
      peopleCount: "2",
      workDaysPerMonth: "22",
      hourlyRate: "700",
    },
  },
  {
    key: "tech-backlog",
    typeLabel: "Тех",
    label: "Бэклог / стабилизация",
    desc: "Для накопившегося техдолга вне текущего спринта",
    values: {
      type: "tech",
      size: "L",
      delayScore: "2",
      impactScore: "1",
      deadlineScore: "1",
      registryStatus: "new",
      operationsPerDay: "3",
      minutesPerOperation: "20",
      peopleCount: "1",
      workDaysPerMonth: "22",
      hourlyRate: "700",
    },
  },
];

const importInput = ref(null);
const isReadOnly = ref(true);
const quadrantCollapsed = ref(false);
const editingTaskId = ref(null);
const activeTemplateKey = ref("");
const tasks = ref([]);
const typeFilter = ref("business");
const directionFilter = ref("all");
const departmentFilter = ref("all");
const sortBy = ref("priority");

const registryMeta = reactive({
  registryName: "Zezeba Task Registry",
  version: "2.0",
  updatedAt: "",
});

function defaultForm() {
  return {
    placement: "queue",
    type: "business",
    taskNumber: "",
    title: "",
    description: "",
    customerDepartment: customerDepartments[0],
    direction: "web",
    registryStatus: "candidate",
    queuePosition: "",
    size: "M",
    delayScore: "2",
    impactScore: "2",
    deadlineScore: "2",
    operationsPerDay: "0",
    minutesPerOperation: "0",
    peopleCount: "0",
    workDaysPerMonth: "22",
    hourlyRate: "0",
    businessGoal: "",
    expectedOutcome: "",
    externalDependencyNote: "",
    consequence: "",
    riskLevel: "medium",
    burnHorizon: "1_3m",
    techCategory: "архитектура",
    needsDecomposition: "false",
    needsTechLead: "false",
  };
}

const form = reactive(defaultForm());

const previewMetrics = computed(() => getTaskMetrics(form));

const statusOptionsForForm = computed(() => {
  if (form.placement === "sprint") {
    return [
      { value: "planned", label: "В спринте" },
      { value: "in_progress", label: "В работе" },
    ];
  }

  return [
    { value: "new", label: "Новая" },
    { value: "candidate", label: "Кандидат" },
    { value: "deferred", label: "Отложена" },
  ];
});

const filteredTasks = computed(() =>
  tasks.value.filter((task) => {
    const matchesType = typeFilter.value === "all" || task.type === typeFilter.value;
    const matchesDirection =
      directionFilter.value === "all" || task.direction === directionFilter.value;
    const matchesDepartment =
      departmentFilter.value === "all" ||
      task.customerDepartment === departmentFilter.value;
    return matchesType && matchesDirection && matchesDepartment;
  }),
);

const sortedTasks = computed(() => {
  const list = [...filteredTasks.value];
  if (sortBy.value === "status") {
    return list.sort((a, b) => {
      const byStatus =
        (statusOrder[a.registryStatus] ?? 99) - (statusOrder[b.registryStatus] ?? 99);
      if (byStatus !== 0) return byStatus;
      return compareByValueDesc(a, b);
    });
  }
  return list.sort(compareByValueDesc);
});

const visibleSprintTasks = computed(() =>
  filteredTasks.value.filter((task) => isSprintStatus(task.registryStatus)),
);

const visibleQueueTasks = computed(() =>
  filteredTasks.value
    .filter((task) => isBacklogStatus(task.registryStatus))
    .sort(compareQueueTasks),
);

const sprintQuadrants = computed(() => {
  const map = { first: [], plan: [], residual: [], no: [] };
  visibleSprintTasks.value.forEach((task) => {
    map[getQuadrantKey(calcValue(task), task.size)].push(task);
  });
  Object.keys(map).forEach((key) => {
    map[key].sort((a, b) => {
      const directionSort = dirLabel(a.direction).localeCompare(dirLabel(b.direction), "ru");
      if (directionSort !== 0) return directionSort;
      return compareByValueDesc(a, b);
    });
  });
  return map;
});

const summaryText = computed(() => {
  const sprintLines = buildSprintSummaryLines();
  const queueLines = buildQueueSummaryLines();

  return [
    "Спринт-старт:",
    sprintLines.length ? sprintLines.join("\n") : "Нет задач в спринте по текущему срезу.",
    "",
    "Очередь:",
    queueLines.length ? queueLines.join("\n") : "Нет задач в очереди по текущему срезу.",
  ].join("\n");
});

const statusOrder = {
  planned: 0,
  in_progress: 1,
  candidate: 2,
  new: 3,
  deferred: 4,
  done: 5,
};

watch(
  registryMeta,
  () => {
    if (!isReadOnly.value) {
      touchMeta();
      saveToLocalStorage();
    }
  },
  { deep: true },
);

watch(
  () => [form.operationsPerDay, form.minutesPerOperation, form.peopleCount, form.workDaysPerMonth, form.hourlyRate, form.delayScore, form.impactScore, form.deadlineScore, form.size],
  () => updatePreview(),
);

function toNumber(value) {
  return Number(value || 0);
}

function getMonthlyHours(task) {
  const totalMinutes =
    toNumber(task.operationsPerDay) *
    toNumber(task.minutesPerOperation) *
    toNumber(task.peopleCount) *
    toNumber(task.workDaysPerMonth);
  return totalMinutes / 60;
}

function getMonthlyMoney(task) {
  return getMonthlyHours(task) * toNumber(task.hourlyRate);
}

function deriveProcessScore(money) {
  if (money > 50000) return 3;
  if (money >= 5000) return 2;
  return 1;
}

function getProcessScore(task) {
  return deriveProcessScore(getMonthlyMoney(task));
}

function calcValue(task) {
  return (
    getProcessScore(task) * 3 +
    toNumber(task.delayScore) * 2 +
    toNumber(task.impactScore) +
    toNumber(task.deadlineScore)
  );
}

function getQuadrantLabel(key) {
  return quadrantDefs.find((item) => item.key === key)?.label || "";
}

function getQuadrantKey(value, size) {
  const highValue = value >= 14;
  const highEffort = size === "L" || size === "XL";
  if (highValue && !highEffort) return "first";
  if (highValue && highEffort) return "plan";
  if (!highValue && !highEffort) return "residual";
  return "no";
}

function getTaskMetrics(task) {
  const hours = getMonthlyHours(task);
  const money = getMonthlyMoney(task);
  const processScore = deriveProcessScore(money);
  const value =
    processScore * 3 +
    toNumber(task.delayScore) * 2 +
    toNumber(task.impactScore) +
    toNumber(task.deadlineScore);
  const quadrantKey = getQuadrantKey(value, task.size);

  return {
    hours,
    money,
    processScore,
    value,
    quadrant: getQuadrantLabel(quadrantKey),
  };
}

function formatMoney(value) {
  return `${Math.round(value).toLocaleString("ru-RU")} ₽`;
}

function formatNumber(value) {
  return Number(value).toLocaleString("ru-RU", {
    maximumFractionDigits: 1,
  });
}

function formatDateTime(value) {
  if (!value) return "Не указано";
  return new Date(value).toLocaleString("ru-RU");
}

function formatDate(value) {
  if (!value) return "Не указано";
  return new Date(value).toLocaleDateString("ru-RU");
}

function statusLabel(status) {
  return (
    {
      new: "Новая",
      candidate: "Кандидат",
      planned: "В спринте",
      in_progress: "В работе",
      done: "Закрыта",
      deferred: "Отложена",
    }[status] || status
  );
}

function dirLabel(direction) {
  return (
    {
      web: "Web",
      onec: "1С",
      common: "Общее",
    }[direction] || direction
  );
}

function processBandLabel(score) {
  return (
    {
      1: "Процесс: <5000 ₽/мес",
      2: "Процесс: 5000–50000 ₽/мес",
      3: "Процесс: >50000 ₽/мес",
    }[score] || "Процесс: не рассчитан"
  );
}

function shortId(id) {
  return String(id).slice(-3);
}

function taskLabel(task) {
  return task.taskNumber ? `#${task.taskNumber} · ${task.title}` : task.title;
}

function compareByValueDesc(a, b) {
  const valueDiff = calcValue(b) - calcValue(a);
  if (valueDiff !== 0) return valueDiff;
  const sizeDiff = sizeRank(a.size) - sizeRank(b.size);
  if (sizeDiff !== 0) return sizeDiff;
  return String(b.updatedAt || "").localeCompare(String(a.updatedAt || ""));
}

function compareQueueTasks(a, b) {
  const aPos = a.queuePosition ?? Number.MAX_SAFE_INTEGER;
  const bPos = b.queuePosition ?? Number.MAX_SAFE_INTEGER;
  if (aPos !== bPos) return aPos - bPos;
  return compareByValueDesc(a, b);
}

function sizeRank(size) {
  return { S: 0, M: 1, L: 2, XL: 3 }[size] ?? 99;
}

function isSprintStatus(status) {
  return status === "planned" || status === "in_progress";
}

function isBacklogStatus(status) {
  return status !== "done" && !isSprintStatus(status);
}

function quadrantStyle(quadrant) {
  return {
    background: quadrant.bg,
    borderColor: quadrant.border,
    color: quadrant.color,
  };
}

function taskDotStyle(task) {
  const score = calcValue(task);
  let background = "#94a3b8";
  let color = "#fff";
  if (score >= 16) background = "#16a34a";
  else if (score >= 13) background = "#2563eb";
  else if (score >= 10) background = "#f59e0b";
  else {
    background = "#ef4444";
    color = "#fff7ed";
  }
  return { background, color };
}

function buildSprintSummaryLines() {
  return [...visibleSprintTasks.value]
    .sort((a, b) => {
      const byDepartment = (a.customerDepartment || "").localeCompare(
        b.customerDepartment || "",
        "ru",
      );
      if (byDepartment !== 0) return byDepartment;
      return compareByValueDesc(a, b);
    })
    .map((task) => {
      const label = task.taskNumber ? `#${task.taskNumber}` : task.title;
      return `${task.customerDepartment} — ${label} — ${statusLabel(task.registryStatus)}`;
    });
}

function buildQueueSummaryLines() {
  const total = visibleQueueTasks.value.length;
  return [...visibleQueueTasks.value]
    .sort(compareQueueTasks)
    .map((task) => {
      const position = task.queuePosition || "позиция не выставлена";
      return `${task.customerDepartment} — ${
        typeof position === "number" ? `позиция ${position}` : position
      } из ${total}`;
    });
}

function updatePreview() {
  return previewMetrics.value;
}

function syncQueuePositionForStatus() {
  if (isSprintStatus(form.registryStatus) || form.registryStatus === "done") {
    form.queuePosition = "";
    return;
  }
  if (!form.queuePosition) {
    form.queuePosition = String(getNextQueuePosition());
  }
}

function syncStatusForPlacement() {
  if (form.placement === "sprint") {
    if (!isSprintStatus(form.registryStatus)) {
      form.registryStatus = "planned";
    }
    form.queuePosition = "";
    return;
  }

  if (!isBacklogStatus(form.registryStatus)) {
    form.registryStatus = "new";
  }
  if (!form.queuePosition) {
    form.queuePosition = String(getNextQueuePosition());
  }
}

function syncPlacementWithStatus() {
  form.placement = isSprintStatus(form.registryStatus) ? "sprint" : "queue";
  syncQueuePositionForStatus();
}

function applyTemplate(templateKey) {
  const preset = taskTemplates.find((item) => item.key === templateKey);
  if (!preset) return;

  const preserve = {
    title: form.title,
    description: form.description,
    taskNumber: form.taskNumber,
    customerDepartment: form.customerDepartment,
    businessGoal: form.businessGoal,
    expectedOutcome: form.expectedOutcome,
    externalDependencyNote: form.externalDependencyNote,
    consequence: form.consequence,
  };

  Object.assign(form, defaultForm(), preset.values, preserve);
  activeTemplateKey.value = templateKey;
  form.placement = isSprintStatus(form.registryStatus) ? "sprint" : "queue";
  syncQueuePositionForStatus();
  updatePreview();
}

function touchMeta() {
  registryMeta.updatedAt = new Date().toISOString();
}

function getNextQueuePosition() {
  const positions = tasks.value
    .map((task) => Number(task.queuePosition || 0))
    .filter((value) => value > 0);
  return positions.length ? Math.max(...positions) + 1 : 1;
}

function formToTask() {
  const base = {
    type: form.type,
    taskNumber: String(form.taskNumber || "").trim(),
    title: form.title.trim(),
    description: form.description.trim(),
    customerDepartment: form.customerDepartment,
    direction: form.direction,
    registryStatus: form.registryStatus,
    queuePosition:
      isBacklogStatus(form.registryStatus) && Number(form.queuePosition) > 0
        ? Number(form.queuePosition)
        : null,
    size: form.size,
    delayScore: toNumber(form.delayScore),
    impactScore: toNumber(form.impactScore),
    deadlineScore: toNumber(form.deadlineScore),
    operationsPerDay: toNumber(form.operationsPerDay),
    minutesPerOperation: toNumber(form.minutesPerOperation),
    peopleCount: toNumber(form.peopleCount),
    workDaysPerMonth: toNumber(form.workDaysPerMonth),
    hourlyRate: toNumber(form.hourlyRate),
    processScore: getProcessScore(form),
    processMonthlyHours: getMonthlyHours(form),
    processMonthlyMoney: getMonthlyMoney(form),
  };

  if (form.type === "tech") {
    return {
      ...base,
      consequence: form.consequence.trim(),
      riskLevel: form.riskLevel,
      burnHorizon: form.burnHorizon,
      techCategory: form.techCategory,
      needsDecomposition: form.needsDecomposition === "true",
      needsTechLead: form.needsTechLead === "true",
      businessGoal: "",
      expectedOutcome: "",
      externalDependencyNote: "",
    };
  }

  return {
    ...base,
    businessGoal: form.businessGoal.trim(),
    expectedOutcome: form.expectedOutcome.trim(),
    externalDependencyNote: form.externalDependencyNote.trim(),
    consequence: "",
    riskLevel: "medium",
    burnHorizon: "1_3m",
    techCategory: "",
    needsDecomposition: false,
    needsTechLead: false,
  };
}

function submitTask() {
  if (!form.title.trim()) {
    window.alert("Введите название задачи");
    return;
  }

  if (!form.customerDepartment) {
    window.alert("Укажите отдел-заказчик");
    return;
  }

  const taskPayload = formToTask();
  touchMeta();

  if (editingTaskId.value) {
    const index = tasks.value.findIndex((task) => task.id === editingTaskId.value);
    if (index !== -1) {
      tasks.value[index] = {
        ...tasks.value[index],
        ...taskPayload,
        updatedAt: new Date().toISOString(),
      };
    }
  } else {
    tasks.value.push({
      id: Date.now(),
      ...taskPayload,
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString(),
    });
  }

  editingTaskId.value = null;
  saveToLocalStorage();
  resetForm();
}

function startEdit(task) {
  editingTaskId.value = task.id;
  activeTemplateKey.value = "";
  Object.assign(form, {
    placement: isSprintStatus(task.registryStatus) ? "sprint" : "queue",
    type: task.type || "business",
    taskNumber: task.taskNumber || "",
    title: task.title || "",
    description: task.description || "",
    customerDepartment: task.customerDepartment || customerDepartments[0],
    direction: task.direction || "web",
    registryStatus: task.registryStatus || "candidate",
    queuePosition: task.queuePosition ? String(task.queuePosition) : "",
    size: task.size || "M",
    delayScore: String(task.delayScore ?? 2),
    impactScore: String(task.impactScore ?? 2),
    deadlineScore: String(task.deadlineScore ?? 2),
    operationsPerDay: String(task.operationsPerDay ?? 0),
    minutesPerOperation: String(task.minutesPerOperation ?? 0),
    peopleCount: String(task.peopleCount ?? 0),
    workDaysPerMonth: String(task.workDaysPerMonth ?? 22),
    hourlyRate: String(task.hourlyRate ?? 0),
    businessGoal: task.businessGoal || "",
    expectedOutcome: task.expectedOutcome || "",
    externalDependencyNote: task.externalDependencyNote || "",
    consequence: task.consequence || "",
    riskLevel: task.riskLevel || "medium",
    burnHorizon: task.burnHorizon || "1_3m",
    techCategory: task.techCategory || "архитектура",
    needsDecomposition: task.needsDecomposition ? "true" : "false",
    needsTechLead: task.needsTechLead ? "true" : "false",
  });
  window.scrollTo({ top: 0, behavior: "smooth" });
}

function deleteTask(taskId) {
  if (!window.confirm("Удалить задачу из реестра?")) return;
  tasks.value = tasks.value.filter((task) => task.id !== taskId);
  if (editingTaskId.value === taskId) resetForm();
  touchMeta();
  saveToLocalStorage();
}

function updateQueuePosition(taskId, value) {
  const index = tasks.value.findIndex((task) => task.id === taskId);
  if (index === -1) return;
  tasks.value[index] = {
    ...tasks.value[index],
    queuePosition: Number(value) > 0 ? Number(value) : null,
    updatedAt: new Date().toISOString(),
  };
  touchMeta();
  saveToLocalStorage();
}

function cancelEdit() {
  editingTaskId.value = null;
  resetForm();
}

function resetForm() {
  Object.assign(form, defaultForm());
  syncStatusForPlacement();
  activeTemplateKey.value = "";
  updatePreview();
}

function enterEditMode() {
  isReadOnly.value = false;
}

function exitEditMode() {
  isReadOnly.value = true;
  cancelEdit();
}

function handleTaskDotClick(task) {
  if (isReadOnly.value) return;
  startEdit(task);
}

async function copySummary() {
  try {
    await navigator.clipboard.writeText(summaryText.value);
    window.alert("Сводка скопирована");
  } catch {
    window.alert("Не удалось скопировать сводку");
  }
}

function saveToLocalStorage() {
  const payload = buildExportPayload();
  localStorage.setItem("taskRegistryData", JSON.stringify(payload));
}

function buildExportPayload() {
  return {
    meta: {
      registryName: registryMeta.registryName,
      version: registryMeta.version,
      updatedAt: registryMeta.updatedAt || new Date().toISOString(),
    },
    items: tasks.value,
  };
}

function exportJSON() {
  touchMeta();
  const blob = new Blob([JSON.stringify(buildExportPayload(), null, 2)], {
    type: "application/json",
  });
  const link = document.createElement("a");
  link.href = URL.createObjectURL(blob);
  link.download = "registry-data.json";
  link.click();
}

function triggerImport() {
  importInput.value?.click();
}

function normalizeImportedTask(task) {
  const normalized = {
    placement: isSprintStatus(task.registryStatus) ? "sprint" : "queue",
    ...task,
    customerDepartment: task.customerDepartment || customerDepartments[0],
    direction: task.direction === "qa" ? "common" : task.direction || "web",
    queuePosition:
      task.queuePosition === null || task.queuePosition === undefined
        ? null
        : Number(task.queuePosition),
    operationsPerDay: toNumber(task.operationsPerDay),
    minutesPerOperation: toNumber(task.minutesPerOperation),
    peopleCount: toNumber(task.peopleCount),
    workDaysPerMonth: toNumber(task.workDaysPerMonth || 22),
    hourlyRate: toNumber(task.hourlyRate),
  };

  if (!normalized.operationsPerDay && task.processScore) {
    normalized.processScore = Number(task.processScore);
  }

  normalized.processMonthlyHours = getMonthlyHours(normalized);
  normalized.processMonthlyMoney = getMonthlyMoney(normalized);
  normalized.processScore = getProcessScore(normalized);

  return normalized;
}

function applyImportedPayload(payload) {
  const meta = payload.meta || {
    registryName: payload.registryName || "Zezeba Task Registry",
    version: payload.version || "2.0",
    updatedAt: payload.publishedAt || new Date().toISOString(),
  };

  registryMeta.registryName = meta.registryName || "Zezeba Task Registry";
  registryMeta.version = meta.version || "2.0";
  registryMeta.updatedAt = meta.updatedAt || new Date().toISOString();

  tasks.value = Array.isArray(payload.items)
    ? payload.items.map(normalizeImportedTask)
    : [];
}

function importJSON(event) {
  const file = event.target.files?.[0];
  if (!file) return;

  const reader = new FileReader();
  reader.onload = (loadEvent) => {
    try {
      const payload = JSON.parse(loadEvent.target?.result || "{}");
      applyImportedPayload(payload);
      saveToLocalStorage();
    } catch {
      window.alert("Ошибка импорта JSON");
    }
  };
  reader.readAsText(file);
  event.target.value = "";
}

onMounted(async () => {
  try {
    const response = await fetch("registry-data.json");
    if (response.ok) {
      const payload = await response.json();
      applyImportedPayload(payload);
      isReadOnly.value = true;
      updatePreview();
      return;
    }
  } catch {
    // ignore fetch errors for local preview
  }

  const stored = localStorage.getItem("taskRegistryData");
  if (stored) {
    try {
      applyImportedPayload(JSON.parse(stored));
    } catch {
      // ignore broken local storage payload
    }
  }

  updatePreview();
});
</script>

<style scoped>
.page-shell {
  min-height: 100vh;
  background:
    radial-gradient(circle at top left, rgba(22, 163, 74, 0.12), transparent 28%),
    linear-gradient(180deg, #f8fafc 0%, #eef2f7 100%);
  padding: 24px 16px 48px;
  color: #0f172a;
}

.page-inner {
  max-width: 1120px;
  margin: 0 auto;
  display: grid;
  gap: 20px;
}

.hero {
  background: linear-gradient(135deg, #0f766e 0%, #166534 45%, #1d4ed8 100%);
  color: #fff;
  border-radius: 22px;
  padding: 24px;
  display: flex;
  justify-content: space-between;
  gap: 16px;
  align-items: flex-start;
  box-shadow: 0 20px 50px rgba(15, 23, 42, 0.18);
}

.hero-badge,
.card-kicker {
  display: inline-flex;
  align-items: center;
  padding: 6px 10px;
  border-radius: 999px;
  font-size: 12px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.08em;
}

.hero-badge {
  background: rgba(255, 255, 255, 0.16);
  margin-bottom: 12px;
}

.hero-title {
  margin: 0;
  font-size: 32px;
  line-height: 1.05;
}

.hero-subtitle {
  margin: 10px 0 0;
  max-width: 640px;
  font-size: 15px;
  line-height: 1.6;
  color: rgba(255, 255, 255, 0.88);
}

.meta-grid {
  display: grid;
  grid-template-columns: 1fr 1.3fr;
  gap: 20px;
}

.meta-grid-single {
  grid-template-columns: 1fr;
}

.card {
  background: rgba(255, 255, 255, 0.96);
  border: 1px solid rgba(148, 163, 184, 0.22);
  border-radius: 18px;
  padding: 20px;
  box-shadow: 0 10px 30px rgba(15, 23, 42, 0.06);
}

.card-kicker {
  background: #e2e8f0;
  color: #334155;
  margin-bottom: 14px;
}

.section-head {
  display: flex;
  justify-content: space-between;
  gap: 16px;
  align-items: flex-start;
  margin-bottom: 18px;
}

.section-title {
  margin: 4px 0 0;
  font-size: 24px;
  line-height: 1.15;
}

.section-caption {
  margin: 8px 0 0;
  color: #64748b;
  font-size: 14px;
  line-height: 1.5;
}

.meta-list {
  display: grid;
  gap: 12px;
}

.meta-list div {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.meta-list span {
  color: #64748b;
  font-size: 13px;
}

.summary-actions,
.form-actions,
.row-actions {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.form-actions {
  margin-top: 24px;
}

.summary-box {
  margin: 16px 0 0;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 14px;
  padding: 14px;
  white-space: pre-wrap;
  font-family:
    "IBM Plex Mono", "SFMono-Regular", Consolas, "Liberation Mono", monospace;
  font-size: 13px;
  line-height: 1.6;
  color: #0f172a;
}

.group-block + .group-block {
  margin-top: 22px;
}

.group-title {
  margin: 0 0 14px;
  font-size: 13px;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  color: #64748b;
}

.form-grid,
.filter-grid,
.fact-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 14px;
}

.fact-grid {
  grid-template-columns: repeat(5, minmax(0, 1fr));
  margin-top: 14px;
}

.field {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.field-span-2 {
  grid-column: span 2;
}

.field-label {
  font-size: 13px;
  font-weight: 600;
  color: #334155;
}

.field-input {
  width: 100%;
  border: 1px solid #cbd5e1;
  border-radius: 12px;
  padding: 12px 14px;
  font: inherit;
  background: #fff;
  color: #0f172a;
  box-sizing: border-box;
}

.field-input:focus {
  outline: none;
  border-color: #16a34a;
  box-shadow: 0 0 0 4px rgba(34, 197, 94, 0.12);
}

.btn {
  appearance: none;
  border: none;
  border-radius: 12px;
  padding: 11px 14px;
  font: inherit;
  font-weight: 600;
  cursor: pointer;
  transition:
    transform 0.15s ease,
    opacity 0.15s ease,
    box-shadow 0.15s ease;
}

.btn:hover {
  transform: translateY(-1px);
}

.btn-primary {
  background: #166534;
  color: #fff;
  box-shadow: 0 10px 24px rgba(22, 101, 52, 0.25);
}

.btn-secondary {
  background: #0f172a;
  color: #fff;
}

.btn-light {
  background: #fff;
  color: #0f172a;
  border: 1px solid #dbe2ea;
}

.btn-danger {
  background: #fff1f2;
  color: #be123c;
  border: 1px solid #fecdd3;
}

.template-grid {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 12px;
}

.template-card {
  border: 1px solid #dbeafe;
  background: #f8fbff;
  border-radius: 16px;
  padding: 14px;
  text-align: left;
  display: flex;
  flex-direction: column;
  gap: 8px;
  cursor: pointer;
  color: #0f172a;
}

.template-card.active {
  border-color: #16a34a;
  background: #f0fdf4;
  box-shadow: inset 0 0 0 1px rgba(34, 197, 94, 0.18);
}

.template-type {
  color: #64748b;
  font-size: 11px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.08em;
}

.preview-panel {
  margin-top: 16px;
  display: grid;
  grid-template-columns: repeat(5, minmax(0, 1fr));
  gap: 12px;
}

.preview-item {
  border-radius: 14px;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  padding: 14px;
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.preview-item span {
  font-size: 12px;
  color: #64748b;
}

.preview-item strong {
  font-size: 16px;
}

.quadrant-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 16px;
}

.quadrant-card {
  border: 2px solid transparent;
  border-radius: 18px;
  padding: 18px;
}

.quadrant-head {
  display: flex;
  justify-content: space-between;
  gap: 10px;
  align-items: flex-start;
}

.quadrant-head h3 {
  margin: 0;
  font-size: 20px;
}

.quadrant-head p {
  margin: 8px 0 0;
  font-size: 14px;
  line-height: 1.5;
}

.quadrant-count {
  min-width: 36px;
  height: 36px;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.9);
  display: inline-flex;
  align-items: center;
  justify-content: center;
  font-weight: 700;
}

.dot-wrap {
  margin-top: 16px;
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.task-dot {
  width: 132px;
  min-height: 68px;
  padding: 10px 12px;
  border-radius: 18px;
  border: none;
  font-weight: 700;
  font-size: 12px;
  line-height: 1.35;
  cursor: pointer;
  box-shadow: 0 10px 20px rgba(15, 23, 42, 0.14);
  text-align: center;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  word-break: break-word;
}

.queue-list,
.list-stack {
  display: grid;
  gap: 14px;
}

.queue-item,
.list-card {
  border: 1px solid #e2e8f0;
  background: #fff;
  border-radius: 16px;
  padding: 16px;
  display: flex;
  justify-content: space-between;
  gap: 14px;
  align-items: flex-start;
}

.queue-main,
.list-main {
  display: flex;
  gap: 14px;
  width: 100%;
}

.queue-position {
  min-width: 56px;
  height: 56px;
  border-radius: 16px;
  background: #0f172a;
  color: #fff;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 18px;
  font-weight: 700;
}

.queue-position-input {
  width: 56px;
  height: 56px;
  border: none;
  background: transparent;
  color: #fff;
  text-align: center;
  font: inherit;
  font-weight: 700;
}

.queue-position-input:focus {
  outline: none;
}

.queue-body {
  flex: 1;
}

.queue-title-row,
.list-title-row {
  display: flex;
  justify-content: space-between;
  gap: 12px;
  align-items: center;
}

.chip-row {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-top: 10px;
}

.chip,
.value-chip {
  display: inline-flex;
  align-items: center;
  border-radius: 999px;
  padding: 6px 10px;
  font-size: 12px;
  font-weight: 600;
}

.chip {
  background: #f1f5f9;
  color: #334155;
}

.value-chip {
  background: #dcfce7;
  color: #166534;
}

.queue-text {
  margin: 12px 0 0;
  color: #475569;
  line-height: 1.6;
}

.empty-state {
  margin: 0;
  color: #64748b;
  line-height: 1.6;
}

@media (max-width: 1080px) {
  .meta-grid,
  .quadrant-grid,
  .template-grid,
  .preview-panel,
  .fact-grid {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
}

@media (max-width: 760px) {
  .hero,
  .section-head,
  .queue-item,
  .list-card,
  .queue-main,
  .list-main {
    flex-direction: column;
  }

  .form-grid,
  .filter-grid,
  .meta-grid,
  .quadrant-grid,
  .template-grid,
  .preview-panel,
  .fact-grid {
    grid-template-columns: 1fr;
  }

  .field-span-2 {
    grid-column: auto;
  }
}
</style>
