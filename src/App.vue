<template>
  <div>
    <!-- Sticky header status koneksi -->
    <div class="sticky top-0 z-50 bg-blue-600 text-white p-2 text-center">
      Status:
      <span :class="connected ? 'text-green-300' : 'text-red-300'">
        {{ connected ? 'Connected' : 'Not Connected' }}
      </span>
    </div>

    <div class="container mx-auto p-4">
      <!-- Tampilan Awal (Dashboard) -->
      <div v-if="currentView === 'initial'" class="flex flex-col items-center space-y-4">
        <h1 class="text-2xl font-bold mb-4">Dashboard MQTT</h1>
        <div class="flex space-x-4">
          <button
            @click="switchTo('connection')"
            class="bg-blue-500 hover:bg-blue-600 text-white px-4 py-2 rounded"
          >
            Sambungkan
          </button>
          <button
            :disabled="!connected"
            @click="switchTo('diag')"
            class="bg-green-500 hover:bg-green-600 text-white px-4 py-2 rounded disabled:opacity-50 disabled:cursor-not-allowed"
          >
            Run Diag
          </button>
        </div>
      </div>

      <!-- Tampilan Koneksi (Sambungkan) -->
      <div v-if="currentView === 'connection'" class="max-w-md mx-auto bg-white shadow rounded p-6">
        <h1 class="text-xl font-bold mb-4 text-center">Sambungkan ke MQTT</h1>
        <form @submit.prevent="connectMqtt" class="space-y-4">
          <div>
            <label for="host" class="block text-gray-700">Broker Address:</label>
            <input
              id="host"
              v-model="connection.host"
              placeholder="contoh: z25505c1.ala.asia-southeast1.emqxsl.com"
              class="w-full border rounded p-2"
            />
          </div>
          <div>
            <label for="port" class="block text-gray-700">WebSocket Port (TLS):</label>
            <input
              id="port"
              v-model="connection.port"
              type="number"
              placeholder="contoh: 8084"
              class="w-full border rounded p-2"
            />
          </div>
          <div>
            <label for="username" class="block text-gray-700">Username:</label>
            <input
              id="username"
              v-model="connection.username"
              placeholder="contoh: webcontrol"
              class="w-full border rounded p-2"
            />
          </div>
          <div>
            <label for="password" class="block text-gray-700">Password:</label>
            <input
              id="password"
              v-model="connection.password"
              type="password"
              placeholder="contoh: 123456789"
              class="w-full border rounded p-2"
            />
          </div>
          <div class="flex justify-between mt-4">
            <button
              type="submit"
              class="bg-blue-500 hover:bg-blue-600 text-white px-4 py-2 rounded"
            >
              Hubungkan
            </button>
            <button
              type="button"
              class="bg-gray-300 hover:bg-gray-200 px-6 py-2 rounded"
              @click="switchTo('initial')"
            >
              Kembali
            </button>
            <button
              type="button"
              @click="disconnectMqtt"
              class="bg-red-500 hover:bg-red-600 text-white px-4 py-2 rounded"
            >
              Putuskan
            </button>
          </div>
        </form>
        <div v-if="errorMessage" class="text-red-500 mt-4">
          {{ errorMessage }}
        </div>
      </div>

      <!-- Tampilan Diagnostic (Run Diag) -->
      <div v-if="currentView === 'diag'">
        <DiagnosticTool :client="client" />
        <div class="sticky bottom-4 z-50 p-2 text-center">
          <!-- sticky top-0 z-50 bg-blue-600 text-white p-2 text-center -->
          <button
            @click="goBackToInitial"
            class="bg-gray-500 hover:bg-gray-600 text-white px-4 py-2 rounded"
          >
            Kembali ke Dashboard
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import mqtt from 'mqtt'
import DiagnosticTool from './components/DiagnosticTool.vue'

export default {
  name: 'App',
  components: { DiagnosticTool },
  data() {
    return {
      currentView: 'initial', // 'initial' | 'connection' | 'diag'
      connected: false,
      client: null,
      errorMessage: '',
      // Nilai default sesuai konfigurasi awal
      connection: {
        host: 'z25505c1.ala.asia-southeast1.emqxsl.com',
        port: '8084', // WebSocket TLS port
        username: 'vendingdua',
        password: '123456789',
      },
    }
  },
  methods: {
    switchTo(view) {
      localStorage.removeItem('errors')
      this.currentView = view
    },
    connectMqtt() {
      this.errorMessage = ''
      if (this.client) {
        this.client.end(true)
        this.client = null
        this.connect = false // Reset state
      }

      const url = `wss://${this.connection.host}:${this.connection.port}/mqtt`
      const options = {
        protocol: 'wss',
        username: this.connection.username,
        password: this.connection.password,
        reconnectPeriod: 1000,
      }

      // Membuat koneksi MQTT
      this.client = mqtt.connect(url, options)

      this.client.on('connect', () => {
        this.connected = true
        console.log('Terkoneksi ke MQTT Broker')
        // Subscribe ke topik status
        this.client.subscribe('esp32/StatusRly', { qos: 1 }, (err) => {
          if (err) {
            console.error('Subscription error:', err)
          }
        })
        // Setelah koneksi sukses, otomatis alihkan ke tampilan diag
        this.currentView = 'diag'
      })

      this.client.on('error', (err) => {
        console.error('MQTT Connection Error:', err)
        this.errorMessage = 'Gagal terhubung: ' + err.message
        this.connected = false
      })

      this.client.on('close', () => {
        this.connected = false
      })
    },
    disconnectMqtt() {
      if (this.client) {
        this.client.end()
        this.connected = false
        console.log('Terputus dari MQTT Broker')
      }
    },
    goBackToInitial() {
      this.currentView = 'initial'
    },
  },
}
</script>

<!-- Pastikan Tailwind CSS sudah diimport secara global -->
