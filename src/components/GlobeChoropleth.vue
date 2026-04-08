<template>
  <!-- Painel de filtros fixo, por cima do globo -->
  <aside class="filter-panel" :class="{ collapsed: panelCollapsed }">
    <button class="collapse-btn" @click="panelCollapsed = !panelCollapsed">
      {{ panelCollapsed ? '›' : '‹' }}
    </button>

    <div class="panel-inner">
      <div class="panel-header">
        <span class="panel-logo">🌐</span>
        <h1 class="panel-title">Explorar<br><span>o Mundo</span></h1>
      </div>

      <div class="filter-section" v-for="section in filters" :key="section.id">
        <div class="section-header" @click="toggleSection(section.id)">
          <span class="section-icon">{{ section.icon }}</span>
          <span class="section-label">{{ section.label }}</span>
          <span class="section-arrow" :class="{ open: openSections[section.id] }">▾</span>
        </div>
        <div class="section-options" :class="{ open: openSections[section.id] }">
          <div
            v-for="opt in section.options"
            :key="opt.value"
            class="option-item"
            :class="{ active: activeFilters[section.id]?.includes(opt.value) }"
            @click="toggleFilter(section.id, opt.value)"
          >
            <span class="option-check">{{ activeFilters[section.id]?.includes(opt.value) ? '✓' : '' }}</span>
            <span class="option-text">{{ opt.label }}</span>
            <span class="option-count" v-if="opt.count">{{ opt.count }}</span>
          </div>
        </div>
      </div>

      <button class="clear-btn" @click="clearFilters">Limpar filtros</button>
    </div>
  </aside>

  <div class="active-badge" v-if="totalActiveFilters > 0">
    {{ totalActiveFilters }} filtro(s) ativo(s)
  </div>

  <div ref="globeEl" class="globe-wrap"></div>
</template>

<script setup>
import { ref, onMounted, reactive, computed } from 'vue'
import Globe from 'globe.gl'
import * as d3 from 'd3'
import * as topojson from 'topojson-client'
import universitiesByCountry from '../data/universitiesByCountry'

const globeEl = ref(null)
const panelCollapsed = ref(false)

let globe = null
let hoverD = null

const openSections = reactive({ emprego: true, universidade: false, idioma: false, cultura: false })
const activeFilters = reactive({ emprego: [], universidade: [], idioma: [], cultura: [] })

const filters = [
  {
    id: 'emprego', icon: '💼', label: 'Emprego',
    options: [
      { value: 'tech',       label: 'Tecnologia',   count: '4.2k' },
      { value: 'saude',      label: 'Saúde',        count: '1.8k' },
      { value: 'engenharia', label: 'Engenharia',   count: '2.1k' },
      { value: 'financas',   label: 'Finanças',     count: '900'  },
      { value: 'educacao',   label: 'Educação',     count: '3.4k' },
      { value: 'artes',      label: 'Artes & Design', count: '650' },
    ],
  },
  {
    id: 'universidade', icon: '🎓', label: 'Universidade',
    options: [
      { value: 'top100',      label: 'Top 100 Mundial' },
      { value: 'bolsas',      label: 'Oferece Bolsas'  },
      { value: 'intercambio', label: 'Intercâmbio'     },
      { value: 'publicas',    label: 'Públicas'        },
      { value: 'privadas',    label: 'Privadas'        },
      { value: 'ead',         label: 'EAD / Online'   },
    ],
  },
  {
    id: 'idioma', icon: '🗣️', label: 'Idioma',
    options: [
      { value: 'ingles',    label: 'Inglês'    },
      { value: 'espanhol',  label: 'Espanhol'  },
      { value: 'frances',   label: 'Francês'   },
      { value: 'alemao',    label: 'Alemão'    },
      { value: 'mandarin',  label: 'Mandarim'  },
      { value: 'japones',   label: 'Japonês'   },
      { value: 'portugues', label: 'Português' },
    ],
  },
  {
    id: 'cultura', icon: '🎭', label: 'Cultura',
    options: [
      { value: 'gastronomia', label: 'Gastronomia'          },
      { value: 'musica',      label: 'Música'               },
      { value: 'esportes',    label: 'Esportes'             },
      { value: 'religiao',    label: 'Diversidade Religiosa' },
      { value: 'festivais',   label: 'Festivais'            },
      { value: 'natureza',    label: 'Natureza & Aventura'  },
    ],
  },
]

