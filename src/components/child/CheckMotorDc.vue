<!-- src/components/child/CheckMotorDc.vue -->
<template>
  <div class="mb-8">
    <div class="flex justify-between items-center mb-2">
      <h2 class="text-xl font-semibold">Check Motor Dc</h2>
      <button
        @click="isOpen = !isOpen"
        class="bg-gray-500 hover:bg-gray-600 text-white px-2 py-1 rounded">
        {{ isOpen ? 'Close' : 'Open' }}
      </button>
    </div>
    <div v-if="isOpen">
      <!-- Segmen A (16 tombol pertama) -->
      <div class="mb-4">
        <h3 class="text-lg font-medium mb-2">Motor Atas</h3>
        <div class="grid grid-cols-4 md:grid-cols-8 gap-4">
          <div v-for="(btn, index) in 16" :key="'a' + index" class="relative">
            <button
              @click="sendCommand('JUAL', getRow(index), getColumn(index), index)"
              @mouseenter="hoveredIndex = index"
              @mouseleave="hoveredIndex = null"
              class="bg-blue-500 hover:bg-blue-600 text-white px-2 py-4 rounded w-full">
              {{ motorStatus[index] && motorStatus[index].completed ? 'OK' : `${getRow(index)}:${getColumn(index)}` }}
            </button>
            <div
              v-if="hoveredIndex === index && motorStatus[index] && (motorStatus[index].delay !== null || motorStatus[index].totalTime !== null)"
              class="absolute z-10 bg-gray-800 text-white text-sm p-2 rounded shadow-lg -top-10 left-0">
              <span v-if="motorStatus[index].delay !== null">Delay: {{ motorStatus[index].delay }} ms</span>
              <span v-if="motorStatus[index].totalTime !== null" class="ml-2">Total: {{ motorStatus[index].totalTime }} ms</span>
            </div>
          </div>
        </div>
      </div>

      <!-- Segmen B (16 tombol berikutnya) -->
      <div>
        <h3 class="text-lg font-medium mb-2">Motor Bawah</h3>
        <div class="grid grid-cols-4 md:grid-cols-8 gap-4">
          <div v-for="(btn, index) in 16" :key="'b' + index" class="relative">
            <button
              @click="sendCommand('JUAL', getRow(index + 16), getColumn(index), index + 16)"
              @mouseenter="hoveredIndex = index + 16"
              @mouseleave="hoveredIndex = null"
              class="bg-green-500 hover:bg-green-600 text-white px-2 py-4 rounded w-full">
              {{ motorStatus[index + 16] && motorStatus[index + 16].completed ? 'OK' : `${getRow(index + 16)}:${getColumn(index)}` }}
            </button>
            <div
              v-if="hoveredIndex === index + 16 && motorStatus[index + 16] && (motorStatus[index + 16].delay !== null || motorStatus[index + 16].totalTime !== null)"
              class="absolute z-10 bg-gray-800 text-white text-sm p-2 rounded shadow-lg -top-10 left-0">
              <span v-if="motorStatus[index + 16].delay !== null">Delay: {{ motorStatus[index + 16].delay }} ms</span>
              <span v-if="motorStatus[index + 16].totalTime !== null" class="ml-2">Total: {{ motorStatus[index + 16].totalTime }} ms</span>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'CheckMotorDc',
  props: {
    client: Object,
    deviceId: String,
    motorStatus: Array
  },
  data() {
    return {
      isOpen: false,
      hoveredIndex: null
    }
  },
  methods: {
    getRow(index) {
      const row = Math.floor(index / 8) + 1;
      return 'R' + row.toString().padStart(2, '0');
    },
    getColumn(index) {
      const col = (index % 8) + 1;
      return 'C' + col.toString().padStart(2, '0');
    },
    sendCommand(command, row, column, index) {
      const topic = 'esp32/CtrlRelay';
      const message = `${this.deviceId} ${command} ${row}:${column}`;
      if (this.client && this.client.connected) {
        this.$emit('start-motor-test', { index, commandSentTime: Date.now() });
        this.client.publish(topic, message, { qos: 1 }, (err) => {
          if (err) {
            console.error("Publish error:", err);
          } else {
            console.log("Command sent:", message);
          }
        });
      } else {
        console.error("MQTT client belum terkoneksi");
      }
    }
  }
}
</script>
