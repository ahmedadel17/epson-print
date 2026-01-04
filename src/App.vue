<script setup>
import EpsonPrint from './components/EpsonPrint.vue'
</script>

<template>
  <div class="app">
    <h1>Epson Printer Application</h1>
    <EpsonPrint />
  </div>
</template>

<style scoped>
.app {
  min-height: 100vh;
  padding: 2rem;
}

.app h1 {
  text-align: center;
  color: #333;
  margin-bottom: 2rem;
}
</style>