// Mapeamento país → categorias de filtro
const countryMeta = {
  'United States of America': { emprego: ['tech','financas','saude','engenharia','educacao','artes'], idioma: ['ingles'], cultura: ['gastronomia','musica','esportes','festivais'], universidade: ['top100','privadas','intercambio'] },
  'United Kingdom':           { emprego: ['tech','financas','saude','educacao'], idioma: ['ingles'], cultura: ['musica','festivais','gastronomia'], universidade: ['top100','intercambio'] },
  'Canada':                   { emprego: ['tech','saude','engenharia','educacao'], idioma: ['ingles','frances'], cultura: ['natureza','esportes'], universidade: ['top100','intercambio','bolsas'] },
  'Australia':                { emprego: ['tech','saude','engenharia'], idioma: ['ingles'], cultura: ['natureza','esportes'], universidade: ['intercambio','bolsas'] },
  'Germany':                  { emprego: ['engenharia','tech','saude'], idioma: ['alemao'], cultura: ['festivais','gastronomia','musica'], universidade: ['top100','publicas','bolsas'] },
  'France':                   { emprego: ['artes','educacao','financas'], idioma: ['frances'], cultura: ['gastronomia','festivais','musica'], universidade: ['top100','bolsas','publicas'] },
  'Japan':                    { emprego: ['tech','engenharia'], idioma: ['japones'], cultura: ['gastronomia','festivais','musica'], universidade: ['top100','intercambio'] },
  'China':                    { emprego: ['tech','engenharia','financas'], idioma: ['mandarin'], cultura: ['gastronomia','festivais'], universidade: ['top100','publicas'] },
  'India':                    { emprego: ['tech','engenharia','educacao'], idioma: ['ingles'], cultura: ['festivais','gastronomia','religiao'], universidade: ['publicas','privadas'] },
  'Brazil':                   { emprego: ['engenharia','educacao','artes'], idioma: ['portugues'], cultura: ['musica','esportes','festivais','gastronomia'], universidade: ['publicas','privadas','ead'] },
  'Portugal':                 { emprego: ['educacao','artes'], idioma: ['portugues'], cultura: ['gastronomia','musica','festivais'], universidade: ['bolsas','intercambio'] },
  'Spain':                    { emprego: ['artes','educacao'], idioma: ['espanhol'], cultura: ['gastronomia','festivais','musica'], universidade: ['intercambio','bolsas'] },
  'Mexico':                   { emprego: ['engenharia','educacao'], idioma: ['espanhol'], cultura: ['gastronomia','festivais','musica'], universidade: ['publicas','privadas'] },
  'Argentina':                { emprego: ['educacao','artes'], idioma: ['espanhol'], cultura: ['esportes','gastronomia','musica'], universidade: ['publicas'] },
  'Netherlands':              { emprego: ['tech','financas','engenharia'], idioma: ['ingles'], cultura: ['festivais'], universidade: ['top100','intercambio'] },
  'Sweden':                   { emprego: ['tech','engenharia','saude'], idioma: ['ingles'], cultura: ['natureza'], universidade: ['top100','bolsas','publicas'] },
  'Norway':                   { emprego: ['engenharia','saude'], idioma: ['ingles'], cultura: ['natureza'], universidade: ['bolsas','publicas'] },
  'Switzerland':              { emprego: ['financas','saude','tech'], idioma: ['alemao','frances'], universidade: ['top100'], cultura: [] },
  'South Korea':              { emprego: ['tech','engenharia'], idioma: ['ingles'], cultura: ['musica','gastronomia'], universidade: ['top100','intercambio'] },
  'New Zealand':              { emprego: ['saude','educacao'], idioma: ['ingles'], cultura: ['natureza','esportes'], universidade: ['intercambio','bolsas'] },
  'Ireland':                  { emprego: ['tech','financas'], idioma: ['ingles'], cultura: ['musica','festivais'], universidade: ['intercambio'] },
  'Singapore':                { emprego: ['tech','financas','engenharia'], idioma: ['ingles'], universidade: ['top100'], cultura: [] },
  'Finland':                  { emprego: ['tech','educacao'], idioma: ['ingles'], cultura: ['natureza'], universidade: ['top100','bolsas','publicas'] },
  'Denmark':                  { emprego: ['tech','saude','engenharia'], idioma: ['ingles'], cultura: ['natureza'], universidade: ['bolsas','publicas'] },
  'Italy':                    { emprego: ['artes','educacao'], idioma: ['ingles'], cultura: ['gastronomia','musica','festivais'], universidade: ['top100','intercambio'] },
}

