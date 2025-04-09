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
          class="bg-purple-500 hover:bg-purple-600 text-white px-4 py-2 rounded disabled:bg-gray-400"
        >
          Run Diagnostic
        </button>
        <button
          @click="cycleMultiplier"
          class="bg-yellow-500 hover:bg-yellow-600 text-white px-4 py-2 rounded"
        >
          Multiplier: {{ multiplier }}x
        </button>
        <button
          v-if="isDiagnosticRunning"
          @click="stopDiagnostic"
          class="bg-red-500 hover:bg-red-600 text-white px-4 py-2 rounded"
        >
          Stop Diagnostic
        </button>
        <simulation-toggle
          :is-simulation-active="isSimulationActive"
          @toggle-simulation="toggleSimulation"
        />
      </div>
    </div>

    <!-- Input Device ID -->
    <div class="mb-6">
      <label for="deviceId" class="block text-gray-700">Device ID:</label>
      <input
        id="deviceId"
        v-model="deviceId"
        placeholder="Masukkan Device ID"
        class="w-full border rounded p-2"
      />
    </div>

    <!-- Komponen Check Motor Dc -->
    <check-motor-dc
      ref="checkMotorDc"
      :client="isSimulationActive ? simulatedClient : client"
      :device-id="deviceId"
      :motor-status="motorDcStatus"
      :multiplier="multiplier"
      @start-motor-test="startMotorDcTest"
      @timeout-motor-test="handleMotorDcTimeout"
    />

    <!-- Komponen Adjust Motor Dc -->
    <adjust-motor-dc
      :client="isSimulationActive ? simulatedClient : client"
      :device-id="deviceId"
    />

    <!-- Komponen Check Motor Stepper -->
    <check-motor-stepper
      :client="isSimulationActive ? simulatedClient : client"
      :device-id="deviceId"
      :stepper-status="stepperStatus"
      @reset-stepper-status="resetStepperStatus"
    />

    <!-- Komponen Status Pesan -->
    <status-messages :messages="statusMessages" @reset-messages="resetStatusMessages" />

    <!-- Komponen Status Error -->
    <status-error :errors="errorLogs" @reset-errors="resetStatusError" />

    <!-- Komponen Error Log Printout -->
    <error-log-printout
      :device-id="deviceId"
      :motor-dc-status="motorDcStatus"
      :stepper-status="stepperStatus"
      :error-logs="errorLogs"
      :multiplier="multiplier"
    />
  </div>
</template>

<script>
import CheckMotorDc from '../components/child/CheckMotorDc.vue'
import AdjustMotorDc from '../components/child/AdjustMotorDc.vue'
import CheckMotorStepper from '../components/child/CheckMotorStepper.vue'
import StatusMessages from '../components/child/StatusMessages.vue'
import StatusError from '../components/child/StatusError.vue'
import SimulationToggle from '../components/child/SimulationToggle.vue'
import ErrorLogPrintout from '../components/child/ErrorLogPrintout.vue'

