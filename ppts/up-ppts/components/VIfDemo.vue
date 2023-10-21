<template>
  <div>
    <h1 v-if="showA" @click="handleA">Hide A</h1>
    <h1 v-else @click="handleA">Show A</h1>
  </div>
</template>

<script setup lang="ts">
import { ref, Ref } from 'vue'

const showA: Ref<boolean> = ref(true)

const handleA = () => {
  showA.value = !showA.value
}
</script>

<style lang="scss" scoped>
</style>
