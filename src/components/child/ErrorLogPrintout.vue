<!-- src/components/child/ErrorLogPrintout.vue -->
<script>
import jsPDF from 'jspdf'
import Chart from 'chart.js/auto'

export default {
  name: 'ErrorLogPrintout',
  props: {
    deviceId: String,
    motorDcStatus: Array,
    stepperStatus: Object,
    errorLogs: Array,
  },
  methods: {
    getErrors() {
      return JSON.parse(localStorage.getItem('errors')) || []
    },
    getMotorLabel(index) {
      const row = Math.floor(index / 8) + 1
      const col = (index % 8) + 1
      return `R${row.toString().padStart(2, '0')}:C${col.toString().padStart(2, '0')}`
    },
    getFormattedDate() {
      const now = new Date()
      const dayNames = [
        'Sunday',
        'Monday',
        'Tuesday',
        'Wednesday',
        'Thursday',
        'Friday',
        'Saturday',
      ]
      return `${now.getDate().toString().padStart(2, '0')}/${(now.getMonth() + 1).toString().padStart(2, '0')}/${now.getFullYear()}: ${now.getHours().toString().padStart(2, '0')}:${now.getMinutes().toString().padStart(2, '0')}:${now.getSeconds().toString().padStart(2, '0')} (WIB) (${dayNames[now.getDay()]})`
    },
    getResultAndCode(status, index) {
      const label = index === null ? 'Stepper' : `Motor Dc [${this.getMotorLabel(index)}]`
      const logs = this.errorLogs.filter((log) => log.message.includes(label))

      if (status.completed) {
        if (logs.length > 0 && logs.some((log) => log.type === 'Warning')) {
          const warnCodes = logs
            .filter((log) => log.type === 'Warning')
            .map((log) => log.message.match(/Warn \d\d/)[0])
            .join(', ')
          return { result: 'Good', code: warnCodes }
        }
        return { result: 'Good', code: '-' }
      } else {
        if (logs.length > 0 && logs.some((log) => log.type === 'Error')) {
          const errorCodes = logs
            .filter((log) => log.type === 'Error')
            .map((log) => log.message.match(/Error \d\d/)[0])
            .join(', ')
          return { result: 'Error', code: errorCodes }
        }
        return { result: 'Error', code: '-' }
      }
    },
    calculateAverage(history) {
      if (history.length === 0) return '-'
      const sum = history.reduce((a, b) => a + b, 0)
      return Math.round(sum / history.length) + ' ms'
    },
    truncateText(text, maxWidth, doc) {
      let truncatedText = text
      while (doc.getTextWidth(truncatedText) > maxWidth && truncatedText.length > 0) {
        truncatedText = truncatedText.slice(0, -1)
      }
      if (truncatedText.length < text.length) {
        truncatedText = truncatedText.slice(0, -3) + '...'
      }
      return truncatedText
    },
    async createGroupChart(motors, title) {
      const canvas = document.getElementById('tempChart')
      canvas.width = 600
      canvas.height = 400

      const ctx = canvas.getContext('2d')
      const datasets = motors
        .map(({ motor, index }) => {
          const iterations = motor.totalTimeHistory.map((_, i) => i + 1)
          const elapsedTimes = motor.totalTimeHistory
          const label = this.getMotorLabel(index)
          const colors = [
            'rgba(54, 162, 235, 0.8)',
            'rgba(255, 99, 132, 0.8)',
            'rgba(75, 192, 192, 0.8)',
            'rgba(255, 159, 64, 0.8)',
            'rgba(153, 102, 255, 0.8)',
            'rgba(255, 205, 86, 0.8)',
            'rgba(0, 128, 0, 0.8)',
            'rgba(128, 0, 128, 0.8)',
          ]
          const colorIndex = index % colors.length

          return {
            label: `Motor ${label}`,
            data: elapsedTimes.map((time, i) => ({ x: iterations[i], y: time })),
            backgroundColor: colors[colorIndex],
            borderColor: colors[colorIndex].replace('0.8', '1'),
            pointRadius: 5,
            showLine: true,
          }
        })
        .filter((dataset) => dataset.data.length > 0)

      const chart = new Chart(ctx, {
        type: 'scatter',
        data: { datasets },
        options: {
          responsive: false,
          scales: {
            x: {
              title: { display: true, text: 'Iteration' },
              min: 1,
              ticks: { stepSize: 1 },
            },
            y: {
              title: { display: true, text: 'Total Elapsed Time (ms)' },
              beginAtZero: true,
            },
          },
          plugins: {
            title: { display: true, text: title },
            legend: { display: true },
          },
        },
      })

      await new Promise((resolve) => setTimeout(resolve, 500))
      const imgData = canvas.toDataURL('image/png')
      chart.destroy()
      return imgData
    },
    async printErrors() {
      const errors = this.getErrors()
      const doc = new jsPDF()
      const pageHeight = doc.internal.pageSize.height
      const pageWidth = doc.internal.pageSize.width
      const maxTextWidth = pageWidth - 20

      // SLIDE 1: Diagnostic Results
      doc.setFontSize(12)
      doc.text(`Unit ID: ${this.deviceId || 'Unknown'}`, 10, 10)
      doc.text(`Date: ${this.getFormattedDate()}`, 10, 16)

      // Motor DC Diagnostic
      doc.text('MOTOR DC DIAGNOSTIC', 10, 22)
      doc.setFontSize(10)
      doc.text('ID', 10, 27)
      doc.text('AVG DELAY', 40, 27)
      doc.text('AVG ELAPSED', 70, 27)
      doc.text('ITERATIONS', 100, 27)
      doc.text('RESULT', 130, 27)
      doc.line(10, 28, 190, 28)
      let y = 32

      const testedMotors = this.motorDcStatus
        .map((motor, index) => (motor.receivedMessages.length > 0 ? { motor, index } : null))
        .filter((item) => item !== null)

      testedMotors.forEach(({ motor, index }) => {
        if (y > pageHeight - 30) {
          doc.addPage()
          y = 10
        }
        const { result } = this.getResultAndCode(motor, index)
        const avgDelay = this.calculateAverage(motor.delayHistory)
        const avgTotalTime = this.calculateAverage(motor.totalTimeHistory)
        const iterations = motor.testCount
        doc.text(`ID:[${this.getMotorLabel(index)}]`, 10, y)
        doc.text(avgDelay, 40, y)
        doc.text(avgTotalTime, 70, y)
        doc.text(`${iterations}x`, 100, y)
        doc.text(result, 130, y)
        y += 5
      })

      // Motor Stepper
      y += 5
      if (y > pageHeight - 30) {
        doc.addPage()
        y = 10
      }
      doc.setFontSize(12)
      doc.text('MOTOR STEPPER', 10, y)
      y += 6
      doc.setFontSize(10)
      doc.text('ID', 10, y)
      doc.text('AVG DELAY', 40, y)
      doc.text('AVG ELAPSED', 70, y)
      doc.text('ITERATIONS', 100, y)
      doc.text('RESULT', 130, y)
      doc.line(10, y + 1, 190, y + 1)
      y += 5

      if (this.stepperStatus.receivedMessages.length > 0) {
        if (y > pageHeight - 30) {
          doc.addPage()
          y = 10
        }
        const { result } = this.getResultAndCode(this.stepperStatus, null)
        const avgDelay = this.calculateAverage(this.stepperStatus.delayHistory)
        const avgTotalTime = this.calculateAverage(this.stepperStatus.totalTimeHistory)
        const iterations = this.stepperStatus.testCount
        doc.text('ID:01', 10, y)
        doc.text(avgDelay, 40, y)
        doc.text(avgTotalTime, 70, y)
        doc.text(`${iterations}x`, 100, y)
        doc.text(result, 130, y)
        y += 5
      }

      // Signed By
      const signatureY = pageHeight - 25
      doc.setFontSize(12)
      doc.text('Signed By', 10, signatureY)
      doc.text('Technician', 10, signatureY + 15)

      // SLIDE 2: Log Error
      doc.addPage()
      doc.setFontSize(10)
      doc.text('Log Error:', 10, 10)
      y = 20

      if (errors.length === 0) {
        doc.text('Tidak ada error yang tercatat.', 10, y)
        y += 10
      } else {
        errors.forEach((error, index) => {
          if (y > pageHeight - 20) {
            doc.addPage()
            y = 10
          }
          const textA = `${index + 1}. [${error.timestamp}] ${error.component} - ${error.type} ${error.code}`
          const textB = `Description: ${error.description}`
          const truncatedTextA = this.truncateText(textA, maxTextWidth, doc)
          const truncatedTextB = this.truncateText(textB, maxTextWidth, doc)
          doc.text(truncatedTextA, 10, y)
          y += 10
          doc.text(truncatedTextB, 10, y)
          y += 10
        })
      }

      // Grafik untuk semua motor DC
      const motorGroup = this.motorDcStatus.map((motor, index) => ({ motor, index }))
      if (motorGroup.some(({ motor }) => motor.totalTimeHistory.length > 0)) {
        if (y > pageHeight - 120) {
          doc.addPage()
          y = 10
        }
        doc.setFontSize(12)
        doc.text('Elapsed Time per Iteration - All Motors (R01-R06, C01-C08)', 10, y)
        y += 10

        const imgData = await this.createGroupChart(
          motorGroup,
          'Elapsed Time per Iteration - All Motors',
        )
        const imgWidth = 180
        const imgHeight = 120
        doc.addImage(imgData, 'PNG', 10, y, imgWidth, imgHeight)
        y += imgHeight + 10
      }

      // Grafik untuk Stepper
      const stepper = [{ motor: this.stepperStatus, index: null }]
      if (stepper[0].motor.totalTimeHistory.length > 0) {
        if (y > pageHeight - 120) {
          doc.addPage()
          y = 10
        }
        doc.setFontSize(12)
        doc.text('Elapsed Time per Iteration - Stepper Motor', 10, y)
        y += 10

        const imgData = await this.createGroupChart(
          stepper,
          'Elapsed Time per Iteration - Stepper Motor',
        )
        const imgWidth = 180
        const imgHeight = 120
        doc.addImage(imgData, 'PNG', 10, y, imgWidth, imgHeight)
        y += imgHeight + 10
      }

      // Simpan PDF
      doc.save('diagnostic_and_error_log.pdf')
    },
    clearErrors() {
      localStorage.removeItem('errors')
      alert('Data error telah dihapus')
    },
  },
}
</script>
