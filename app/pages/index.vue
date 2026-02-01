<script setup lang="ts">
import StackBalanceGame from '~/components/StackBalanceGame.client.vue'
useHead({ title: 'Stack & Balance' })
</script>

<template>
  <main
    style="
      min-height: 100vh;
      display: flex;
      justify-content: center;
      align-items: center;
      padding: 24px;
      background: radial-gradient(1200px 800px at 50% 0%, #0B1224 0%, #05070D 55%, #000 100%);
      color: white;
    "
  >
    <ClientOnly fallback-tag="div" fallback="Loading game...">
      <StackBalanceGame />
    </ClientOnly>
  </main>
</template>
