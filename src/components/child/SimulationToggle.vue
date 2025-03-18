<!-- src/components/child/SimulationToggle.vue -->
<template>
  <div class="flex items-center gap-2">
    <button
      @click="toggleSimulation"
      :class="isSimulationActive ? 'bg-red-500 hover:bg-red-600' : 'bg-green-500 hover:bg-green-600'"
      class="text-white px-4 py-2 rounded">
      {{ isSimulationActive ? 'Stop Simulation' : 'Start Simulation' }}
    </button>
  </div>
</template>

<script>
export default {
  name: 'SimulationToggle',
  props: {
    isSimulationActive: Boolean
  },
  methods: {
    toggleSimulation() {
      this.$emit('toggle-simulation', !this.isSimulationActive);
    }
  }
}
</script>
