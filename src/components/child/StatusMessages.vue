<!-- src/components/StatusMessages.vue -->
<template>
  <div class="mt-6">
    <div class="flex justify-between items-center mb-2">
      <h2 class="text-xl font-semibold">Status Pesan</h2>
      <button
        @click="resetMessages"
        class="bg-red-500 hover:bg-red-600 text-white px-3.25 py-1 rounded">
        Reset Pesan
      </button>
    </div>
    <ul class="list-disc pl-5 max-h-64 overflow-y-auto">
      <li v-for="(msg, index) in messages" :key="index" class="text-gray-700">
        {{ msg }}
      </li>
    </ul>
  </div>
</template>

<script>
export default {
  name: 'StatusMessages',
  props: {
    messages: {
      type: Array,
      default: () => []
    }
  },
  methods: {
    resetMessages() {
      this.$emit('reset-messages');
    }
  }
}
</script>
