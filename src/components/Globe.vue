<template>
  <div ref="globeContainer" class="globe"></div>
</template>

<script setup>
import { onMounted, ref } from 'vue'
import Globe from 'globe.gl'

const globeContainer = ref(null)

onMounted(() => {
  const globe = Globe()(globeContainer.value)
    .globeImageUrl('//unpkg.com/three-globe/example/img/earth-night.jpg')
    .backgroundImageUrl('//unpkg.com/three-globe/example/img/night-sky.png')

  globe.controls().autoRotate = true
  globe.controls().autoRotateSpeed = 0.5
})
</script>

<style scoped>
.globe {
  width: 100%;
  height: 100vh;
}
</style>
