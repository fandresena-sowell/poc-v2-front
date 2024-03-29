<template>
  <q-page padding class="row">
    <template v-if="todos.length === 0">
      <div class="col column q-px-md">
        <div class="q-mb-lg">
          <div class="text-h2 text-primary">Hey</div>
          <div class="text-h2">you are</div>
          <div class="text-h2">free today</div>
        </div>
        <p class="text-caption text-grey q-mb-xl">Time to get active</p>
        <q-btn
          v-if="user._id"
          outline
          rounded
          color="primary"
          size="lg"
          label="Add a new task"
          @click="openDialog"
        />
        <q-btn
          v-else
          outline
          rounded
          color="primary"
          size="lg"
          label="Login to start"
          disable
        />
      </div>
    </template>
    <template v-else>
      <div class="col column q-px-md">
        <div class="q-mb-lg">
          <div class="text-h1 text-primary">{{ todos.length }}</div>
          <div class="text-h2">tasks</div>
          <div class="text-h2">for today</div>
        </div>
        <p class="text-caption text-grey q-mb-lg">
          {{ doneTodos.length }} tasks done
        </p>
        <q-btn
          v-if="user._id"
          outline
          rounded
          color="primary"
          size="lg"
          label="Add a new task"
          @click="openDialog"
        />
        <q-scroll-area class="q-mt-xl" style="min-height: 200px; width: auto">
          <template v-for="todo in todos" v-bind:key="todo.id">
            <poc-todo-item
              :todo="todo"
              :is-loading="
                tasks.findIndex((t) => t.payload._id === todo._id) !== -1
              "
              @click="handleClicked"
              @delete="handleDeleted"
            ></poc-todo-item>
          </template>
        </q-scroll-area>
      </div>
    </template>
  </q-page>
</template>

<script lang="ts" setup>
import { ref, computed, watch } from 'vue'
import { useQuasar } from 'quasar'
import { useAuthStore } from 'src/store/auth'
import { todoCollection, tasksCollection } from 'src/boot/pouchorm'
import lodash from 'lodash'

import { ITodoItem } from 'src/models/interfaces/ITodoItem'
import PocTodoItem from 'src/components/PocTodoItem'
import { ITask } from 'src/models/interfaces/ITask'

const authStore = useAuthStore()
const user = computed(() => {
  return authStore.getUser
})

const tasks = computed(() => {
  return authStore.getTasks
})

const refreshList = async (todoId?: string) => {
  // TODO: refresh list
  console.log('refreshing list', todoId)
  if (!todoId) {
    try {
      const storedTodos = await todoCollection.find({
        user: user.value._id,
      })
      todos.value = storedTodos.map((t) => {
        return {
          _id: t._id,
          label: t.label,
          state: t.state,
        }
      })
      console.log('done refreshing list')
    } catch (error) {
      console.error('err loading todos', error)
    }
  } else {
    const storedTodo = await todoCollection.findById(todoId)
    const index = todos.value.findIndex((t) => t._id === todoId)
    if (index !== -1) {
      todos.value[index] = {
        _id: storedTodo._id,
        label: storedTodo.label,
        state: storedTodo.state,
      }
    }
  }
}

watch(user, async (value) => {
  if (!value._id) {
    todos.value = []
    return
  }
  await refreshList()
})

watch(tasks, async (value, oldValue) => {
  if (value.length < oldValue.length) {
    const differences = lodash.differenceBy(oldValue, value, (elem) => elem._id)
    if (differences.length === 1) {
      const task = differences[0]
      if (task.type === 'edit') {
        await refreshList(task.payload._id)
      } else {
        await refreshList()
      }
    } else await refreshList()
  }
})

const todos = ref<ITodoItem[]>([])
const doneTodos = computed(() => todos.value.filter((t) => t.state === 'done'))

const $q = useQuasar()

const handleClicked = async ({ _id, state }: ITodoItem) => {
  if (!_id) return

  console.log('handleClicked', { _id, state })
  const editTask: ITask = {
    type: 'edit',
    state: 'pending',
    payload: {
      _id,
      state,
    },
  }
  authStore.pushTask(editTask)
  await tasksCollection.upsert(editTask)
}

const handleDeleted = async ({ _id }: ITodoItem) => {
  // TODO: implement
  console.log('handleClicked', { _id })
  const deleteTask: ITask = {
    type: 'delete',
    state: 'pending',
    payload: {
      _id,
    },
  }
  authStore.pushTask(deleteTask)
  await tasksCollection.upsert(deleteTask)
}

const openDialog = () => {
  $q.dialog({
    title: 'Add a task',
    message: 'What do you want to be done today?',
    prompt: {
      model: '',
      type: 'text', // optional
    },
    cancel: true,
    persistent: true,
  })
    .onOk((label: string) => {
      const createTask: ITask = {
        type: 'create',
        state: 'pending',
        payload: {
          label,
          state: 'pending',
          user: authStore.getUser._id,
        },
      }
      authStore.pushTask(createTask)
      void tasksCollection.upsert(createTask)
    })
    .onCancel(() => {
      // console.log('>>>> Cancel')
    })
    .onDismiss(() => {
      // console.log('I am triggered on both OK and Cancel')
    })
}
</script>

<style lang="sass" scope>
.text-caption
  font-size: 1.5rem
  font-weight: 400
  line-height: 1.25rem
  letter-spacing: 0.03333em
</style>
