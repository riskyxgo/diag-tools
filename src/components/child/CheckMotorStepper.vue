<!-- src/components/child/CheckMotorStepper.vue -->
<template>
  <div class="mb-8">
    <div class="flex justify-between items-center mb-2">
      <h2 class="text-xl font-semibold">Check Motor Stepper</h2>
      <button
        @click="isOpen = !isOpen"
        class="bg-gray-500 hover:bg-gray-600 text-white px-2 py-1 rounded">
        {{ isOpen ? 'Close' : 'Open' }}
      </button>
    </div>
    <div v-if="isOpen" class="flex flex-col gap-4">
      <div class="flex items-center gap-4 relative">
        <button
          @click="sendTestStepperCommand()"
          @mouseenter="isHovered = true"
          @mouseleave="isHovered = false"
          class="bg-purple-500 hover:bg-purple-600 text-white px-4 py-2 rounded">
          {{ isStepperOk ? 'OK' : 'Test Stepper' }}
        </button>
        <div
          v-if="isHovered && (stepperDelay !== null || totalTime !== null)"
          class="absolute z-10 bg-gray-800 text-white text-sm p-2 rounded shadow-lg -top-10 left-20">
          <span v-if="stepperDelay !== null">Delay: {{ stepperDelay }} ms</span>
          <span v-if="totalTime !== null" class="ml-2">Total: {{ totalTime }} ms</span>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'CheckMotorStepper',
  props: {
    client: Object,
    deviceId: String,
    stepperStatus: Object
  },
  data() {
    return {
      isOpen: false,
      stepperDelay: null,
      totalTime: null,
      commandSentTime: null,
      isHovered: false
    }
  },
  computed: {
    isStepperOk() {
      return this.stepperStatus && this.stepperStatus.completed;
    }
  },
  watch: {
    'stepperStatus.firstMessageTime'(newTime) {
      if (this.commandSentTime && newTime) {
        this.stepperDelay = newTime - this.commandSentTime;
      }
    },
    'stepperStatus.lastMessageTime'(newTime) {
      if (this.commandSentTime && newTime) {
        this.totalTime = newTime - this.commandSentTime;
      }
    }
  },
  methods: {
    sendDiagnosticCommand(command, position = '') {
      const topic = 'esp32/CtrlRelay';
      const message = position
        ? `${this.deviceId} ${command} ${position}`
        : `${this.deviceId} ${command}`;

      if (this.client && this.client.connected) {
        this.client.publish(topic, message, { qos: 1 }, (err) => {
          if (err) {
            console.error("Publish error:", err);
          } else {
            console.log("Diagnostic command sent:", message);
          }
        });
      } else {
        console.error("MQTT client belum terkoneksi");
      }
    },
    sendTestStepperCommand() {
      this.commandSentTime = Date.now();
      this.stepperDelay = null;
      this.totalTime = null;
      this.$emit('reset-stepper-status');
      this.sendDiagnosticCommand('TEST-STEPPER');
    }
  }
}
</script>
