<template>
  <main class="app-shell">
    <header class="app-header"><h1>Task Manager</h1><p class="save-status" role="status">{{ saveMessage }}</p></header>
    <section class="stats-grid" aria-label="Task statistics">
      <article v-for="(count, label) in taskStats" :key="label" class="stat-card"><span>{{ label }}</span><strong>{{ count }}</strong></article>
    </section>
    <section class="workspace">
      <TaskForm :editing-task="editingTask" @save="saveTask" @cancel="editingTask = null">
        <template #heading><h2>{{ editingTask ? 'Edit task' : 'New task' }}</h2></template>
      </TaskForm>
      <section class="task-panel" aria-label="Tasks">
        <section class="filters" aria-label="Task filters">
          <label>Search<input v-model="searchQuery" type="search" placeholder="Search by title" /></label>
          <label>Status<select v-model="statusFilter"><option value="all">All</option><option value="active">Active</option><option value="completed">Completed</option></select></label>
          <label>Priority<select v-model="priorityFilter"><option value="all">All priorities</option><option v-for="priority in priorities" :key="priority">{{ priority }}</option></select></label>
        </section>
        <div class="task-list">
          <TaskItem v-for="task in filteredTasks" :key="task.id" :task="task" @edit="startEdit" @delete="deleteTask" @complete="toggleComplete" @priority="changePriority" />
          <p v-if="!filteredTasks.length" class="empty-state">{{ tasks.length ? 'No matching tasks.' : 'No tasks yet.' }}</p>
        </div>
      </section>
    </section>
  </main>
</template>

<script>
import { defineComponent, h, resolveComponent } from 'vue'
import { Pencil, Trash2, Check, RotateCcw, Plus, Save } from '@lucide/vue'

const storageKey = 'sis2-vue-tasks'
const priorities = ['High', 'Medium', 'Low']
const blankForm = () => ({ title: '', description: '', priority: 'Medium' })

export const BaseButton = defineComponent({
  name: 'BaseButton',
  props: { type: { default: 'button' }, variant: { default: 'primary' } },
  render() {
    return h('button', { type: this.type, class: ['base-button', this.variant] }, this.$slots.default?.())
  },
})

const TaskForm = defineComponent({
  name: 'TaskForm',
  props: { editingTask: { type: Object, default: null } },
  emits: ['save', 'cancel'],
  data: () => ({ form: blankForm(), error: '' }),
  watch: {
    editingTask: {
      immediate: true,
      handler(task) {
        this.form = task ? { title: task.title, description: task.description, priority: task.priority } : blankForm()
        this.error = ''
      },
    },
  },
  methods: {
    submit(event) {
      event.preventDefault()
      const title = this.form.title.trim()
      if (!title) {
        this.error = 'Enter a task title.'
        return
      }
      this.$emit('save', { ...this.form, title, description: this.form.description.trim() })
      this.form = blankForm()
      this.error = ''
    },
  },
  render() {
    const Button = resolveComponent('BaseButton')
    const field = (label, key, tag, extra = {}) => h('label', [label, h(tag, {
      ...extra, value: this.form[key], onInput: (event) => { this.form[key] = event.target.value },
    })])
    return h('form', { class: 'task-form', onSubmit: this.submit }, [
      this.$slots.heading?.(),
      field('Title', 'title', 'input', { required: true, maxlength: 120, placeholder: 'Task title' }),
      field('Description', 'description', 'textarea', { rows: 4, maxlength: 2000, placeholder: 'Task description' }),
      h('label', ['Priority', h('select', { value: this.form.priority, onChange: (event) => { this.form.priority = event.target.value } }, priorities.map((value) => h('option', { value }, value)))]),
      this.error ? h('p', { role: 'alert' }, this.error) : null,
      h('div', { class: 'form-actions' }, [
        h(Button, { type: 'submit' }, () => [h(this.editingTask ? Save : Plus, { size: 16, 'aria-hidden': true }), this.editingTask ? 'Save changes' : 'Add task']),
        this.editingTask ? h(Button, { variant: 'ghost', onClick: () => this.$emit('cancel') }, () => 'Cancel') : null,
      ]),
    ])
  },
})

