<script setup lang="ts">
import { ref, computed, watch } from 'vue'

// Todo type
type Todo = {
  id: number
  text: string
  completed: boolean
  editing?: boolean
}

// Filter options
const FILTERS = {
  all: 'All',
  active: 'Active',
  completed: 'Completed',
} as const
type FilterKey = keyof typeof FILTERS

// Storage key
const STORAGE_KEY = 'todos-vue-v1'

// reactive state
const todos = ref<Todo[]>([])
const newTodo = ref('')
const editingText = ref('')
const filter = ref<FilterKey>('all')

// Load todos from localStorage
function loadTodos() {
  const saved = localStorage.getItem(STORAGE_KEY)
  if (saved) {
    try {
      todos.value = JSON.parse(saved)
    } catch {}
  }
}
loadTodos()

// Persist todos to localStorage on changes
watch(
  todos,
  () => {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(todos.value))
  },
  { deep: true }
)

// PUBLIC_INTERFACE
function addTodo() {
  if (!newTodo.value.trim()) return
  todos.value.unshift({
    id: Date.now(),
    text: newTodo.value.trim(),
    completed: false,
  })
  newTodo.value = ''
}

// PUBLIC_INTERFACE
function removeTodo(id: number) {
  todos.value = todos.value.filter((t) => t.id !== id)
}

// PUBLIC_INTERFACE
function toggleComplete(todo: Todo) {
  todo.completed = !todo.completed
}

// PUBLIC_INTERFACE
function editTodo(todo: Todo) {
  todos.value.forEach((t) => (t.editing = false))
  editingText.value = todo.text
  todo.editing = true
}

// PUBLIC_INTERFACE
function saveEdit(todo: Todo) {
  if (!editingText.value.trim()) {
    removeTodo(todo.id)
  } else {
    todo.text = editingText.value.trim()
    todo.editing = false
  }
}

// PUBLIC_INTERFACE
function cancelEdit(todo: Todo) {
  todo.editing = false
  editingText.value = todo.text
}

// PUBLIC_INTERFACE
function clearCompleted() {
  todos.value = todos.value.filter((t) => !t.completed)
}

// PUBLIC_INTERFACE
function setFilter(key: FilterKey) {
  filter.value = key
}

// PUBLIC_INTERFACE
function markAll(checked: boolean) {
  todos.value.forEach((t) => (t.completed = checked))
}

// PUBLIC_INTERFACE
const filteredTodos = computed(() => {
  if (filter.value === 'active') return todos.value.filter((t) => !t.completed)
  if (filter.value === 'completed') return todos.value.filter((t) => t.completed)
  return todos.value
})

const remaining = computed(() => todos.value.filter((t) => !t.completed).length)
const anyTodos = computed(() => todos.value.length > 0)
const anyCompleted = computed(() => todos.value.some((t) => t.completed))

// Handle Enter for edit/save
function handleEditKey(e: KeyboardEvent, todo: Todo) {
  if (e.key === 'Enter') saveEdit(todo)
  if (e.key === 'Escape') cancelEdit(todo)
}
</script>

<template>
  <div class="todo-app-container">
    <div class="todo-card">
      <h1 class="app-title">My Todos</h1>
      <form @submit.prevent="addTodo" class="input-wrap">
        <input
          v-model="newTodo"
          @keydown.enter="addTodo"
          class="todo-input"
          type="text"
          placeholder="What needs to be done?"
          autocomplete="off"
        />
        <button class="accent add-btn" type="submit" aria-label="Add">+</button>
      </form>

      <div v-if="anyTodos" class="tools-wrap">
        <div class="filters">
          <button
            v-for="(label, key) in FILTERS"
            :key="key"
            :class="{ active: filter === key }"
            @click="setFilter(key as FilterKey)"
          >
            {{ label }}
          </button>
        </div>
        <div class="bulk-tools">
          <button v-if="anyCompleted" class="secondary" @click="clearCompleted" aria-label="Clear completed">
            Clear Completed
          </button>
          <button class="primary" @click="markAll(true)">Mark All Complete</button>
          <button class="secondary" @click="markAll(false)">Unmark All</button>
        </div>
      </div>

      <ul v-if="filteredTodos.length" class="todo-list">
        <li
          v-for="todo in filteredTodos"
          :key="todo.id"
          :class="{ completed: todo.completed, editing: todo.editing }"
          class="todo-item"
        >
          <label v-if="!todo.editing" class="check-wrap">
            <input type="checkbox" v-model="todo.completed" @change="toggleComplete(todo)" />
            <span class="check"></span>
          </label>
          <div v-if="!todo.editing" class="todo-text" @dblclick="editTodo(todo)">
            {{ todo.text }}
          </div>
          <input
            v-if="todo.editing"
            v-model="editingText"
            @keydown="(e) => handleEditKey(e, todo)"
            @blur="saveEdit(todo)"
            class="edit-input"
            type="text"
            autofocus
          />
          <div class="actions" v-if="!todo.editing">
            <button class="edit" @click="editTodo(todo)" title="Edit"><span>Edit</span></button>
            <button class="delete" @click="removeTodo(todo.id)" title="Delete"><span>Delete</span></button>
          </div>
        </li>
      </ul>
      <div v-else class="empty-state">
        <p>No todos... Add your first one!</p>
      </div>
      <footer class="footer">
        <span class="left">
          {{ remaining }} {{ remaining === 1 ? 'item' : 'items' }} left
        </span>
      </footer>
    </div>
  </div>
</template>

<style scoped>
/* Theme colors */
:root {
  --primary: #42b883;
  --secondary: #35495e;
  --accent: #ffb300;
  --card-bg: var(--color-background, #fff);
}

.todo-app-container {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--color-background);
  transition: background 0.3s;
}

