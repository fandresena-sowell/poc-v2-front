<template>
  <div>
    <q-item>
      <q-item-section avatar>
        <q-checkbox
          true-value="done"
          false-value="pending"
          indeterminate-value="canceled"
          v-model="checked"
          keep-color
          :color="isLoading ? 'warning' : 'teal'"
          :disable="isLoading"
        />
      </q-item-section>
      <q-item-section>
        <q-item-label :class="todo.state === 'canceled' ? 'text-strike' : ''">
          {{ todo.label }}
        </q-item-label>
      </q-item-section>
      <q-item-section avatar>
        <div class="row">
          <q-btn
            icon="delete"
            flat
            round
            @click="$emit('delete', { _id: todo._id, state: 'canceled' })"
            :disable="isLoading"
          ></q-btn>
        </div>
      </q-item-section>
    </q-item>
  </div>
</template>

<script lang="ts" setup>
import { computed, PropType, defineEmits, defineProps } from 'vue'
import { ITodoItem } from 'src/models/interfaces/ITodoItem'
const props = defineProps({
  todo: {
    type: Object as PropType<ITodoItem>,
    required: true,
  },
  isLoading: {
    type: Boolean,
    defaults: () => true,
  },
})

const emit = defineEmits(['click', 'delete'])

const checked = computed({
  get: () => props.todo.state,
  set: (value) => {
    emit('click', { _id: props.todo._id, state: value })
  },
})
</script>
