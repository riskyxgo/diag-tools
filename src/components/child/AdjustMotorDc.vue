<!-- src/components/child/AdjustMotorDC.vue -->
<template>
  <div class="mb-8">
    <div class="flex justify-between items-center mb-2">
      <h2 class="text-xl font-semibold">Adjust Motor DC</h2>
      <button
        @click="isOpen = !isOpen"
        class="bg-gray-500 hover:bg-gray-600 text-white px-2 py-1 rounded">
        {{ isOpen ? 'Close' : 'Open' }}
      </button>
    </div>
    <div v-if="isOpen" class="grid grid-cols-4 md:grid-cols-8 gap-4">
      <button
        v-for="(btn, index) in 16"
        :key="'adj' + index"
        @click="sendAdjustCommand(getRow(index), getColumn(index))"
        class="bg-purple-500 hover:bg-purple-600 text-white px-2 py-4 rounded">
        {{ getRow(index) }}:{{ getColumn(index) }}
      </button>
    </div>
  </div>
</template>

<script>
export default {
  name: 'AdjustMotorDC',
  props: {
    client: Object,
    deviceId: String
  },
  data() {
    return {
      isOpen: false
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
    sendAdjustCommand(row, column) {
      const topic = 'esp32/CtrlRelay';
      const message = `${this.deviceId} ADJUST ${row}:${column}`;
      if (this.client && this.client.connected) {
        this.client.publish(topic, message, { qos: 1 }, (err) => {
          if (err) {
            console.error("Adjust command error:", err);
          } else {
            console.log("Adjust command sent:", message);
          }
        });
      } else {
        console.error("MQTT client belum terkoneksi");
      }
    }
  }
}
</script>
