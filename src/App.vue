<script setup>
import CalendarEvents from './components/CalendarEvents.vue'
</script>

<template>
  <main class="min-h-screen flex items-center justify-center" style="background:#076890ff;">
  <div class="w-full max-w-2xl p-6 rounded shadow-lg">
      <CalendarEvents />
    </div>
  </main>
</template>

<style scoped>
body {
  background: #076890ff;
  color: white;
}
</style>