export default {
  name: 'DiagnosticTool',
  components: {
    CheckMotorDc,
    AdjustMotorDc,
    CheckMotorStepper,
    StatusMessages,
    StatusError,
    SimulationToggle,
    ErrorLogPrintout,
  },
  props: ['client'],
  data() {
    return {
      deviceId: 'QrtVB-8888',
      statusMessages: [],
      errorLogs: [],
      simulatedClient: { connected: true },
      isSimulationActive: false,
      isDiagnosticRunning: false,
      multiplier: 1,
      stepperStatus: {
        expectedMessages: [
          'Stepper putar naik',
          'Stepper sampai posisi',
          'Stepper putar turun',
          'Stepper posisi terbawah',
          'Uji Stepper Selesai',
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
        totalTimeHistory: [],
      },
      motorDcStatus: Array(48)
        .fill(null)
        .map(() => ({
          expectedMessages: [
            'Stepper putar naik',
            'Stepper sampai posisi',
            'Relay Aktif Motor Berputar',
            'Relay OFF Motor Berhenti',
            'Stepper putar turun',
            'Stepper posisi terbawah',
            'Stepper Disable',
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
          totalTimeHistory: [],
        })),
      activeMotorIndex: null,
      abortController: null,
    }
  },
  methods: {
    handleBeforeUnload() {
      this.stopDiagnostic() // Hentikan diagnostik saat web ditutup
    },
    saveError(error) {
      let errors = JSON.parse(localStorage.getItem('errors')) || []
      errors.push(error)
      localStorage.setItem('errors', JSON.stringify(errors))
    },
    getLimitSwitchId() {
      return Math.floor(Math.random() * 4) + 11
    },
    cycleMultiplier() {
      if (this.multiplier === 1) this.multiplier = 10
      else if (this.multiplier === 10) this.multiplier = 100
      else if (this.multiplier === 100) this.multiplier = 1000
      else if (this.multiplier === 1000) this.multiplier = 10000
      else this.multiplier = 1
    },
    handleMessage(topic, message) {
      const msg = message.toString()
      console.log(
        `Pesan diterima: ${msg}, Active Motor: ${this.activeMotorIndex !== null ? this.getMotorLabel(this.activeMotorIndex) : 'None'}`,
      )
      const fullMessage = `Topic: ${topic} | Pesan: ${msg}`
      this.statusMessages.push(fullMessage)

      if (this.activeMotorIndex !== null && topic === 'esp32/StatusRly') {
        const motor = this.motorDcStatus[this.activeMotorIndex]
        const nextExpectedIndex = motor.receivedMessages.length
        const expectedMsg = motor.expectedMessages[nextExpectedIndex]

        if (msg.includes('Timeout')) {
          if (motor.receivedMessages.length === 0) {
            const errorData = {
              timestamp: new Date().toISOString(),
              type: 'Error',
              code: '01',
              description: 'Timeout detected - No response from motor',
              component: `Motor Dc [${this.getMotorLabel(this.activeMotorIndex)}]`,
            }
            this.errorLogs.push({
              type: 'Error',
              message: `Motor Dc [${this.getMotorLabel(this.activeMotorIndex)}] - Error 01: Timeout detected - No response from motor`,
            })
            this.saveError(errorData)
          }
          motor.isActive = false
          this.activeMotorIndex = null
          return
        }

        const msgWithoutPrefix = msg.replace(/^Menu \d+ - /, '').trim()
        if (msgWithoutPrefix === expectedMsg) {
          motor.receivedMessages.push(msg)
          const currentTime = Date.now()
          if (motor.receivedMessages.length === 1) {
            motor.firstMessageTime = currentTime
            motor.delay = currentTime - motor.commandSentTime
            motor.delayHistory.push(motor.delay)
          }
          if (motor.receivedMessages.length === motor.expectedMessages.length) {
            motor.lastMessageTime = currentTime
            motor.totalTime = currentTime - motor.commandSentTime
            motor.totalTimeHistory.push(motor.totalTime)
            motor.completed = true
            motor.isActive = false
            this.checkWarnings(`Motor Dc [${this.getMotorLabel(this.activeMotorIndex)}]`, motor)
            this.activeMotorIndex = null
          }
        } else if (nextExpectedIndex < motor.expectedMessages.length) {
          const errorData = {
            timestamp: new Date().toISOString(),
            type: 'Error',
            code: '02',
            description: `Not sequential feedback order (Expected: ${expectedMsg}, Received: ${msg})`,
            component: `Motor Dc [${this.getMotorLabel(this.activeMotorIndex)}]`,
          }
          this.errorLogs.push({
            type: 'Error',
            message: `Motor Dc [${this.getMotorLabel(this.activeMotorIndex)}] - Error 02: Not sequential feedback order (Expected: ${expectedMsg}, Received: ${msg})`,
          })
          this.saveError(errorData)
        }
      }
    },
    checkWarnings(component, status) {
      if (status.delay > 2000) {
        const warnData = {
          timestamp: new Date().toISOString(),
          type: 'Warning',
          code: '01',
          description: `Delay too long (${status.delay} ms > 2000 ms)`,
          component: component,
        }
        this.errorLogs.push({
          type: 'Warning',
          message: `${component} - Warn 01: Delay too long (${status.delay} ms > 2000 ms)`,
        })
        this.saveError(warnData)
      }
      if (status.totalTime > 25000) {
        const warnData = {
          timestamp: new Date().toISOString(),
          type: 'Warning',
          code: '02',
          description: `Total time too long (${status.totalTime} ms > 25000 ms)`,
          component: component,
        }
        this.errorLogs.push({
          type: 'Warning',
          message: `${component} - Warn 02: Total time too long (${status.totalTime} ms > 25000 ms)`,
        })
        this.saveError(warnData)
      }
    },
    getMotorLabel(index) {
      const row = Math.floor(index / 8) + 1
      const col = (index % 8) + 1
      return `R${row.toString().padStart(2, '0')}:C${col.toString().padStart(2, '0')}`
    },
    resetStatusMessages() {
      this.statusMessages = []
    },
    resetStatusError() {
      this.errorLogs = []
    },
    resetStepperStatus() {
      this.stepperStatus.receivedMessages = []
      this.stepperStatus.commandSentTime = Date.now()
      this.stepperStatus.firstMessageTime = null
      this.stepperStatus.lastMessageTime = null
      this.stepperStatus.delay = null
      this.stepperStatus.totalTime = null
      this.stepperStatus.completed = false
      this.stepperStatus.isActive = true
      this.stepperStatus.testCount++
      if (this.isSimulationActive) {
        this.simulateStepperMessages()
      }
    },
    startMotorDcTest({ index, commandSentTime }) {
      if (this.activeMotorIndex === null) {
        this.activeMotorIndex = index
        const motor = this.motorDcStatus[index]
        motor.receivedMessages = []
        motor.commandSentTime = commandSentTime
        motor.firstMessageTime = null
        motor.lastMessageTime = null
        motor.delay = null
        motor.totalTime = null
        motor.completed = false
        motor.isActive = true
        motor.testCount++
        if (this.isSimulationActive) {
          this.simulateMotorDcMessages(index)
        }
      } else {
        console.log('Pengujian lain sedang berlangsung. Harap tunggu hingga selesai.')
      }
    },
    handleMotorDcTimeout({ index }) {
      const motor = this.motorDcStatus[index]
      motor.isActive = false
      this.activeMotorIndex = null
      const errorData = {
        timestamp: new Date().toISOString(),
        type: 'Error',
        code: '01',
        description: 'Timeout detected - No response from motor',
        component: `Motor Dc [${this.getMotorLabel(index)}]`,
      }
      this.errorLogs.push({
        type: 'Error',
        message: `Motor Dc [${this.getMotorLabel(index)}] - Error 01: Timeout detected - No response from motor`,
      })
      this.saveError(errorData)
    },
    simulateStepperMessages() {
      const messages = this.stepperStatus.expectedMessages.slice()
      let currentIndex = 0
      const simulate = () => {
        if (!this.stepperStatus.isActive || currentIndex >= messages.length + 1) return

        const randomDelay = Math.floor(Math.random() * 400) + 100
        setTimeout(() => {
          const rand = Math.random()
          let simulatedMsg

          if (rand < 0.1) {
            simulatedMsg = 'Timeout occurred'
          } else if (rand < 0.15) {
            const limitSwitchId = this.getLimitSwitchId()
            simulatedMsg = `Limit Switch triggered (ID:${limitSwitchId})`
          } else if (rand < 0.2) {
            simulatedMsg = messages[Math.floor(Math.random() * messages.length)]
          } else if (currentIndex < messages.length) {
            simulatedMsg = `Simulated - ${messages[currentIndex]}`
            currentIndex++
          } else {
            simulatedMsg = 'Extra message'
          }

          this.handleMessage('esp32/StatusRly', simulatedMsg)
          simulate()
        }, randomDelay)
      }
      simulate()
    },
    simulateMotorDcMessages(index) {
      const motor = this.motorDcStatus[index]
      const messages = motor.expectedMessages.slice()
      let currentIndex = 0
      const simulate = () => {
        if (!motor.isActive || currentIndex >= messages.length + 1) return

        setTimeout(() => {
          const rand = Math.random()
          let simulatedMsg

          if (rand < 0.1) {
            simulatedMsg = 'Timeout occurred'
          } else if (rand < 0.15) {
            const limitSwitchId = this.getLimitSwitchId()
            simulatedMsg = `Limit Switch triggered (ID:${limitSwitchId})`
          } else if (rand < 0.2) {
            simulatedMsg = messages[Math.floor(Math.random() * messages.length)]
          } else if (currentIndex < messages.length) {
            simulatedMsg = `Simulated - ${messages[currentIndex]}`
            currentIndex++
          } else {
            simulatedMsg = 'Extra message'
          }

          this.handleMessage('esp32/StatusRly', simulatedMsg)
          simulate()
        }, 100)
      }
      simulate()
    },
    toggleSimulation(value) {
      this.isSimulationActive = value
      if (!this.isSimulationActive) {
        this.stepperStatus.isActive = false
        this.activeMotorIndex = null
        this.motorDcStatus.forEach((motor) => (motor.isActive = false))
        this.isDiagnosticRunning = false
      }
    },

    async runDiagnostic() {
      if (this.isDiagnosticRunning) return

      this.isDiagnosticRunning = true
      this.abortController = new AbortController()
      console.log(`Starting full diagnostic with multiplier ${this.multiplier}x...`)

      const checkMotorDc = this.$refs.checkMotorDc
      if (!checkMotorDc) {
        console.error('CheckMotorDc component not found.')
        this.isDiagnosticRunning = false
        this.abortController = null
        return
      }

      try {
        for (let index = 0; index < this.motorDcStatus.length; index++) {
          if (!this.isDiagnosticRunning || this.abortController.signal.aborted) {
            console.log('Diagnostic stopped.')
            this.activeMotorIndex = null
            return
          }

          const row = checkMotorDc.getRow(index)
          const column = checkMotorDc.getColumn(index)

          for (let i = 0; i < this.multiplier; i++) {
            if (!this.isDiagnosticRunning || this.abortController.signal.aborted) {
              console.log('Diagnostic stopped during multiplier loop.')
              this.activeMotorIndex = null
              return
            }

            console.log(
              `Testing Motor Dc [${this.getMotorLabel(index)}] - Iteration ${i + 1}/${this.multiplier}`,
            )
            const testPromise = checkMotorDc.sendCommand('JUAL', row, column, index)

            // Batasi waktu maksimum 50 detik per perintah
            await Promise.race([
              testPromise,
              new Promise((_, reject) =>
                setTimeout(() => reject(new Error('Timeout after 50 seconds')), 50000),
              ),
            ]).catch((err) => {
              console.log(
                `Motor Dc [${this.getMotorLabel(index)}] timed out or errored: ${err.message}`,
              )
              this.handleMotorDcTimeout({ index }) // Panggil timeout handler
            })
          }
        }

        if (this.isDiagnosticRunning && !this.abortController.signal.aborted) {
          console.log('Diagnostic completed.')
          this.isDiagnosticRunning = false
        }
      } catch (error) {
        console.error('Error during diagnostic:', error)
      } finally {
        this.abortController = null
      }
    },
    stopDiagnostic() {
      if (this.isDiagnosticRunning && this.abortController) {
        this.abortController.abort() // Batalkan semua Promise dan timeout
        this.isDiagnosticRunning = false
        this.activeMotorIndex = null
        this.stepperStatus.isActive = false
        this.motorDcStatus.forEach((motor) => (motor.isActive = false))
        console.log('Diagnostic stopped by user.')
      }
    },
  },
  mounted() {
    if (this.client && !this.isSimulationActive) {
      console.log('Mengatur listener MQTT...') // Tambahkan ini
      this.client.on('message', (topic, message) => {
        console.log('Pesan MQTT diterima:', topic, message) // Tambahkan ini
        if (topic === 'esp32/StatusRly') {
          this.handleMessage(topic, message)
        }
      })
    } else {
      console.log('MQTT tidak diatur. Client:', this.client, 'Simulasi:', this.isSimulationActive)
    }
    window.addEventListener('beforeunload', this.handleBeforeUnload)
  },

  beforeUnmount() {
    // Bersihkan event listener saat komponen dihancurkan
    window.removeEventListener('beforeunload', this.handleBeforeUnload)
    this.stopDiagnostic() // Hentikan diagnostik jika masih berjalan
  },
}
</script>
