<script setup lang="ts">
import { computed, onMounted, reactive, ref, useTemplateRef, watch } from 'vue'

const REPEAT = {
  NONE: 0,
  DAILY: 1,
  WEEKLY: 2,
  BIWEEKLY: 3,
} as const

const PRIORITY = {
  LOW: 0,
  MEDIUM: 1,
  HIGH: 2,
  HIGHEST: 3,
} as const

interface Todo {
  // to track in the UI rendering process, and for update/delete operation
  id: number
  title: string
  // group: string
  priority: (typeof PRIORITY)[keyof typeof PRIORITY]
  repeat: (typeof REPEAT)[keyof typeof REPEAT]
  deleted: boolean
  completed: boolean
}

const todos = ref<Todo[]>([])

// we need a computed property to filter out data to be rendered in the UI, instead of using v-for and v-if, we only need to use v-for
// we also need to have unique id in the data, so vue can optimize the operation of changing the element
const activeTodos = computed(() => {
  return todos.value.filter((todo) => !todo.deleted)
})

const todoDialog = reactive({
  isOpen: false,
  open() {
    this.isOpen = true
  },
  close() {
    this.isOpen = false
  },
})

const defaultQuery: Todo = {
  id: Date.now(),
  title: '',
  repeat: REPEAT.NONE,
  priority: PRIORITY.LOW,
  deleted: false,
  completed: false,
} as const

const query = reactive({
  // shallow copy object using ...
  value: { ...defaultQuery } as Todo,
  reset() {
    this.value = {
      ...defaultQuery,
      id: Date.now(),
    }
  },
})

const debouncer = {
  ids: new Map<string, ReturnType<typeof setTimeout>>(),
  debounce(id: string, callback: () => void, timeMs: number) {
    // if it has id, clear the previous timeout
    if (this.ids.has(id)) {
      clearTimeout(this.ids.get(id))
      this.ids.delete(id)
    }

    // create new timeout and track the id
    const nid = setTimeout(callback, timeMs)
    this.ids.set(id, nid)
  },
}

const inputElementRef = useTemplateRef('todo-input')

function onSubmit() {
  createTodo()
  inputElementRef.value?.focus()
}

function createTodo() {
  todos.value.push(query.value)
  query.reset()
}

function deleteTodo(id: number) {
  const todo = todos.value.find((todo) => todo.id === id)

  // this is the correct way to check for an undefined value, using typeof keyword.
  // if you want to check for null value, you use it directly like this: (myvar === null)
  if (typeof todo !== 'undefined') {
    todo.deleted = true
  }

  debounceClearDeleted(2000)
}

function clearTodo() {
  todos.value = []
}

interface Storage<T extends object> {
  [k: string]: any
  save(value: T): void
  load(): T | null
}

// persistant storage
const LocalStorageStorage: Storage<Todo[]> = {
  KEY: 'TODOLIST',
  save(value) {
    localStorage.setItem(this.KEY, JSON.stringify(value))
  },
  load() {
    const item = localStorage.getItem(this.KEY)

    if (item === null) {
      return null
    }

    return JSON.parse(item) as Todo[]
  },
}

watch(
  todos,
  (value) => {
    debouncer.debounce(
      'LOCAL_SAVE',
      () => {
        console.info('Saving to Local storage')
        LocalStorageStorage.save(value)
      },
      1000,
    )

    // deep: 1, to see array changes, for example when the length of the array change because of .push and .pop
    // if you set deep: true, it will track any data change within the array, for example arr[1] = "abc", this will get tracked.
    // if you don't use any deep option, it'll only watch direct assignment to the .value, e.g list.value = newArray
  },
  { deep: true },
)

function debounceClearDeleted(timeMs: number = 5000) {
  debouncer.debounce('CLEAR_DELETED', clearDeleted, timeMs)
}

function clearDeleted() {
  todos.value = activeTodos.value
}

onMounted(() => {
  // hydrate todo
  todos.value = LocalStorageStorage.load() ?? []
  debounceClearDeleted(2000)
})
</script>

<template>
  <div>
    <h1>Todo List</h1>
    <button @click="clearTodo">Clear</button>
    <button @click="todoDialog.open">New</button>

    <div
      v-for="todo in activeTodos"
      style="display: flex; gap: 12px; padding-inline: 12px; border: 2px solid blue"
    >
      <p style="flex-grow: 1">{{ todo.title }}</p>
      <p>{{ todo.priority }}</p>
      <p>{{ todo.repeat }}</p>
      <button @click="deleteTodo(todo.id)">Delete</button>
    </div>

    <div v-if="todoDialog.isOpen">
      <p>Create new todo</p>
      <button @click="todoDialog.close">Close</button>
      <form action="" @submit.prevent="onSubmit">
        <input
          v-model="query.value.title"
          autofocus
          type="text"
          name=""
          ref="todo-input"
          placeholder="Random text from a list"
        />
        <input type="submit" value="CREATE" />
      </form>
    </div>
  </div>
</template>