let colorScale

const totalActiveFilters = computed(() =>
  Object.values(activeFilters).reduce((acc, arr) => acc + arr.length, 0)
)

function countryMatchesFilters(name) {
  if (totalActiveFilters.value === 0) return true
  const meta = countryMeta[name]
  if (!meta) return false
  for (const [cat, selected] of Object.entries(activeFilters)) {
    if (selected.length === 0) continue
    const values = meta[cat] ?? []
    if (!selected.some(v => values.includes(v))) return false
  }
  return true
}

function refreshGlobe() {
  if (!globe) return
  globe
    .polygonCapColor(d => {
      return countryMatchesFilters(d.properties.name)
        ? colorScale(d.properties.universities)
        : 'rgba(20,5,30,0.35)'
    })
    .polygonAltitude(d => {
      if (d === hoverD) return 0.12
      return countryMatchesFilters(d.properties.name)
        ? 0.02 + d.properties.universities / 50000
        : 0.005
    })
    .polygonSideColor(d =>
      countryMatchesFilters(d.properties.name)
        ? 'rgba(128,0,128,0.25)'
        : 'rgba(20,5,30,0.1)'
    )
}

function toggleSection(id) { openSections[id] = !openSections[id] }

function toggleFilter(sectionId, value) {
  const arr = activeFilters[sectionId]
  const idx = arr.indexOf(value)
  if (idx === -1) arr.push(value)
  else arr.splice(idx, 1)
  refreshGlobe()
}

function clearFilters() {
  Object.keys(activeFilters).forEach(k => (activeFilters[k] = []))
  refreshGlobe()
}

onMounted(async () => {
  const world = await fetch('https://unpkg.com/world-atlas@2/countries-110m.json').then(r => r.json())
  const countries = topojson.feature(world, world.objects.countries)
  colorScale = d3.scaleSequentialSqrt(d3.interpolatePurples)

  countries.features.forEach(d => {
    d.properties.universities = universitiesByCountry[d.properties.name] ?? 0
  })
  colorScale.domain([0, d3.max(countries.features, d => d.properties.universities)])

  globe = Globe()(globeEl.value)
    .width(window.innerWidth)
    .height(window.innerHeight)
    .globeImageUrl('//unpkg.com/three-globe/example/img/earth-dark.jpg')
    .backgroundColor('#000011')
    .polygonsData(countries.features)
    .polygonAltitude(d => d === hoverD ? 0.12 : 0.02 + d.properties.universities / 50000)
    .polygonsTransitionDuration(300)
    .polygonCapColor(d => colorScale(d.properties.universities))
    .polygonSideColor(() => 'rgba(128,0,128,0.25)')
    .polygonStrokeColor(() => '#2d004b')
    .polygonLabel(d => `<b>${d.properties.name}</b><br/>Universidades: ${d.properties.universities}`)
    .onPolygonHover(d => { hoverD = d; globe.polygonAltitude(globe.polygonAltitude()) })
    .onPolygonClick(d => {
      const [lng, lat] = d3.geoCentroid(d)
      globe.pointOfView({ lat, lng, altitude: 1.4 }, 1000)
    })

  globe.controls().autoRotate = true
  globe.controls().autoRotateSpeed = 0.3
  globe.controls().maxDistance = globe.getGlobeRadius() * 4
  globe.controls().minDistance = globe.getGlobeRadius() * 1.1

  window.addEventListener('resize', () => {
    globe.width(window.innerWidth).height(window.innerHeight)
  })
})
</script>

<style>
@import url('https://fonts.googleapis.com/css2?family=Syne:wght@400;600;700;800&family=DM+Sans:wght@300;400;500&display=swap');

.globe-wrap {
  position: fixed;
  inset: 0;
  z-index: 0;
  cursor: grab;
}
.globe-wrap:active { cursor: grabbing; }

.filter-panel {
  position: fixed;
  top: 0;
  left: 0;
  height: 100vh;
  width: 300px;
  background: linear-gradient(160deg, rgba(13,0,16,0.93) 0%, rgba(10,0,15,0.93) 100%);
  backdrop-filter: blur(14px);
  -webkit-backdrop-filter: blur(14px);
  border-right: 1px solid rgba(180,60,255,0.18);
  display: flex;
  flex-direction: column;
  transition: transform 0.35s cubic-bezier(.4,0,.2,1);
  z-index: 100;
  font-family: 'DM Sans', sans-serif;
}

.filter-panel.collapsed {
  transform: translateX(-256px);
}

