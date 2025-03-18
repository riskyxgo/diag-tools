<!-- src/components/DiagnosticTool.vue -->
<template>
  <div class="p-4">
    <h1 class="text-2xl font-bold mb-4">Diagnostic Tool Motor Control</h1>

    <!-- Input Device ID -->
    <div class="mb-6">
      <label for="deviceId" class="block text-gray-700">Device ID:</label>
      <input
        id="deviceId"
        v-model="deviceId"
        placeholder="Masukkan Device ID"
        class="w-full border rounded p-2" />
    </div>

    <!-- Komponen Check Motor Dc (untuk penjualan) -->
    <check-motor-dc
      :client="client"
      :device-id="deviceId"
      :motor-status="motorDcStatus"
      @start-motor-test="startMotorDcTest" />

    <!-- Komponen Adjust Motor Dc (untuk teknisi - ADJUST) -->
    <adjust-motor-dc :client="client" :device-id="deviceId" />

    <!-- Komponen Check Motor Stepper (untuk teknisi - tes lainnya) -->
    <check-motor-stepper
      :client="client"
      :device-id="deviceId"
      :stepper-status="stepperStatus"
      @reset-stepper-status="resetStepperStatus" />

    <!-- Komponen Status Pesan -->
    <status-messages :messages="statusMessages" @reset-messages="resetStatusMessages" />

    <!-- Komponen Status Error -->
    <status-error :errors="errorLogs" />
  </div>
</template>

<script>
import CheckMotorDc from '../components/child/CheckMotorDc.vue';
import AdjustMotorDc from '../components/child/AdjustMotorDc.vue';
import CheckMotorStepper from '../components/child/CheckMotorStepper.vue';
import StatusMessages from '../components/child/StatusMessages.vue';
import StatusError from '../components/child/StatusError.vue';