.todo-card {
  min-width: 340px;
  max-width: 400px;
  background: var(--card-bg);
  margin: 2rem auto 0;
  border-radius: 15px;
  box-shadow: 0 2px 16px 0 rgba(60,60,60,0.07);
  padding: 2.2rem 1.8rem 1.4rem;
  display: flex;
  flex-direction: column;
  gap: 1.4rem;
}

.app-title {
  text-align: center;
  color: var(--primary);
  font-family: 'Inter', sans-serif;
  font-weight: 600;
  letter-spacing: 1px;
}

.input-wrap {
  display: flex;
  gap: 0.67rem;
}

.todo-input {
  flex: 1;
  padding: 0.7rem 1.05rem;
  background: var(--color-background-mute, #f2f2f2);
  border: none;
  border-radius: 6px;
  color: var(--secondary);
  font-size: 1.08rem;
  outline: none;
  box-sizing: border-box;
  box-shadow: 0 0.5px #eee;
  transition: background 0.2s;
}

.todo-input:focus {
  background: #eaeaea;
}

.add-btn {
  min-width: 2.5rem;
  font-size: 1.5rem;
  font-weight: bold;
  color: #fff;
  background: var(--accent);
  border: none;
  border-radius: 6px;
  cursor: pointer;
  transition: background 0.17s;
}

.add-btn:hover {
  background: #ffa600;
}

.tools-wrap {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.5rem;
  margin-bottom: -0.7rem;
}

.filters {
  display: flex;
  gap: 0.4rem;
}

.filters button {
  background: none;
  border: 2px solid var(--primary);
  border-radius: 7px;
  padding: 0.29rem 0.80rem;
  color: var(--primary);
  cursor: pointer;
  font-weight: 500;
  font-size: 1rem;
  opacity: 0.7;
  transition: background 0.16s, color 0.18s, opacity 0.18s;
}

.filters .active,
.filters button.active {
  background: var(--primary);
  color: #fff;
  opacity: 1;
}

.bulk-tools {
  display: flex;
  gap: 0.25rem;
}

.bulk-tools button {
  background: none;
  border: none;
  border-radius: 6px;
  padding: 0.24rem 0.62rem;
  font-weight: 500;
  font-size: 0.96rem;
  cursor: pointer;
  transition: background 0.16s, color 0.16s;
}

.bulk-tools .primary {
  background: var(--primary);
  color: #fff;
}
.bulk-tools .secondary {
  background: var(--secondary);
  color: #fff;
  margin-left: 2px;
}

.todo-list {
  list-style: none;
  padding: 0;
  margin: 0;
}
.todo-item {
  background: var(--color-background-soft, #f8f8f8);
  padding: 0.76rem 0.64rem 0.76rem 0.5rem;
  margin-bottom: 0.3rem;
  border-radius: 8px;
  display: flex;
  align-items: center;
  gap: 0.9rem;
  transition: background 0.18s;
  box-shadow: 0 0.5px 0 #e6e6e6;
}

.todo-item:last-child {
  margin-bottom: 0;
}

.check-wrap {
  display: flex;
  align-items: center;
}
.check-wrap input[type="checkbox"] {
  accent-color: var(--primary);
  width: 1.04rem;
  height: 1.04rem;
  margin-right: 0.40rem;
  cursor: pointer;
}
.todo-text {
  flex: 1;
  font-size: 1.05rem;
  color: var(--secondary);
  cursor: pointer;
  transition: color 0.14s;
}
.todo-item.completed .todo-text {
  color: #aaa;
  text-decoration: line-through;
  opacity: 0.7;
}
.todo-item.editing .todo-text {
  display: none;
}
.edit-input {
  flex: 1;
  padding: 0.7rem 0.95rem;
  font-size: 1.05rem;
  border-radius: 5px;
  border: 1.2px solid var(--primary);
  outline: none;
  background: #fff;
}
.actions {
  display: flex;
  gap: 0.15rem;
}
.actions button {
  border: none;
  background: var(--card-bg);
  cursor: pointer;
  padding: 0.25rem 0.46rem;
  border-radius: 5px;
  margin-left: 1px;
  font-size: 0.97rem;
  color: var(--secondary);
  transition: background 0.16s, color 0.18s;
}
.actions button.edit {
  color: var(--primary);
}
.actions button.delete {
  color: var(--accent);
}

/* Slight button hover effects */
.actions button:hover,
.bulk-tools button:hover,
.filters button:hover {
  opacity: 0.9;
  filter: brightness(1.07);
}
.empty-state {
  text-align: center;
  color: #bbb;
  margin: 1.7rem 0 0.6rem 0;
  font-size: 1.08rem;
}

.footer {
  text-align: left;
  margin-top: 0.7rem;
  color: var(--secondary);
  font-size: 0.98rem;
  opacity: 0.82;
}

/* Support dark mode */
@media (prefers-color-scheme: dark) {
  .todo-card {
    background: #22293a;
    box-shadow: 0 2px 18px 0 rgba(24,28,38,0.24);
  }
  .todo-item {
    background: #273041;
    box-shadow: 0 0.5px 0 #1d212f;
  }
  .todo-text {
    color: #ddd;
  }
  .edit-input {
    background: #1b222b;
    color: #eee;
    border-color: var(--primary);
  }
  .actions button {
    background: #232634;
    color: #a3b0cb;
  }
  .filters button {
    background: #232634;
    border-color: var(--primary);
    color: var(--primary);
  }
  .filters button.active {
    background: var(--primary);
    color: #fff;
    opacity: 1;
  }
  .bulk-tools button.primary {
    background: var(--primary);
    color: #fff;
  }
  .bulk-tools button.secondary {
    background: var(--secondary);
    color: #fff;
  }
}
</style>