/* Botão de colapsar */
.collapse-btn {
  position: absolute;
  top: 50%;
  right: -16px;
  transform: translateY(-50%);
  width: 32px;
  height: 32px;
  border-radius: 50%;
  background: #1a0028;
  border: 1px solid rgba(180,60,255,0.45);
  color: #b43cff;
  font-size: 20px;
  line-height: 1;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 0 12px rgba(180,60,255,0.3);
  z-index: 101;
}
.collapse-btn:hover { background: #2a004a; }

/* Conteúdo interno */
.panel-inner {
  padding: 32px 20px 24px;
  overflow-y: auto;
  height: 100%;
  display: flex;
  flex-direction: column;
  gap: 6px;
  scrollbar-width: thin;
  scrollbar-color: rgba(180,60,255,0.3) transparent;
}
.panel-inner::-webkit-scrollbar { width: 4px; }
.panel-inner::-webkit-scrollbar-thumb { background: rgba(180,60,255,0.3); border-radius: 2px; }

.panel-header {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 24px;
}
.panel-logo { font-size: 26px; }
.panel-title {
  font-family: 'Syne', sans-serif;
  font-weight: 800;
  font-size: 18px;
  color: #fff;
  line-height: 1.2;
  margin: 0;
}
.panel-title span { color: #b43cff; }

/* Seções */
.filter-section {
  border: 1px solid rgba(180,60,255,0.1);
  border-radius: 10px;
  overflow: hidden;
  transition: border-color 0.2s;
}
.filter-section:hover { border-color: rgba(180,60,255,0.28); }

.section-header {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 12px 14px;
  cursor: pointer;
  user-select: none;
  background: rgba(180,60,255,0.04);
  transition: background 0.2s;
}
.section-header:hover { background: rgba(180,60,255,0.1); }

.section-icon { font-size: 15px; }
.section-label {
  flex: 1;
  font-family: 'Syne', sans-serif;
  font-weight: 600;
  font-size: 13px;
  color: #e0d0f0;
  letter-spacing: 0.03em;
}
.section-arrow {
  font-size: 12px;
  color: #7a4fa0;
  display: inline-block;
  transition: transform 0.25s;
}
.section-arrow.open { transform: rotate(180deg); }

/* Opções */
.section-options {
  max-height: 0;
  overflow: hidden;
  transition: max-height 0.3s cubic-bezier(.4,0,.2,1);
}
.section-options.open { max-height: 400px; }

.option-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 9px 14px;
  cursor: pointer;
  border-top: 1px solid rgba(180,60,255,0.06);
  transition: background 0.15s;
}
.option-item:hover { background: rgba(180,60,255,0.08); }
.option-item.active { background: rgba(180,60,255,0.14); }

.option-check {
  width: 16px;
  height: 16px;
  border: 1px solid rgba(180,60,255,0.4);
  border-radius: 4px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 11px;
  color: #b43cff;
  flex-shrink: 0;
  background: rgba(180,60,255,0.06);
}
.option-item.active .option-check {
  background: #b43cff;
  color: #fff;
  border-color: #b43cff;
}

.option-text {
  flex: 1;
  font-size: 13px;
  color: #c4b0d8;
}
.option-item.active .option-text { color: #ecddf8; }

.option-count {
  font-size: 11px;
  color: #6a4a8a;
  background: rgba(180,60,255,0.08);
  padding: 1px 6px;
  border-radius: 10px;
}

/* Botão limpar */
.clear-btn {
  margin-top: auto;
  padding: 10px;
  border: 1px solid rgba(180,60,255,0.25);
  border-radius: 8px;
  background: transparent;
  color: #8a5aaa;
  font-family: 'DM Sans', sans-serif;
  font-size: 13px;
  cursor: pointer;
  transition: all 0.2s;
}
.clear-btn:hover {
  background: rgba(180,60,255,0.1);
  color: #c490e8;
  border-color: rgba(180,60,255,0.5);
}

/* Badge */
.active-badge {
  position: fixed;
  bottom: 24px;
  right: 24px;
  background: #b43cff;
  color: #fff;
  font-family: 'Syne', sans-serif;
  font-size: 12px;
  font-weight: 600;
  padding: 6px 14px;
  border-radius: 20px;
  box-shadow: 0 0 20px rgba(180,60,255,0.5);
  z-index: 100;
  animation: fadeIn 0.2s ease;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(6px); }
  to   { opacity: 1; transform: translateY(0); }
}
</style>