const TaskItem = defineComponent({
  name: 'TaskItem',
  props: { task: { type: Object, required: true } },
  emits: ['edit', 'delete', 'complete', 'priority'],
  computed: {
    dateLabel() { return new Date(this.task.createdAt).toLocaleDateString() },
  },
  render() {
    const task = this.task
    const Button = resolveComponent('BaseButton')
    return h('article', { class: ['task-item', { completed: task.completed }] }, [
      h('div', { class: 'task-main' }, [
        h('div', [h('p', { class: 'task-meta' }, `${this.dateLabel} / ${task.completed ? 'Completed' : 'Active'}`), h('h3', task.title), h('p', task.description)]),
        h('span', { class: ['priority-badge', `priority-${task.priority.toLowerCase()}`] }, task.priority),
      ]),
      h('div', { class: 'task-controls' }, [
        h('label', ['Priority', h('select', { value: task.priority, onChange: (event) => this.$emit('priority', task.id, event.target.value) }, priorities.map((value) => h('option', { value }, value)))]),
        h(Button, { variant: 'success', onClick: () => this.$emit('complete', task.id) }, () => [h(task.completed ? RotateCcw : Check, { size: 16, 'aria-hidden': true }), task.completed ? 'Activate' : 'Complete']),
        h(Button, { variant: 'secondary', title: 'Edit task', 'aria-label': `Edit ${task.title}`, onClick: () => this.$emit('edit', task) }, () => h(Pencil, { size: 17, 'aria-hidden': true })),
        h(Button, { variant: 'danger', title: 'Delete task', 'aria-label': `Delete ${task.title}`, onClick: () => this.$emit('delete', task.id) }, () => h(Trash2, { size: 17, 'aria-hidden': true })),
      ]),
    ])
  },
})

export default {
  name: 'App',
  components: { TaskForm, TaskItem },
  data() {
    return { tasks: [], editingTask: null, searchQuery: '', statusFilter: 'all', priorityFilter: 'all', priorities, saveMessage: 'Saved locally' }
  },
  computed: {
    filteredTasks() {
      const query = this.searchQuery.trim().toLowerCase()
      return this.tasks.filter((task) => task.title.toLowerCase().includes(query)
        && (this.statusFilter === 'all' || task.completed === (this.statusFilter === 'completed'))
        && (this.priorityFilter === 'all' || task.priority === this.priorityFilter))
    },
    taskStats() {
      const completed = this.tasks.filter((task) => task.completed).length
      return { Total: this.tasks.length, Active: this.tasks.length - completed, Completed: completed }
    },
  },
  watch: {
    tasks: {
      deep: true,
      handler(tasks) {
        try {
          localStorage.setItem(storageKey, JSON.stringify(tasks))
          this.saveMessage = 'Saved locally'
        } catch {
          this.saveMessage = 'Unable to save in this browser'
        }
      },
    },
  },
  created() {
    try {
      const tasks = JSON.parse(localStorage.getItem(storageKey) || '[]')
      if (!Array.isArray(tasks) || !tasks.every((task) => task && ['string', 'number'].includes(typeof task.id)
        && typeof task.title === 'string' && typeof task.description === 'string'
        && typeof task.completed === 'boolean' && priorities.includes(task.priority)
        && typeof task.createdAt === 'string' && Number.isFinite(Date.parse(task.createdAt)))
        || new Set(tasks.map((task) => task.id)).size !== tasks.length) throw new Error('Invalid tasks')
      this.tasks = tasks
    } catch {
      this.saveMessage = 'Saved tasks could not be loaded'
    }
  },
  mounted() { document.title = 'Task Manager' },
  methods: {
    saveTask(payload) {
      if (this.editingTask) {
        this.tasks = this.tasks.map((task) => task.id === this.editingTask.id ? { ...task, ...payload } : task)
        this.editingTask = null
      } else {
        this.tasks.unshift({ ...payload, id: crypto.randomUUID(), createdAt: new Date().toISOString(), completed: false })
      }
    },
    startEdit(task) { this.editingTask = { ...task } },
    deleteTask(id) {
      this.tasks = this.tasks.filter((task) => task.id !== id)
      if (this.editingTask?.id === id) this.editingTask = null
    },
    toggleComplete(id) {
      this.tasks = this.tasks.map((task) => task.id === id ? { ...task, completed: !task.completed } : task)
    },
    changePriority(id, priority) {
      this.tasks = this.tasks.map((task) => task.id === id ? { ...task, priority } : task)
    },
  },
}
</script>

