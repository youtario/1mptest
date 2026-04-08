<template>
  <div ref="globeEl" class="globe"></div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import Globe from 'globe.gl'
import * as d3 from 'd3'
import * as topojson from 'topojson-client'
import universitiesByCountry from '../data/universitiesByCountry'

const globeEl = ref(null)

let globe = null
let hoverD = null

onMounted(async () => {

  const world = await fetch(
    'https://unpkg.com/world-atlas@2/countries-110m.json'
  ).then(res => res.json())

  const countries = topojson.feature(
    world,
    world.objects.countries
  )

  const colorScale = d3.scaleSequentialSqrt(d3.interpolatePurples)

  countries.features.forEach(d => {
    const name = d.properties.name
    d.properties.universities =
      universitiesByCountry[name] ?? 0
  })

  colorScale.domain([
    0,
    d3.max(countries.features, d => d.properties.universities)
  ])

  globe = Globe()(globeEl.value)
    .globeImageUrl('//unpkg.com/three-globe/example/img/earth-dark.jpg')
    .backgroundColor('#000')

    .polygonsData(countries.features)

    .polygonAltitude(d =>
      d === hoverD
        ? 0.12
        : 0.02 + d.properties.universities / 50000
    )
    .polygonsTransitionDuration(250)

    .polygonCapColor(d => colorScale(d.properties.universities))
    .polygonSideColor(() => 'rgba(128, 0, 128, 0.25)')
    .polygonStrokeColor(() => '#2d004b')

    .polygonLabel(d => `
      <b>${d.properties.name}</b><br/>
      Universidades: ${d.properties.universities}
    `)

    .onPolygonHover(d => {
      hoverD = d
      globe.polygonAltitude(globe.polygonAltitude())
    })

    .onPolygonClick(d => {
      const [lng, lat] = d3.geoCentroid(d)
      globe.pointOfView({ lat, lng, altitude: 1.4 }, 1000)
    })

  globe.controls().autoRotate = true
  globe.controls().autoRotateSpeed = 0.3
})
</script>

<style scoped>
.globe {
  width: 100%;
  height: 100vh;
  cursor: grab;
}
</style>
