<template>
  <div class="epson-print">
    <div class="print-config">
      <h2>Epson Printer Configuration</h2>
      <div class="input-group">
        <label for="printer-ip">Printer IP Address:</label>
        <input
          id="printer-ip"
          v-model="printerIP"
          type="text"
          placeholder="192.168.1.100"
          class="input-field"
        />
      </div>
      <div class="input-group">
        <label for="printer-port">Port (optional):</label>
        <input
          id="printer-port"
          v-model="printerPort"
          type="text"
          placeholder="8008"
          class="input-field"
        />
      </div>
    </div>

    <div class="print-actions">
      <button @click="printTest" :disabled="isPrinting || !printerIP" class="print-btn">
        {{ isPrinting ? 'Printing...' : 'Print Test Page' }}
      </button>
      <button @click="printCustom" :disabled="isPrinting || !printerIP" class="print-btn">
        {{ isPrinting ? 'Printing...' : 'Print Custom Text' }}
      </button>
    </div>

    <div v-if="statusMessage" :class="['status-message', statusType]">
      {{ statusMessage }}
    </div>

    <div class="custom-text-area">
      <label for="custom-text">Custom Text to Print:</label>
      <textarea
        id="custom-text"
        v-model="customText"
        rows="5"
        class="textarea-field"
        placeholder="Enter text to print..."
      ></textarea>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const printerIP = ref('192.168.1.100')
const printerPort = ref('8008')
const isPrinting = ref(false)
const statusMessage = ref('')
const statusType = ref('info')
const customText = ref('Hello from Epson Printer!')

const getPrinterURL = () => {
  const port = printerPort.value || '8008'
  return `http://${printerIP.value}:${port}/cgi-bin/eposprint.cgi`
}

const buildPrintXML = (text, align = 'left', size = 1) => {
  const alignMap = {
    left: 'left',
    center: 'center',
    right: 'right'
  }
  
  const sizeMap = {
    1: '1',
    2: '2',
    3: '3',
    4: '4'
  }
  
  return `<?xml version="1.0" encoding="UTF-8"?>
<s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/">
  <s:Body>
    <epos-print xmlns="http://www.epson-pos.com/schemas/2011/03/epos-print">
      <text lang="en" />
      <text smooth="true" />
      <text align="${alignMap[align] || 'left'}" />
      <text size="${sizeMap[size] || '1'}" />
      <text>${escapeXml(text)}</text>
      <cut type="feed" />
    </epos-print>
  </s:Body>
</s:Envelope>`
}

const escapeXml = (text) => {
  return text
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&apos;')
}

const sendPrintRequest = async (xmlData) => {
  const url = getPrinterURL()
  
  try {
    const response = await fetch(url, {
      method: 'POST',
      headers: {
        'Content-Type': 'text/xml; charset=utf-8',
        'SOAPAction': '""'
      },
      body: xmlData,
      mode: 'no-cors' // Required for cross-origin requests to printer
    })
    
    // With no-cors mode, we can't read the response, but the request is sent
    return { success: true }
  } catch (error) {
    // Even with no-cors, errors might occur
    throw new Error('Failed to send print request: ' + error.message)
  }
}

const printTest = async () => {
  if (!printerIP.value) {
    showStatus('Please enter a printer IP address', 'error')
    return
  }

  isPrinting.value = true
  statusMessage.value = 'Sending print job to printer...'
  statusType.value = 'info'

  try {
    const dateTime = new Date().toLocaleString()
    const testText = `EPSON TEST PRINT

Date: ${dateTime}
Printer IP: ${printerIP.value}

This is a test print from Vue.js application.

--------------------------------`

    // Build XML with multiple text commands
    const xml = `<?xml version="1.0" encoding="UTF-8"?>
<s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/">
  <s:Body>
    <epos-print xmlns="http://www.epson-pos.com/schemas/2011/03/epos-print">
      <text lang="en" />
      <text smooth="true" />
      <text align="center" />
      <text size="2" />
      <text>EPSON TEST PRINT</text>
      <text size="1" />
      <text align="left" />
      <text></text>
      <text>Date: ${escapeXml(dateTime)}</text>
      <text>Printer IP: ${escapeXml(printerIP.value)}</text>
      <text></text>
      <text>This is a test print from Vue.js application.</text>
      <text></text>
      <text>--------------------------------</text>
      <cut type="feed" />
    </epos-print>
  </s:Body>
</s:Envelope>`

    await sendPrintRequest(xml)
    
    isPrinting.value = false
    showStatus('Print job sent successfully!', 'success')
  } catch (error) {
    isPrinting.value = false
    showStatus('Print error: ' + error.message, 'error')
  }
}