export default {
  name: 'DiagnosticTool',
  components: {
    CheckMotorDc,
    AdjustMotorDc,
    CheckMotorStepper,
    StatusMessages,
    StatusError
  },
  props: ['client'],
  data() {
    return {
      deviceId: 'ArUTx-1234',
      statusMessages: [],
      errorLogs: [],
      stepperStatus: {
        expectedMessages: [
          'Stepper putar naik',
          'Stepper sampai posisi',
          'Stepper putar turun',
          'Stepper posisi terbawah',
          'Uji Stepper Selesai'
        ],
        receivedMessages: [],
        firstMessageTime: null,
        lastMessageTime: null,
        completed: false,
        isActive: false
      },
      motorDcStatus: Array(32).fill(null).map(() => ({
        expectedMessages: [
          'Stepper putar naik',
          'Stepper sampai posisi',
          'Relay Aktif Motor Berputar',
          'Relay OFF Motor Berhenti',
          'Stepper putar turun',
          'Stepper posisi terbawah',
          'Stepper Disable'
        ],
        receivedMessages: [],
        commandSentTime: null,
        firstMessageTime: null,
        lastMessageTime: null,
        delay: null,
        totalTime: null,
        completed: false,
        isActive: false
      })),
      activeMotorIndex: null
    }
  },
  methods: {
    handleMessage(topic, message) {
      const msg = message.toString();
      const fullMessage = `Topic: ${topic} | Pesan: ${msg}`;
      this.statusMessages.push(fullMessage);

      // Handle Stepper Status
      if (this.stepperStatus.isActive && topic === 'esp32/StatusRly') {
        const nextExpectedIndex = this.stepperStatus.receivedMessages.length;
        const expectedMsg = this.stepperStatus.expectedMessages[nextExpectedIndex];

        if (msg.includes('Timeout')) {
          this.errorLogs.push({ type: 'Error', message: `Stepper - Error 01: Timeout detected` });
          this.stepperStatus.isActive = false;
        } else if (nextExpectedIndex < this.stepperStatus.expectedMessages.length && !msg.includes(expectedMsg)) {
          this.errorLogs.push({ type: 'Error', message: `Stepper - Error 02: Not sequential feedback order (Expected: ${expectedMsg}, Received: ${msg})` });
        } else if (msg.includes(expectedMsg)) {
          this.stepperStatus.receivedMessages.push(msg);
          const currentTime = Date.now();
          if (this.stepperStatus.receivedMessages.length === 1) {
            this.stepperStatus.firstMessageTime = currentTime;
          }
          if (this.stepperStatus.receivedMessages.length === this.stepperStatus.expectedMessages.length) {
            this.stepperStatus.lastMessageTime = currentTime;
            this.stepperStatus.completed = true;
            this.stepperStatus.isActive = false;
            this.checkWarnings('Stepper', this.stepperStatus);
          }
        }
      }

      // Handle Motor Dc Status
      if (this.activeMotorIndex !== null && topic === 'esp32/StatusRly') {
        const motor = this.motorDcStatus[this.activeMotorIndex];
        const nextExpectedIndex = motor.receivedMessages.length;
        const expectedMsg = motor.expectedMessages[nextExpectedIndex];

        if (msg.includes('Timeout')) {
          this.errorLogs.push({ type: 'Error', message: `Motor Dc [${this.getMotorLabel(this.activeMotorIndex)}] - Error 01: Timeout detected` });
          motor.isActive = false;
          this.activeMotorIndex = null;
        } else if (nextExpectedIndex < motor.expectedMessages.length && !msg.includes(expectedMsg)) {
          this.errorLogs.push({ type: 'Error', message: `Motor Dc [${this.getMotorLabel(this.activeMotorIndex)}] - Error 02: Not sequential feedback order (Expected: ${expectedMsg}, Received: ${msg})` });
        } else if (msg.includes(expectedMsg)) {
          motor.receivedMessages.push(msg);
          const currentTime = Date.now();

          if (motor.receivedMessages.length === 1) {
            motor.firstMessageTime = currentTime;
            motor.delay = currentTime - motor.commandSentTime;
          }
          if (motor.receivedMessages.length === motor.expectedMessages.length) {
            motor.lastMessageTime = currentTime;
            motor.totalTime = currentTime - motor.commandSentTime;
            motor.completed = true;
            motor.isActive = false;
            this.checkWarnings(`Motor Dc [${this.getMotorLabel(this.activeMotorIndex)}]`, motor);
            this.activeMotorIndex = null;
          }
        }
      }

      // Check Incorrect Feedback Count (Error 03)
      if (this.stepperStatus.isActive && this.stepperStatus.receivedMessages.length > this.stepperStatus.expectedMessages.length) {
        this.errorLogs.push({ type: 'Error', message: `Stepper - Error 03: Incorrect feedback count (Expected: ${this.stepperStatus.expectedMessages.length}, Received: ${this.stepperStatus.receivedMessages.length})` });
        this.stepperStatus.isActive = false;
      }
      if (this.activeMotorIndex !== null && this.motorDcStatus[this.activeMotorIndex].receivedMessages.length > this.motorDcStatus[this.activeMotorIndex].expectedMessages.length) {
        this.errorLogs.push({ type: 'Error', message: `Motor Dc [${this.getMotorLabel(this.activeMotorIndex)}] - Error 03: Incorrect feedback count (Expected: ${this.motorDcStatus[this.activeMotorIndex].expectedMessages.length}, Received: ${this.motorDcStatus[this.activeMotorIndex].receivedMessages.length})` });
        this.motorDcStatus[this.activeMotorIndex].isActive = false;
        this.activeMotorIndex = null;
      }
    },
    checkWarnings(component, status) {
      if (status.delay > 2000) {
        this.errorLogs.push({ type: 'Warning', message: `${component} - Warn 01: Delay too long (${status.delay} ms > 2000 ms)` });
      }
      if (status.totalTime > 20000) {
        this.errorLogs.push({ type: 'Warning', message: `${component} - Warn 02: Total time too long (${status.totalTime} ms > 20000 ms)` });
      }
    },
    getMotorLabel(index) {
      const row = Math.floor(index / 8) + 1;
      const col = (index % 8) + 1;
      return `R${row.toString().padStart(2, '0')}:C${col.toString().padStart(2, '0')}`;
    },
    resetStatusMessages() {
      this.statusMessages = [];
    },
    resetStepperStatus() {
      this.stepperStatus.receivedMessages = [];
      this.stepperStatus.firstMessageTime = null;
      this.stepperStatus.lastMessageTime = null;
      this.stepperStatus.completed = false;
      this.stepperStatus.isActive = true;
    },
    startMotorDcTest({ index, commandSentTime }) {
      if (this.activeMotorIndex === null) {
        this.activeMotorIndex = index;
        const motor = this.motorDcStatus[index];
        motor.receivedMessages = [];
        motor.commandSentTime = commandSentTime;
        motor.firstMessageTime = null;
        motor.lastMessageTime = null;
        motor.delay = null;
        motor.totalTime = null;
        motor.completed = false;
        motor.isActive = true;
      } else {
        console.log("Pengujian lain sedang berlangsung. Harap tunggu hingga selesai.");
      }
    }
  },
  mounted() {
    if (this.client) {
      this.client.on('message', (topic, message) => {
        if (topic === 'esp32/StatusRly') {
          this.handleMessage(topic, message);
        }
      });
    }
  }
}
</script>
