<!-- src/components/DiagnosticTool.vue -->
<template>
  <div class="p-4">
    <div class="flex justify-between items-center mb-4">
      <h1 class="text-2xl font-bold">Diagnostic Tool Motor Control</h1>
    </div>
    <div class="flex justify-center items-center mb-4">
      <div class="flex gap-4">
        <button
          @click="runDiagnostic"
          :disabled="isDiagnosticRunning"
          class="bg-purple-500 hover:bg-purple-600 text-white px-4 py-2 rounded disabled:bg-gray-400">
          Run Diagnostic
        </button>
        <button
          @click="cycleMultiplier"
          class="bg-yellow-500 hover:bg-yellow-600 text-white px-4 py-2 rounded">
          Multiplier: {{ multiplier }}x
        </button>
        <button
          v-if="isDiagnosticRunning"
          @click="stopDiagnostic"
          class="bg-red-500 hover:bg-red-600 text-white px-4 py-2 rounded">
          Stop Diagnostic
        </button>
        <simulation-toggle
          :is-simulation-active="isSimulationActive"
          @toggle-simulation="toggleSimulation" />
      </div>
    </div>

    <!-- Input Device ID -->
    <div class="mb-6">
      <label for="deviceId" class="block text-gray-700">Device ID:</label>
      <input
        id="deviceId"
        v-model="deviceId"
        placeholder="Masukkan Device ID"
        class="w-full border rounded p-2" />
    </div>

    <!-- Komponen Check Motor Dc -->
    <check-motor-dc
      ref="checkMotorDc"
      :client="isSimulationActive ? simulatedClient : client"
      :device-id="deviceId"
      :motor-status="motorDcStatus"
      :multiplier="multiplier"
      @start-motor-test="startMotorDcTest" />

    <!-- Komponen Adjust Motor Dc -->
    <adjust-motor-dc :client="isSimulationActive ? simulatedClient : client" :device-id="deviceId" />

    <!-- Komponen Check Motor Stepper -->
    <check-motor-stepper
      :client="isSimulationActive ? simulatedClient : client"
      :device-id="deviceId"
      :stepper-status="stepperStatus"
      @reset-stepper-status="resetStepperStatus" />

    <!-- Komponen Status Pesan -->
    <status-messages :messages="statusMessages" @reset-messages="resetStatusMessages" />

    <!-- Komponen Status Error -->
    <status-error :errors="errorLogs" @reset-errors="resetStatusError"/>

    <!-- Komponen Error Log Printout -->
    <error-log-printout
      :device-id="deviceId"
      :motor-dc-status="motorDcStatus"
      :stepper-status="stepperStatus"
      :error-logs="errorLogs"
      :multiplier="multiplier" />
  </div>
</template>

<script>
import CheckMotorDc from '../components/child/CheckMotorDc.vue';
import AdjustMotorDc from '../components/child/AdjustMotorDc.vue';
import CheckMotorStepper from '../components/child/CheckMotorStepper.vue';
import StatusMessages from '../components/child/StatusMessages.vue';
import StatusError from '../components/child/StatusError.vue';
import SimulationToggle from '../components/child/SimulationToggle.vue';
import ErrorLogPrintout from '../components/child/ErrorLogPrintout.vue';