const printCustom = async () => {
  if (!printerIP.value) {
    showStatus('Please enter a printer IP address', 'error')
    return
  }

  if (!customText.value.trim()) {
    showStatus('Please enter some text to print.', 'error')
    return
  }

  isPrinting.value = true
  statusMessage.value = 'Sending print job to printer...'
  statusType.value = 'info'

  try {
    const lines = customText.value.split('\n')
    let xml = `<?xml version="1.0" encoding="UTF-8"?>
<s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/">
  <s:Body>
    <epos-print xmlns="http://www.epson-pos.com/schemas/2011/03/epos-print">
      <text lang="en" />
      <text smooth="true" />
      <text align="left" />
      <text size="1" />`

    lines.forEach(line => {
      xml += `<text>${escapeXml(line)}</text>`
    })

    xml += `
      <text></text>
      <text>--------------------------------</text>
      <cut type="feed" />
    </epos-print>
  </s:Body>
</s:Envelope>`

    await sendPrintRequest(xml)
    
    isPrinting.value = false
    showStatus('Print job sent successfully!', 'success')
  } catch (error) {
    isPrinting.value = false
    showStatus('Print error: ' + error.message, 'error')
  }
}

const showStatus = (message, type = 'info') => {
  statusMessage.value = message
  statusType.value = type
  
  // Auto-clear success messages after 5 seconds
  if (type === 'success') {
    setTimeout(() => {
      if (statusMessage.value === message) {
        statusMessage.value = ''
      }
    }, 5000)
  }
}
</script>

<style scoped>
.epson-print {
  max-width: 600px;
  margin: 2rem auto;
  padding: 2rem;
  background: #f5f5f5;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.print-config {
  margin-bottom: 2rem;
}

.print-config h2 {
  margin-top: 0;
  color: #333;
  font-size: 1.5rem;
}

.input-group {
  margin-bottom: 1rem;
}

.input-group label {
  display: block;
  margin-bottom: 0.5rem;
  font-weight: 500;
  color: #555;
}

.input-field {
  width: 100%;
  padding: 0.75rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 1rem;
  box-sizing: border-box;
}

.input-field:focus {
  outline: none;
  border-color: #42b883;
  box-shadow: 0 0 0 2px rgba(66, 184, 131, 0.2);
}

.print-actions {
  display: flex;
  gap: 1rem;
  margin-bottom: 1rem;
}

.print-btn {
  flex: 1;
  padding: 0.75rem 1.5rem;
  background: #42b883;
  color: white;
  border: none;
  border-radius: 4px;
  font-size: 1rem;
  font-weight: 500;
  cursor: pointer;
  transition: background 0.3s;
}

.print-btn:hover:not(:disabled) {
  background: #35a372;
}

.print-btn:disabled {
  background: #ccc;
  cursor: not-allowed;
}

.status-message {
  padding: 1rem;
  border-radius: 4px;
  margin-bottom: 1rem;
  font-weight: 500;
}

.status-message.success {
  background: #d4edda;
  color: #155724;
  border: 1px solid #c3e6cb;
}

.status-message.error {
  background: #f8d7da;
  color: #721c24;
  border: 1px solid #f5c6cb;
}

.status-message.info {
  background: #d1ecf1;
  color: #0c5460;
  border: 1px solid #bee5eb;
}

.custom-text-area {
  margin-top: 1rem;
}

.custom-text-area label {
  display: block;
  margin-bottom: 0.5rem;
  font-weight: 500;
  color: #555;
}

.textarea-field {
  width: 100%;
  padding: 0.75rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 1rem;
  font-family: inherit;
  box-sizing: border-box;
  resize: vertical;
}

.textarea-field:focus {
  outline: none;
  border-color: #42b883;
  box-shadow: 0 0 0 2px rgba(66, 184, 131, 0.2);
}
</style>
