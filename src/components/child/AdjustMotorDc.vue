<!-- src/components/child/AdjustMotorDc.vue -->
<template>
  <div class="mb-8">
    <div class="flex justify-between items-center mb-2">
      <h2 class="text-xl font-semibold">Adjust Motor DC</h2>
      <button
        @click="isOpen = !isOpen"
        class="bg-gray-500 hover:bg-gray-600 text-white px-2 py-1 rounded"
      >
        {{ isOpen ? 'Close' : 'Open' }}
      </button>
    </div>
    <div v-if="isOpen" class="grid grid-cols-4 md:grid-cols-8 gap-4">
      <button
        v-for="(motor, index) in totalMotors"
        :key="'adj' + index"
        @click="sendAdjustCommand(getRow(index), getColumn(index))"
        class="bg-purple-500 hover:bg-purple-600 text-white px-2 py-4 rounded"
      >
        {{ getRowDisplay(index) }}:{{ getColumn(index) }}
      </button>
    </div>
  </div>
</template>

<script>
export default {
  name: 'AdjustMotorDc',
  props: {
    client: Object,
    deviceId: String,
    rowConfig: Array,
  },
  data() {
    return {
      isOpen: false,
    }
  },
  computed: {
    totalMotors() {
      return this.rowConfig.reduce((sum, row) => sum + row.columns, 0)
    },
  },
  methods: {
    getRow(index) {
      let totalCols = 0
      for (let i = 0; i < this.rowConfig.length; i++) {
        if (index < totalCols + this.rowConfig[i].columns) {
          return this.rowConfig[i].address
        }
        totalCols += this.rowConfig[i].columns
      }
      return 'R01'
    },
    getRowDisplay(index) {
      let totalCols = 0
      for (let i = 0; i < this.rowConfig.length; i++) {
        if (index < totalCols + this.rowConfig[i].columns) {
          return this.rowConfig[i].display || this.rowConfig[i].address
        }
        totalCols += this.rowConfig[i].columns
      }
      return 'R01'
    },
    getColumn(index) {
      let totalCols = 0
      for (let i = 0; i < this.rowConfig.length; i++) {
        if (index < totalCols + this.rowConfig[i].columns) {
          const col = (index - totalCols) + 1
          return `C${col.toString().padStart(2, '0')}`
        }
        totalCols += this.rowConfig[i].columns
      }
      return 'C01'
    },
    sendAdjustCommand(row, column) {
      const topic = 'esp32/CtrlRelay'
      const message = `${this.deviceId} ADJUST ${row}:${column}`
      if (this.client && this.client.connected) {
        this.client.publish(topic, message, { qos: 1 }, (err) => {
          if (err) {
            console.error('Adjust command error:', err)
          } else {
            console.log('Adjust command sent:', message)
          }
        })
      } else {
        console.error('MQTT client belum terkoneksi')
      }
    },
  },
}
</script>
