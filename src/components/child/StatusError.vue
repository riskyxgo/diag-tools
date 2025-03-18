<!-- src/components/child/StatusError.vue -->
<template>
  <div class="mt-6">
    <div class="flex justify-between items-center mb-2">
      <h2 class="text-xl font-semibold">Status Error</h2>
      <button
        @click="resetErrors"
        class="bg-red-500 hover:bg-red-600 text-white px-4 py-1 rounded">
        Reset Error
      </button>
    </div>
    <div class="max-h-64 overflow-y-auto">
      <ul class="list-disc pl-5">
        <li v-for="(error, index) in errors" :key="index" :class="error.type === 'error' ? 'text-red-600' : 'text-yellow-600'">
          {{ error.message }} ({{ error.timestamp }})
        </li>
      </ul>
    </div>
  </div>
</template>

<script>
export default {
  name: 'StatusError',
  props: {
    errors: {
      type: Array,
      default: () => []
    }
  },
  methods: {
    resetErrors() {
      this.$emit('reset-errors');
    }
  }
}
</script>
