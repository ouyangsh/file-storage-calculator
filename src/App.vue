<script setup>
import { ref } from 'vue'
import PriceCalculator from './components/PriceCalculator.vue'
import EmailRelay from './components/EmailRelay.vue'

const currentTab = ref('calculator') // 'calculator' or 'relay'
</script>

<template>
  <div class="calculator-container">
    <div class="header">
      <nav class="tabs-segmented">
        <button 
          :class="{ active: currentTab === 'calculator' }" 
          @click="currentTab = 'calculator'"
        >
          价格计算
        </button>
        <button 
          :class="{ active: currentTab === 'relay' }" 
          @click="currentTab = 'relay'"
        >
          邮箱代发
        </button>
      </nav>
    </div>

    <main class="content">
      <Transition name="slide-fade" mode="out-in">
        <PriceCalculator v-if="currentTab === 'calculator'" />
        <EmailRelay v-else />
      </Transition>
    </main>
  </div>
</template>

<style>
:root {
  font-family: 'Inter', system-ui, -apple-system, sans-serif;
  background-color: #f2f4f7;
  color: #1f2937;
  margin: 0;
  display: flex;
  justify-content: center;
  min-height: 100vh;
}

body {
  margin: 0;
  width: 100%;
}

.calculator-container {
  width: 100%;
  max-width: 480px;
  padding: 40px 20px;
  display: flex;
  flex-direction: column;
  gap: 24px;
}

.header {
  display: flex;
  justify-content: center;
  width: 100%;
}

.tabs-segmented {
  display: flex;
  background-color: #e5e7eb;
  padding: 4px;
  border-radius: 12px;
  width: 100%;
  max-width: 320px;
}

.tabs-segmented button {
  flex: 1;
  padding: 10px 16px;
  border: none;
  background: transparent;
  color: #6b7280;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  border-radius: 10px;
  transition: all 0.2s ease;
}

.tabs-segmented button.active {
  background-color: #ffffff;
  color: #2563eb;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.content {
  position: relative;
  width: 100%;
}

/* Transition */
.slide-fade-enter-active,
.slide-fade-leave-active {
  transition: all 0.3s ease-out;
}

.slide-fade-enter-from {
  opacity: 0;
  transform: translateX(20px);
}

.slide-fade-leave-to {
  opacity: 0;
  transform: translateX(-20px);
}

/* Shared Component Styles (Injecting into global for simplicity in this project) */
.card {
  background: #ffffff;
  border-radius: 24px;
  padding: 32px;
  box-shadow: 0 10px 25px -5px rgba(0, 0, 0, 0.05), 0 8px 10px -6px rgba(0, 0, 0, 0.05);
  display: flex;
  flex-direction: column;
  gap: 24px;
}

.title {
  font-size: 24px;
  font-weight: 700;
  text-align: center;
  margin: 0;
  color: #111827;
}

.input-group {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

label {
  font-size: 13px;
  font-weight: 600;
  color: #4b5563;
}
</style>
