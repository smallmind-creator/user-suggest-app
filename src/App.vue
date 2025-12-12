<script setup lang="ts">
import MainPage from './MainPage.vue'
</script>

<template>
  <MainPage />
</template>

<!-- Глобальный сброс стилей -->
<style>
html,
body {
  margin: 0;
  padding: 0;
  width: 100%;
  height: 100%;
  overflow: hidden;
}

*,
*::before,
*::after {
  box-sizing: border-box;
}
</style>
