<!-- src/components/child/CheckMotorDc.vue -->
<template>
  <div class="mb-8">
    <div class="flex justify-between items-center mb-2">
      <h2 class="text-xl font-semibold">Check Motor DC</h2>
      <button
        @click="isOpen = !isOpen"
        class="bg-gray-500 hover:bg-gray-600 text-white px-2 py-1 rounded"
      >
        {{ isOpen ? 'Close' : 'Open' }}
      </button>
    </div>
    <div v-if="isOpen">
      <div class="grid grid-cols-4 md:grid-cols-8 gap-4">
        <div v-for="(btn, index) in 48" :key="'dc' + index" class="relative">
          <button
            @click="sendCommand('JUAL', getRow(index), getColumn(index), index)"
            @mouseenter="hoveredIndex = index"
            @mouseleave="hoveredIndex = null"
            :disabled="isAnyMotorActive && !motorStatus[index].isActive"
            class="bg-blue-500 hover:bg-blue-600 text-white px-2 py-4 rounded w-full relative flex items-center justify-center"
            :class="{
              'opacity-50 cursor-not-allowed': isAnyMotorActive && !motorStatus[index].isActive,
            }"
          >
            <span
              v-if="motorStatus[index].isActive"
              class="absolute inset-0 flex items-center justify-center"
            >
              <svg
                class="animate-spin h-5 w-5 text-white"
                xmlns="http://www.w3.org/2000/svg"
                fill="none"
                viewBox="0 0 24 24"
              >
                <circle
                  class="opacity-25"
                  cx="12"
                  cy="12"
                  r="10"
                  stroke="currentColor"
                  stroke-width="4"
                ></circle>
                <path
                  class="opacity-75"
                  fill="currentColor"
                  d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"
                ></path>
              </svg>
            </span>
            <span :class="{ invisible: motorStatus[index].isActive }">
              {{
                motorStatus[index] && motorStatus[index].completed
                  ? 'OK'
                  : `${getRow(index)}:${getColumn(index)}`
              }}
            </span>
          </button>
          <div
            v-if="
              hoveredIndex === index &&
              motorStatus[index] &&
              (motorStatus[index].delay !== null || motorStatus[index].totalTime !== null)
            "
            class="absolute z-10 bg-gray-800 text-white text-sm p-2 rounded shadow-lg -top-10 left-0"
          >
            <span v-if="motorStatus[index].delay !== null"
              >Delay: {{ motorStatus[index].delay }} ms</span
            >
            <span v-if="motorStatus[index].totalTime !== null" class="ml-2"
              >Total: {{ motorStatus[index].totalTime }} ms</span
            >
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
    motorStatus: Array,
    multiplier: Number,
  },
  data() {
    return {
      isOpen: false,
      hoveredIndex: null,
    }
  },
  computed: {
    isAnyMotorActive() {
      return this.motorStatus.some((motor) => motor.isActive)
    },
  },
  methods: {
    getRow(index) {
      const row = Math.floor(index / 8) + 1 // 6 baris (R01 - R06)
      return 'R' + row.toString().padStart(2, '0')
    },
    getColumn(index) {
      const col = (index % 8) + 1 // 8 kolom (C01 - C08)
      return 'C' + col.toString().padStart(2, '0')
    },
    async sendCommand(command, row, column, index) {
      const topic = 'esp32/CtrlRelay'
      const message = `${this.deviceId} ${command} ${row}:${column}`

      if (this.client && this.client.connected) {
        for (let i = 0; i < this.multiplier; i++) {
          console.log(`Testing ${row}:${column} - Iteration ${i + 1}/${this.multiplier}`)
          this.$emit('start-motor-test', { index, commandSentTime: Date.now() })
          await new Promise((resolve, reject) => {
            this.client.publish(topic, message, { qos: 1 }, (err) => {
              if (err) {
                console.error('Publish error:', err)
                reject(err)
              } else {
                console.log('Command sent:', message)
                const checkCompletion = () => {
                  if (!this.motorStatus[index].isActive) {
                    resolve()
                  } else {
                    setTimeout(checkCompletion, 100)
                  }
                }
                checkCompletion()
              }
            })
          })
        }
      } else {
        console.error('MQTT client belum terkoneksi')
      }
    },
  },
}
</script>
