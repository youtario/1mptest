<template>
  <div ref="globeEl" class="globe"></div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import Globe from 'globe.gl'
import * as d3 from 'd3'
import * as topojson from 'topojson-client'

const globeEl = ref(null)

// país atualmente em hover
let hoverD = null

onMounted(async () => {
  // 1️⃣ Carrega o TopoJSON dos países
  const world = await fetch(
    'https://unpkg.com/world-atlas@2/countries-110m.json'
  ).then(res => res.json())

  // 2️⃣ Converte TopoJSON → GeoJSON
  const countries = topojson.feature(
    world,
    world.objects.countries
  )

  // 3️⃣ Escala de cores
  const colorScale = d3.scaleSequentialSqrt(d3.interpolateYlOrRd)

  // 4️⃣ Dados de exemplo
  countries.features.forEach(d => {
    d.properties.value = Math.random() * 100
  })

  colorScale.domain([
    0,
    d3.max(countries.features, d => d.properties.value)
  ])

  // 5️⃣ Inicializa o globo
  const globe = Globe()(globeEl.value)
    .globeImageUrl('//unpkg.com/three-globe/example/img/earth-dark.jpg')
    .backgroundColor('#000')
    .polygonsData(countries.features)

    // 🔥 ALTURA DINÂMICA NO HOVER
    .polygonAltitude(d => (d === hoverD ? 0.12 : 0.06))

    .polygonCapColor(d => colorScale(d.properties.value))
    .polygonSideColor(() => 'rgba(0, 100, 0, 0.15)')
    .polygonStrokeColor(() => '#111')

    // Tooltip
    .polygonLabel(d => `
      <b>${d.properties.name ?? 'País'}</b><br/>
      Valor: ${d.properties.value.toFixed(1)}
    `)

    // Hover
    .onPolygonHover(d => {
      hoverD = d
    })

  // 6️⃣ Controles
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
