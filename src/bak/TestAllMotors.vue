<!-- src/components/child/TestAllMotors.vue -->
<template>
  <div class="mb-8">
    <div class="flex justify-between items-center mb-2">
      <h2 class="text-xl font-semibold">Test All Motors</h2>
      <button
        @click="isOpen = !isOpen"
        class="bg-gray-500 hover:bg-gray-600 text-white px-2 py-1 rounded">
        {{ isOpen ? 'Close' : 'Open' }}
      </button>
    </div>
    <div v-if="isOpen" class="flex gap-4">
      <button
        @click="startAllTests"
        :disabled="isRunning"
        class="bg-blue-500 hover:bg-blue-600 text-white px-4 py-2 rounded">
        Start
      </button>
      <button
        @click="stopAllTests"
        :disabled="!isRunning"
        class="bg-red-500 hover:bg-red-600 text-white px-4 py-2 rounded">
        Stop
      </button>
      <button
        @click="resetAllTests"
        class="bg-gray-500 hover:bg-gray-600 text-white px-4 py-2 rounded">
        Reset
      </button>
    </div>
  </div>
</template>

<script>
export default {
  name: 'TestAllMotors',
  props: {
    client: Object,
    deviceId: String
  },
  data() {
    return {
      isOpen: false,
      isRunning: false,
      testInterval: null,
      currentMotorIndex: 0,
      testCount: 0,
      errorLog: [],
      cycleStartTime: null
    }
  },
  methods: {
    sendCommand(command, row = '', column = '') {
      const topic = 'esp32/CtrlRelay';
      const message = row && column
        ? `${this.deviceId} ${command} ${row}:${column}`
        : `${this.deviceId} ${command}`;

      if (this.client && this.client.connected) {
        this.client.publish(topic, message, { qos: 1 }, (err) => {
          if (err) {
            console.error("Publish error:", err);
            this.errorLog.push({ motor: `${row}:${column}`, error: err.message });
          } else {
            console.log("Command sent:", message);
          }
        });
      } else {
        console.error("MQTT client belum terkoneksi");
      }
    },
    startAllTests() {
      if (!this.isRunning) {
        this.isRunning = true;
        this.currentMotorIndex = 0;
        this.testCount = 0;
        this.errorLog = [];
        this.runMotorTest();
      }
    },
    runMotorTest() {
      if (this.currentMotorIndex < 32) {
        const row = 'R' + Math.floor(this.currentMotorIndex / 8 + 1).toString().padStart(2, '0');
        const col = 'C' + ((this.currentMotorIndex % 8) + 1).toString().padStart(2, '0');
        this.$emit('start-motor-check', { index: this.currentMotorIndex, testCount: this.testCount });
        this.sendCommand('JUAL', row, col);
        this.cycleStartTime = Date.now();

        this.testInterval = setTimeout(() => {
          if (this.isRunning) {
            const cycleDuration = (Date.now() - this.cycleStartTime) / 1000;
            if (cycleDuration > 20) {
              this.errorLog.push({ motor: `${row}:${col}`, error: 'Cycle time exceeded 20 seconds' });
            }
            this.testCount++;
            if (this.testCount < 10) {
              this.runMotorTest();
            } else {
              this.testCount = 0;
              this.currentMotorIndex++;
              this.runMotorTest();
            }
          }
        }, 20000); // 20 seconds per cycle
      } else {
        this.isRunning = false;
        clearTimeout(this.testInterval);
        this.$emit('all-tests-completed', { errorLog: this.errorLog });
      }
    },
    stopAllTests() {
      if (this.isRunning) {
        this.isRunning = false;
        clearTimeout(this.testInterval);
        this.sendCommand('STOP'); // Perintah stop khusus (bisa disesuaikan di ESP32)
      }
    },
    resetAllTests() {
      this.isRunning = false;
      clearInterval(this.testInterval);
      this.$emit('reset-all-status');
    }
  }
}
</script>
