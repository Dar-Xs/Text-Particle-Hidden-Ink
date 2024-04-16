<script setup lang="ts">
import ParticleMask from "./components/ParticleBlockMask.vue";
</script>

<template>
  <h1>隐藏墨水 Demo</h1>
  <ParticleMask />
  <p>原理验证阶段早期 demo</p>
  <p>通过 Canvas API 实现，粒子物理模拟完全通过 CPU 串行计算</p>
  <p><strong>右侧抓手</strong>可以改变气泡宽度</p>
</template>

<style scoped>
strong {
  color: #4a7def;
}
</style>
