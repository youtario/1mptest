<template>
  <div ref="globeEl" class="globe"></div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import Globe from 'globe.gl'
import * as d3 from 'd3'
import * as topojson from 'topojson-client'

const globeEl = ref(null)
let hoverD = null
let globe = null

onMounted(async () => {
  // 1️⃣ TopoJSON
  const world = await fetch(
    'https://unpkg.com/world-atlas@2/countries-110m.json'
  ).then(res => res.json())

  const countries = topojson.feature(
    world,
    world.objects.countries
  )

  // 2️⃣ Escala de cores
  const colorScale = d3.scaleSequentialSqrt(d3.interpolateYlOrRd)

  countries.features.forEach(d => {
    d.properties.value = Math.random() * 100
  })

  colorScale.domain([
    0,
    d3.max(countries.features, d => d.properties.value)
  ])

  // 3️⃣ Globo
  globe = Globe()(globeEl.value)
    .globeImageUrl('//unpkg.com/three-globe/example/img/earth-dark.jpg')
    .backgroundColor('#000')
    .polygonsData(countries.features)
    .polygonCapColor(d => colorScale(d.properties.value))
    .polygonSideColor(() => 'rgba(0, 100, 0, 0.15)')
    .polygonStrokeColor(() => '#111')
    .polygonLabel(d => `
      <b>${d.properties.name ?? 'País'}</b><br/>
      Valor: ${d.properties.value.toFixed(1)}
    `)
    .polygonAltitude(d => d === hoverD ? 0.12 : 0.06)
    .polygonsTransitionDuration(200)

    // 🔥 HOVER FUNCIONAL
    .onPolygonHover(d => {
      hoverD = d
      globe.polygonAltitude(globe.polygonAltitude())
    })

  globe.controls().autoRotate = true
  globe.controls().autoRotateSpeed = 0.4
})
</script>

<style scoped>
.globe {
  width: 100%;
  height: 100vh;
}
</style>