<style>
* { box-sizing: border-box; }
body { margin: 0; background: #f5f7f8; color: #23282d; font-family: Arial, sans-serif; letter-spacing: 0; }
button, input, select, textarea { font: inherit; }
h1, h2, h3, p { margin-top: 0; }
h1 { margin-bottom: 0; font-size: 28px; }
h2 { font-size: 20px; margin-bottom: 20px; }
h3 { font-size: 18px; margin-bottom: 8px; }
.app-shell { width: min(1180px, calc(100% - 32px)); margin: auto; padding: 32px 0; }
.app-header { display: flex; flex-wrap: wrap; gap: 16px; align-items: center; justify-content: space-between; margin-bottom: 24px; }
.save-status { color: #58636d; font-size: 14px; margin: 0; }
.stats-grid { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 12px; margin-bottom: 32px; }
.stat-card { background: white; padding: 18px; border: 1px solid #dce1e5; border-radius: 6px; }
.stat-card span { display: block; color: #58636d; font-size: 14px; }
.stat-card strong { display: block; font-size: 28px; margin-top: 8px; }
.workspace { display: grid; grid-template-columns: 320px minmax(0, 1fr); gap: 32px; align-items: start; }
.task-form { position: sticky; top: 16px; min-width: 0; }
label { display: grid; gap: 8px; font-size: 14px; font-weight: 600; color: #46505a; min-width: 0; }
.task-form label + label { margin-top: 16px; }
input, select, textarea { width: 100%; min-width: 0; border: 1px solid #bdc7ce; border-radius: 6px; background: white; color: #23282d; padding: 11px 12px; }
textarea { resize: vertical; min-height: 112px; }
input:focus, select:focus, textarea:focus { outline: 2px solid #1f6feb; outline-offset: 2px; }
button:focus-visible { outline: 2px solid #1f6feb; outline-offset: 3px; }
.form-actions, .task-controls { display: flex; flex-wrap: wrap; gap: 10px; }
.form-actions { margin-top: 18px; }
.base-button { display: inline-flex; align-items: center; justify-content: center; gap: 8px; min-height: 42px; min-width: 42px; border: 0; border-radius: 6px; padding: 0 12px; color: white; font-weight: 600; cursor: pointer; }
.base-button:hover { filter: brightness(0.92); }
.primary { background: #1f6feb; }
.secondary { background: #4d5963; }
.success { background: #168354; }
.danger { background: #bd2531; }
.ghost { background: #e4e9ed; color: #23282d; }
.task-panel { min-width: 0; }
.filters { display: grid; grid-template-columns: minmax(0, 1fr) 125px 140px; gap: 12px; margin-bottom: 20px; }
.task-list { display: grid; gap: 12px; }
.task-item { padding: 18px; background: white; border: 1px solid #dce1e5; border-radius: 6px; }
.task-main { display: flex; gap: 16px; justify-content: space-between; margin-bottom: 16px; }
.task-main > div { min-width: 0; overflow-wrap: anywhere; }
.task-main p { white-space: pre-wrap; }
.task-meta { color: #58636d; font-size: 13px; margin-bottom: 8px; }
.completed h3 { text-decoration: line-through; color: #64717a; }
.priority-badge { align-self: start; flex-shrink: 0; border-radius: 4px; padding: 6px 10px; font-size: 13px; font-weight: 600; }
.priority-high { background: #ffe5e9; color: #a91d2d; }
.priority-medium { background: #fff0ca; color: #805500; }
.priority-low { background: #dff7ea; color: #12603b; }
.task-controls { align-items: end; }
.task-controls label { width: 135px; }
.empty-state { border: 1px dashed #bdc7ce; border-radius: 6px; padding: 28px 16px; color: #58636d; text-align: center; }
[role='alert'] { color: #a91d2d; margin-top: 12px; }
@media (max-width: 900px) {
  .workspace { grid-template-columns: 1fr; }
  .task-form { position: static; }
}
@media (max-width: 540px) {
  .app-shell { padding-top: 20px; }
  .filters { grid-template-columns: repeat(2, minmax(0, 1fr)); }
  .filters label:first-child { grid-column: 1 / -1; }
  .stat-card { padding: 12px; }
  .stat-card span { font-size: 12px; }
  .task-main { flex-wrap: wrap; }
  .task-main > div { width: 100%; }
}
</style>