export default {
  name: 'DiagnosticTool',
  components: {
    CheckMotorDc,
    AdjustMotorDc,
    CheckMotorStepper,
    StatusMessages,
    StatusError,
    SimulationToggle,
    ErrorLogPrintout
  },
  props: ['client'],
  data() {
    return {
      deviceId: 'ArUTx-1234',
      statusMessages: [],
      errorLogs: [],
      simulatedClient: { connected: true },
      isSimulationActive: false,
      isDiagnosticRunning: false,
      multiplier: 1, // Default multiplier 1x
      stepperStatus: {
        expectedMessages: [
          'Stepper putar naik',
          'Stepper sampai posisi',
          'Stepper putar turun',
          'Stepper posisi terbawah',
          'Uji Stepper Selesai'
        ],
        receivedMessages: [],
        commandSentTime: null,
        firstMessageTime: null,
        lastMessageTime: null,
        delay: null,
        totalTime: null,
        completed: false,
        isActive: false,
        testCount: 0,
        delayHistory: [],
        totalTimeHistory: []
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
        isActive: false,
        testCount: 0,
        delayHistory: [],
        totalTimeHistory: []
      })),
      activeMotorIndex: null
    }
  },
  methods: {
    saveError(error) {
      let errors = JSON.parse(localStorage.getItem('errors')) || [];
      errors.push(error);
      localStorage.setItem('errors', JSON.stringify(errors));
    },
    getLimitSwitchId() {
      return Math.floor(Math.random() * 4) + 11; // 11, 12, 13, atau 14
    },
    cycleMultiplier() {
      // Siklus antara 1x, 10x, 100x, 1000x, 10000x
      if (this.multiplier === 1) this.multiplier = 10;
      else if (this.multiplier === 10) this.multiplier = 100;
      else if (this.multiplier === 100) this.multiplier = 1000;
      else if (this.multiplier === 1000) this.multiplier = 10000;
      else this.multiplier = 1;
    },
    handleMessage(topic, message) {
      const msg = message.toString();
      const fullMessage = `Topic: ${topic} | Pesan: ${msg}`;
      this.statusMessages.push(fullMessage);

      if (this.stepperStatus.isActive && topic === 'esp32/StatusRly') {
        const nextExpectedIndex = this.stepperStatus.receivedMessages.length;
        const expectedMsg = this.stepperStatus.expectedMessages[nextExpectedIndex];

        if (msg.includes('Timeout')) {
          const errorData = {
            timestamp: new Date().toISOString(),
            type: 'Error',
            code: '01',
            description: 'Timeout detected',
            component: 'Stepper'
          };
          this.errorLogs.push({ type: 'Error', message: `Stepper - Error 01: Timeout detected` });
          this.saveError(errorData);
          this.stepperStatus.isActive = false;
        } else if (msg.includes('Limit Switch')) {
          const limitSwitchId = this.getLimitSwitchId();
          const errorData = {
            timestamp: new Date().toISOString(),
            type: 'Error',
            code: limitSwitchId.toString().padStart(2, '0'),
            description: `Limit Switch triggered (ID:${limitSwitchId})`,
            component: 'Stepper'
          };
          this.errorLogs.push({ type: 'Error', message: `Stepper - Error ${limitSwitchId.toString().padStart(2, '0')}: Limit Switch triggered (ID:${limitSwitchId})` });
          this.saveError(errorData);
          this.stepperStatus.isActive = false;
        } else if (nextExpectedIndex < this.stepperStatus.expectedMessages.length && !msg.includes(expectedMsg)) {
          const errorData = {
            timestamp: new Date().toISOString(),
            type: 'Error',
            code: '02',
            description: `Not sequential feedback order (Expected: ${expectedMsg}, Received: ${msg})`,
            component: 'Stepper'
          };
          this.errorLogs.push({ type: 'Error', message: `Stepper - Error 02: Not sequential feedback order (Expected: ${expectedMsg}, Received: ${msg})` });
          this.saveError(errorData);
        } else if (msg.includes(expectedMsg)) {
          this.stepperStatus.receivedMessages.push(msg);
          const currentTime = Date.now();
          if (this.stepperStatus.receivedMessages.length === 1) {
            this.stepperStatus.firstMessageTime = currentTime;
            this.stepperStatus.delay = currentTime - this.stepperStatus.commandSentTime;
            this.stepperStatus.delayHistory.push(this.stepperStatus.delay);
          }
          if (this.stepperStatus.receivedMessages.length === this.stepperStatus.expectedMessages.length) {
            this.stepperStatus.lastMessageTime = currentTime;
            this.stepperStatus.totalTime = currentTime - this.stepperStatus.commandSentTime;
            this.stepperStatus.totalTimeHistory.push(this.stepperStatus.totalTime);
            this.stepperStatus.completed = true;
            this.stepperStatus.isActive = false;
            this.checkWarnings('Stepper', this.stepperStatus);
          }
        }
      }

      if (this.activeMotorIndex !== null && topic === 'esp32/StatusRly') {
        const motor = this.motorDcStatus[this.activeMotorIndex];
        const nextExpectedIndex = motor.receivedMessages.length;
        const expectedMsg = motor.expectedMessages[nextExpectedIndex];

        if (msg.includes('Timeout')) {
          const errorData = {
            timestamp: new Date().toISOString(),
            type: 'Error',
            code: '01',
            description: 'Timeout detected',
            component: `Motor Dc [${this.getMotorLabel(this.activeMotorIndex)}]`
          };
          this.errorLogs.push({ type: 'Error', message: `Motor Dc [${this.getMotorLabel(this.activeMotorIndex)}] - Error 01: Timeout detected` });
          this.saveError(errorData);
          motor.isActive = false;
          this.activeMotorIndex = null;
        } else if (msg.includes('Limit Switch')) {
          const limitSwitchId = this.getLimitSwitchId();
          const errorData = {
            timestamp: new Date().toISOString(),
            type: 'Error',
            code: limitSwitchId.toString().padStart(2, '0'),
            description: `Limit Switch triggered (ID:${limitSwitchId})`,
            component: `Motor Dc [${this.getMotorLabel(this.activeMotorIndex)}]`
          };
          this.errorLogs.push({ type: 'Error', message: `Motor Dc [${this.getMotorLabel(this.activeMotorIndex)}] - Error ${limitSwitchId.toString().padStart(2, '0')}: Limit Switch triggered (ID:${limitSwitchId})` });
          this.saveError(errorData);
          motor.isActive = false;
          this.activeMotorIndex = null;
        } else if (nextExpectedIndex < motor.expectedMessages.length && !msg.includes(expectedMsg)) {
          const errorData = {
            timestamp: new Date().toISOString(),
            type: 'Error',
            code: '02',
            description: `Not sequential feedback order (Expected: ${expectedMsg}, Received: ${msg})`,
            component: `Motor Dc [${this.getMotorLabel(this.activeMotorIndex)}]`
          };
          this.errorLogs.push({ type: 'Error', message: `Motor Dc [${this.getMotorLabel(this.activeMotorIndex)}] - Error 02: Not sequential feedback order (Expected: ${expectedMsg}, Received: ${msg})` });
          this.saveError(errorData);
        } else if (msg.includes(expectedMsg)) {
          motor.receivedMessages.push(msg);
          const currentTime = Date.now();

          if (motor.receivedMessages.length === 1) {
            motor.firstMessageTime = currentTime;
            motor.delay = currentTime - motor.commandSentTime;
            motor.delayHistory.push(motor.delay);
          }
          if (motor.receivedMessages.length === motor.expectedMessages.length) {
            motor.lastMessageTime = currentTime;
            motor.totalTime = currentTime - motor.commandSentTime;
            motor.totalTimeHistory.push(motor.totalTime);
            motor.completed = true;
            motor.isActive = false;
            this.checkWarnings(`Motor Dc [${this.getMotorLabel(this.activeMotorIndex)}]`, motor);
            this.activeMotorIndex = null;
          }
        }
      }

      if (this.stepperStatus.isActive && this.stepperStatus.receivedMessages.length > this.stepperStatus.expectedMessages.length) {
        const errorData = {
          timestamp: new Date().toISOString(),
          type: 'Error',
          code: '03',
          description: `Incorrect feedback count (Expected: ${this.stepperStatus.expectedMessages.length}, Received: ${this.stepperStatus.receivedMessages.length})`,
          component: 'Stepper'
        };
        this.errorLogs.push({ type: 'Error', message: `Stepper - Error 03: Incorrect feedback count (Expected: ${this.stepperStatus.expectedMessages.length}, Received: ${this.stepperStatus.receivedMessages.length})` });
        this.saveError(errorData);
        this.stepperStatus.isActive = false;
      }
      if (this.activeMotorIndex !== null && this.motorDcStatus[this.activeMotorIndex].receivedMessages.length > this.motorDcStatus[this.activeMotorIndex].expectedMessages.length) {
        const errorData = {
          timestamp: new Date().toISOString(),
          type: 'Error',
          code: '03',
          description: `Incorrect feedback count (Expected: ${this.motorDcStatus[this.activeMotorIndex].expectedMessages.length}, Received: ${this.motorDcStatus[this.activeMotorIndex].receivedMessages.length})`,
          component: `Motor Dc [${this.getMotorLabel(this.activeMotorIndex)}]`
        };
        this.errorLogs.push({ type: 'Error', message: `Motor Dc [${this.getMotorLabel(this.activeMotorIndex)}] - Error 03: Incorrect feedback count (Expected: ${this.motorDcStatus[this.activeMotorIndex].expectedMessages.length}, Received: ${this.motorDcStatus[this.activeMotorIndex].receivedMessages.length})` });
        this.saveError(errorData);
        this.motorDcStatus[this.activeMotorIndex].isActive = false;
        this.activeMotorIndex = null;
      }
    },
    checkWarnings(component, status) {
      if (status.delay > 2000) {
        const warnData = {
          timestamp: new Date().toISOString(),
          type: 'Warning',
          code: '01',
          description: `Delay too long (${status.delay} ms > 2000 ms)`,
          component: component
        };
        this.errorLogs.push({ type: 'Warning', message: `${component} - Warn 01: Delay too long (${status.delay} ms > 2000 ms)` });
        this.saveError(warnData);
      }
      if (status.totalTime > 25000) {
        const warnData = {
          timestamp: new Date().toISOString(),
          type: 'Warning',
          code: '02',
          description: `Total time too long (${status.totalTime} ms > 25000 ms)`,
          component: component
        };
        this.errorLogs.push({ type: 'Warning', message: `${component} - Warn 02: Total time too long (${status.totalTime} ms > 25000 ms)` });
        this.saveError(warnData);
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
    resetStatusError() {
      this.errorLogs = [];
    },
    resetStepperStatus() {
      this.stepperStatus.receivedMessages = [];
      this.stepperStatus.commandSentTime = Date.now();
      this.stepperStatus.firstMessageTime = null;
      this.stepperStatus.lastMessageTime = null;
      this.stepperStatus.delay = null;
      this.stepperStatus.totalTime = null;
      this.stepperStatus.completed = false;
      this.stepperStatus.isActive = true;
      this.stepperStatus.testCount++;
      if (this.isSimulationActive) {
        this.simulateStepperMessages();
      }
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
        motor.testCount++;
        if (this.isSimulationActive) {
          this.simulateMotorDcMessages(index);
        }
      } else {
        console.log("Pengujian lain sedang berlangsung. Harap tunggu hingga selesai.");
      }
    },
    simulateStepperMessages() {
      const messages = this.stepperStatus.expectedMessages.slice();
      let currentIndex = 0;
      const simulate = () => {
        if (!this.stepperStatus.isActive || currentIndex >= messages.length + 1) return;

        const randomDelay = Math.floor(Math.random() * 400) + 100;
        setTimeout(() => {
          const rand = Math.random();
          let simulatedMsg;

          if (rand < 0.1) {
            simulatedMsg = "Timeout occurred";
          } else if (rand < 0.15) {
            const limitSwitchId = this.getLimitSwitchId();
            simulatedMsg = `Limit Switch triggered (ID:${limitSwitchId})`;
          } else if (rand < 0.2) {
            simulatedMsg = messages[Math.floor(Math.random() * messages.length)];
          } else if (currentIndex < messages.length) {
            simulatedMsg = `Simulated - ${messages[currentIndex]}`;
            currentIndex++;
          } else {
            simulatedMsg = "Extra message";
          }

          this.handleMessage('esp32/StatusRly', simulatedMsg);
          simulate();
        }, randomDelay);
      };
      simulate();
    },
    simulateMotorDcMessages(index) {
      const motor = this.motorDcStatus[index];
      const messages = motor.expectedMessages.slice();
      let currentIndex = 0;
      const simulate = () => {
        if (!motor.isActive || currentIndex >= messages.length + 1) return;

        setTimeout(() => {
          const rand = Math.random();
          let simulatedMsg;

          if (rand < 0.1) {
            simulatedMsg = "Timeout occurred";
          } else if (rand < 0.15) {
            const limitSwitchId = this.getLimitSwitchId();
            simulatedMsg = `Limit Switch triggered (ID:${limitSwitchId})`;
          } else if (rand < 0.2) {
            simulatedMsg = messages[Math.floor(Math.random() * messages.length)];
          } else if (currentIndex < messages.length) {
            simulatedMsg = `Simulated - ${messages[currentIndex]}`;
            currentIndex++;
          } else {
            simulatedMsg = "Extra message";
          }

          this.handleMessage('esp32/StatusRly', simulatedMsg);
          simulate();
        }, 100);
      };
      simulate();
    },
    toggleSimulation(value) {
      this.isSimulationActive = value;
      if (!this.isSimulationActive) {
        this.stepperStatus.isActive = false;
        this.activeMotorIndex = null;
        this.motorDcStatus.forEach(motor => (motor.isActive = false));
        this.isDiagnosticRunning = false;
      }
    },
    async runDiagnostic() {
      if (this.isDiagnosticRunning) return;

      this.isDiagnosticRunning = true;
      console.log(`Starting full diagnostic with multiplier ${this.multiplier}x...`);

      const checkMotorDc = this.$refs.checkMotorDc;
      if (!checkMotorDc) {
        console.error("CheckMotorDc component not found.");
        this.isDiagnosticRunning = false;
        return;
      }

      // Jalankan semua motor DC satu per satu
      for (let index = 0; index < this.motorDcStatus.length; index++) {
        if (!this.isDiagnosticRunning) {
          console.log("Diagnostic stopped.");
          this.activeMotorIndex = null;
          return;
        }

        const row = checkMotorDc.getRow(index);
        const column = checkMotorDc.getColumn(index);

        // Ulangi pengujian untuk motor ini sebanyak multiplier
        for (let i = 0; i < this.multiplier; i++) {
          if (!this.isDiagnosticRunning) {
            console.log("Diagnostic stopped during multiplier loop.");
            this.activeMotorIndex = null;
            return;
          }

          console.log(`Testing Motor Dc [${this.getMotorLabel(index)}] - Iteration ${i + 1}/${this.multiplier}`);
          checkMotorDc.sendCommand('JUAL', row, column, index);

          // Tunggu hingga pengujian motor selesai
          await new Promise(resolve => {
            const checkCompletion = () => {
              if (this.activeMotorIndex === null || !this.isDiagnosticRunning) {
                resolve();
              } else {
                setTimeout(checkCompletion, 100);
              }
            };
            checkCompletion();
          });
        }
      }

      // Jalankan stepper setelah semua motor selesai (tanpa multiplier)
      if (this.isDiagnosticRunning) {
        console.log("Testing Stepper...");
        this.resetStepperStatus();

        await new Promise(resolve => {
          const checkCompletion = () => {
            if (!this.stepperStatus.isActive || !this.isDiagnosticRunning) {
              resolve();
            } else {
              setTimeout(checkCompletion, 100);
            }
          };
          checkCompletion();
        });
      }

      if (this.isDiagnosticRunning) {
        console.log("Diagnostic completed.");
        this.isDiagnosticRunning = false;
      }
    },
    stopDiagnostic() {
      this.isDiagnosticRunning = false;
      this.activeMotorIndex = null;
      this.stepperStatus.isActive = false;
      this.motorDcStatus.forEach(motor => (motor.isActive = false));
      console.log("Diagnostic stopped by user.");
    }
  },
  mounted() {
    if (this.client && !this.isSimulationActive) {
      this.client.on('message', (topic, message) => {
        if (topic === 'esp32/StatusRly') {
          this.handleMessage(topic, message);
        }
      });
    }
  }
}
</script>
