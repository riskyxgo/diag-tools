<template>
  <div class="flex flex-row justify-between items-center mt-5">
    <button @click="printErrors" class="bg-blue-500 text-white px-4 py-2 rounded">Download PDF</button>
    <button @click="clearErrors" class="bg-red-500 text-white px-4 py-2 rounded ml-2">Clear Errors</button>
  </div>
</template>

<script>
import jsPDF from 'jspdf';

export default {
  name: 'ErrorLogPrintout',
  props: {
    deviceId: String,
    motorDcStatus: Array,
    stepperStatus: Object,
    errorLogs: Array
  },
  methods: {
    getErrors() {
      return JSON.parse(localStorage.getItem('errors')) || [];
    },
    getMotorLabel(index) {
      const row = Math.floor(index / 8) + 1;
      const col = (index % 8) + 1;
      return `R${row.toString().padStart(2, '0')}:C${col.toString().padStart(2, '0')}`;
    },
    getFormattedDate() {
      const now = new Date();
      const dayNames = ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday'];
      return `${now.getDate().toString().padStart(2, '0')}/${(now.getMonth() + 1).toString().padStart(2, '0')}/${now.getFullYear()}: ${now.getHours().toString().padStart(2, '0')}:${now.getMinutes().toString().padStart(2, '0')}:${now.getSeconds().toString().padStart(2, '0')} (WIB) (${dayNames[now.getDay()]})`;
    },
    getResultAndCode(status, index) {
      const label = index === null ? 'Stepper' : `Motor Dc [${this.getMotorLabel(index)}]`;
      const logs = this.errorLogs.filter(log => log.message.includes(label));

      if (status.completed) {
        if (logs.length > 0 && logs.some(log => log.type === 'Warning')) {
          const warnCodes = logs.filter(log => log.type === 'Warning').map(log => log.message.match(/Warn \d\d/)[0]).join(', ');
          return { result: 'Good', code: warnCodes };
        }
        return { result: 'Good', code: '-' };
      } else {
        if (logs.length > 0 && logs.some(log => log.type === 'Error')) {
          const errorCodes = logs.filter(log => log.type === 'Error').map(log => log.message.match(/Error \d\d/)[0]).join(', ');
          return { result: 'Error', code: errorCodes };
        }
        return { result: 'Error', code: '-' };
      }
    },
    truncateText(text, maxWidth, doc) {
      let truncatedText = text;
      while (doc.getTextWidth(truncatedText) > maxWidth && truncatedText.length > 0) {
        truncatedText = truncatedText.slice(0, -1);
      }
      if (truncatedText.length < text.length) {
        truncatedText = truncatedText.slice(0, -3) + '...';
      }
      return truncatedText;
    },
    printErrors() {
      const errors = this.getErrors();
      const doc = new jsPDF();
      const pageHeight = doc.internal.pageSize.height;
      const pageWidth = doc.internal.pageSize.width;
      const maxTextWidth = pageWidth - 20;

      // SLIDE 1: Diagnostic Results
      doc.setFontSize(12);
      doc.text(`Unit ID: ${this.deviceId || 'Unknown'}`, 10, 10);
      doc.text(`Date: ${this.getFormattedDate()}`, 10, 16);

      // Motor DC Diagnostic - Segment 1 (Motor 0-15)
      doc.text('MOTOR DC DIAGNOSTIC', 10, 22);
      doc.setFontSize(10);
      doc.text('ID', 10, 27);
      doc.text('DELAY', 40, 27);
      doc.text('ELAPSED TIME', 70, 27);
      doc.text('RESULT', 100, 27);
      doc.line(10, 28, 190, 28); // Garis header
      let y = 32;

      const testedMotors = this.motorDcStatus
        .map((motor, index) => motor.receivedMessages.length > 0 ? { motor, index } : null)
        .filter(item => item !== null);
      const segment1 = testedMotors.filter(item => item.index < 16); // Motor 0-15
      const segment2 = testedMotors.filter(item => item.index >= 16); // Motor 16-31

      segment1.forEach(({ motor, index }) => {
        if (y > pageHeight - 30) {
          doc.addPage();
          y = 10;
        }
        const { result } = this.getResultAndCode(motor, index);
        const delay = motor.delay !== null ? `${motor.delay} ms` : '-';
        const totalTime = motor.totalTime !== null ? `${motor.totalTime} ms` : '-';
        doc.text(`ID:[${this.getMotorLabel(index)}]`, 10, y);
        doc.text(delay, 40, y);
        doc.text(totalTime, 70, y);
        doc.text(result, 100, y);
        y += 5;
      });

      // Motor DC Diagnostic - Segment 2 (Motor 16-31)
      if (segment2.length > 0) {
        y += 5;
        if (y > pageHeight - 30) {
          doc.addPage();
          y = 10;
        }
        doc.setFontSize(10);
        doc.text('ID', 10, y);
        doc.text('DELAY', 40, y);
        doc.text('ELAPSED TIME', 70, y);
        doc.text('RESULT', 100, y);
        doc.line(10, y + 1, 190, y + 1);
        y += 5;

        segment2.forEach(({ motor, index }) => {
          if (y > pageHeight - 30) {
            doc.addPage();
            y = 10;
          }
          const { result } = this.getResultAndCode(motor, index);
          const delay = motor.delay !== null ? `${motor.delay} ms` : '-';
          const totalTime = motor.totalTime !== null ? `${motor.totalTime} ms` : '-';
          doc.text(`ID:[${this.getMotorLabel(index)}]`, 10, y);
          doc.text(delay, 40, y);
          doc.text(totalTime, 70, y);
          doc.text(result, 100, y);
          y += 5;
        });
      }

      // Motor Stepper
      y += 5;
      if (y > pageHeight - 30) {
        doc.addPage();
        y = 10;
      }
      doc.setFontSize(12);
      doc.text('MOTOR STEPPER', 10, y);
      y += 6;
      doc.setFontSize(10);
      doc.text('ID', 10, y);
      doc.text('DELAY', 40, y);
      doc.text('ELAPSED TIME', 70, y);
      doc.text('RESULT', 100, y);
      doc.line(10, y + 1, 190, y + 1);
      y += 5;

      if (this.stepperStatus.receivedMessages.length > 0) {
        if (y > pageHeight - 30) {
          doc.addPage();
          y = 10;
        }
        const { result } = this.getResultAndCode(this.stepperStatus, null);
        const delay = this.stepperStatus.delay !== null ? `${this.stepperStatus.delay} ms` : '-';
        const totalTime = this.stepperStatus.totalTime !== null ? `${this.stepperStatus.totalTime} ms` : '-';
        doc.text('ID:01', 10, y);
        doc.text(delay, 40, y);
        doc.text(totalTime, 70, y);
        doc.text(result, 100, y);
        y += 5;
      }

      // Signed By
      const signatureY = pageHeight - 25;
      doc.setFontSize(12);
      doc.text('Signed By', 10, signatureY);
      doc.text('Technician', 10, signatureY + 15);

      // SLIDE 2: Log Error
      doc.addPage();
      doc.setFontSize(10);
      doc.text('Log Error:', 10, 10);
      y = 20;

      if (errors.length === 0) {
        doc.text('Tidak ada error yang tercatat.', 10, y);
      } else {
        errors.forEach((error, index) => {
          if (y > pageHeight - 20) {
            doc.addPage();
            y = 10;
          }
          const textA = `${index + 1}. [${error.timestamp}] ${error.component} - ${error.type} ${error.code}`;
          const textB = `Description: ${error.description}`;
          const truncatedTextA = this.truncateText(textA, maxTextWidth, doc);
          const truncatedTextB = this.truncateText(textB, maxTextWidth, doc);
          doc.text(truncatedTextA, 10, y);
          y += 10;
          doc.text(truncatedTextB, 10, y);
          y += 10;
        });
      }

      // Simpan PDF
      doc.save('diagnostic_and_error_log.pdf');
    },
    clearErrors() {
      localStorage.removeItem('errors');
      alert('Data error telah dihapus');
    }
  }
}
</script>
