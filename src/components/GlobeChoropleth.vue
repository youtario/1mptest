<template>
  <!-- Navbar -->
  <header class="navbar">
    <div class="navbar_logo">
     <!--<img src="/logogb.png" alt="Global Bridge" />-->
    </div>
    <nav class="navbar_links">
      <a href="#">Meu destino</a>
      <a href="#">Agências</a>
      <a href="#">Anos</a>
      <a href="#">Contato</a>
    </nav>
    <button class="navbar_cta">
      Cadastre-se
      <span class="cta-icon">↗</span>
    </button>
  </header>

  <aside class="filter-panel" :class="{ collapsed: panelCollapsed }">
    <button class="collapse-btn" @click="panelCollapsed = !panelCollapsed">
      {{ panelCollapsed ? '›' : '‹' }}
    </button>
    <div class="panel-inner">
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

  <aside class="agencies-panel">
    <h2 class="agencies-title">Agências Disponíveis</h2>
    <div class="agencies-list">
      <div v-for="agency in filteredAgencies" :key="agency.id" class="agency-card">
        <div class="agency-header">
          <span class="agency-name">{{ agency.name }}</span>
          <span class="agency-stars">
            <span v-for="i in 5" :key="i" class="star" :class="{ filled: i <= agency.stars }">★</span>
          </span>
        </div>
        <p class="agency-desc">{{ agency.description }}</p>
        <div class="agency-footer">
          <span class="agency-location">📍 {{ agency.location }}</span>
          <button class="agency-btn">Acessar</button>
        </div>
      </div>
      <p v-if="filteredAgencies.length === 0" class="no-agencies">
        Nenhuma agência para os filtros selecionados.
      </p>
    </div>
  </aside>

  <div class="active-badge" v-if="totalActiveFilters > 0">
    {{ totalActiveFilters }} filtro(s) ativo(s)
  </div>

  <!-- Globo -->
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
      { value: 'tech',       label: 'Tecnologia',     count: '4.2k' },
      { value: 'saude',      label: 'Saúde',          count: '1.8k' },
      { value: 'engenharia', label: 'Engenharia',     count: '2.1k' },
      { value: 'financas',   label: 'Finanças',       count: '900'  },
      { value: 'educacao',   label: 'Educação',       count: '3.4k' },
      { value: 'artes',      label: 'Artes & Design', count: '650'  },
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
      { value: 'ead',         label: 'EAD / Online'    },
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
      { value: 'gastronomia', label: 'Gastronomia'           },
      { value: 'musica',      label: 'Música'                },
      { value: 'esportes',    label: 'Esportes'              },
      { value: 'religiao',    label: 'Diversidade Religiosa' },
      { value: 'festivais',   label: 'Festivais'             },
      { value: 'natureza',    label: 'Natureza & Aventura'   },
    ],
  },
]

const agencies = [
  {
    id: 1, name: 'Japan Express', stars: 5,
    description: 'Especializada em intercâmbio no Japão com suporte completo em português, visto e moradia.',
    location: 'Tokyo, Japão',
    tags: { idioma: ['japones'], universidade: ['intercambio', 'bolsas'] },
  },
  {
    id: 2, name: 'Living Japan', stars: 4,
    description: 'Programas de imersão cultural e linguística em diversas cidades japonesas.',
    location: 'Osaka, Japão',
    tags: { idioma: ['japones'], cultura: ['gastronomia', 'festivais'] },
  },
  {
    id: 3, name: 'MEXT', stars: 3,
    description: 'Bolsas governamentais japonesas para graduação e pós-graduação no exterior.',
    location: 'Kyoto, Japão',
    tags: { universidade: ['top100', 'bolsas', 'publicas'], idioma: ['japones'] },
  },
  {
    id: 4, name: 'EF Education', stars: 5,
    description: 'Cursos de idiomas em mais de 50 países com certificação internacional reconhecida.',
    location: 'Londres, Reino Unido',
    tags: { idioma: ['ingles', 'frances', 'alemao'], universidade: ['intercambio'] },
  },
  {
    id: 5, name: 'Campus France', stars: 4,
    description: 'Agência oficial francesa para estudantes internacionais com bolsas e vistos.',
    location: 'Paris, França',
    tags: { idioma: ['frances'], universidade: ['top100', 'bolsas', 'publicas'] },
  },
  {
    id: 6, name: 'DAAD Brasil', stars: 5,
    description: 'Serviço Alemão de Intercâmbio Acadêmico com bolsas para universidades alemãs.',
    location: 'Berlim, Alemanha',
    tags: { idioma: ['alemao'], universidade: ['top100', 'bolsas', 'publicas'] },
  },
  {
    id: 7, name: 'Study USA', stars: 4,
    description: 'Intercâmbio nos EUA: high school, college e programas de trabalho.',
    location: 'Nova York, EUA',
    tags: { idioma: ['ingles'], universidade: ['top100', 'intercambio', 'privadas'], emprego: ['tech', 'financas'] },
  },
  {
    id: 8, name: 'Ibero Intercâmbio', stars: 3,
    description: 'Programas na Espanha e América Latina com foco em cultura e gastronomia.',
    location: 'Madrid, Espanha',
    tags: { idioma: ['espanhol'], cultura: ['gastronomia', 'festivais', 'musica'] },
  },
]

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

const filteredAgencies = computed(() => {
  if (totalActiveFilters.value === 0) return agencies
  return agencies.filter(agency => {
    for (const [cat, selected] of Object.entries(activeFilters)) {
      if (selected.length === 0) continue
      const values = agency.tags?.[cat] ?? []
      if (!selected.some(v => values.includes(v))) return false
    }
    return true
  })
})

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
    .polygonCapColor(d =>
      countryMatchesFilters(d.properties.name)
        ? colorScale(d.properties.universities)
        : 'rgba(20,5,30,0.35)'
    )
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
    .backgroundColor('#3b1060')
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
@import url('https://fonts.googleapis.com/css2?family=Syne:wght@400;600;700;800&family=DM+Sans:wght@300;400;500&family=Montserrat:wght@400;600;700&display=swap');

/* ── Navbar ── */
.navbar {
  position: fixed;
  top: 12px;
  left: 50%;
  transform: translateX(-50%);
  width: calc(100% - 440px); /* espaço para os dois painéis */
  max-width: 900px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 20px;
  height: 52px;
  background: #ffffff;
  border-radius: 12px;
  box-shadow: 0 2px 20px rgba(0,0,0,0.15);
  font-family: 'Montserrat', sans-serif;
  z-index: 200;
}

.navbar_logo img {
  height: 32px;
  display: block;
}

.navbar_links {
  display: flex;
  gap: 28px;
}

.navbar_links a {
  text-decoration: none;
  color: #1a1a1a;
  font-size: 13px;
  font-weight: 600;
  transition: color 0.2s;
}
.navbar_links a:hover { color: #7b2d8b; }

.navbar_cta {
  display: flex;
  align-items: center;
  gap: 8px;
  background: #7b2d8b;
  color: #fff;
  border: none;
  border-radius: 8px;
  padding: 8px 16px;
  font-family: 'Montserrat', sans-serif;
  font-size: 12px;
  font-weight: 700;
  cursor: pointer;
  transition: background 0.2s;
  white-space: nowrap;
}
.navbar_cta:hover { background: #5a1f68; }

.cta-icon {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 22px;
  height: 22px;
  background: rgba(255,255,255,0.2);
  border-radius: 50%;
  font-size: 12px;
}

/* ── Globe ── */
.globe-wrap {
  position: fixed;
  inset: 0;
  z-index: 0;
  cursor: grab;
}
.globe-wrap:active { cursor: grabbing; }

/* ── Painel ESQUERDO (filtros) ── */
.filter-panel {
  position: fixed;
  top: 0;
  left: 0;
  height: 100vh;
  width: 210px;
  background: #f5f0f0;
  border-right: 1px solid #e0d8e8;
  display: flex;
  flex-direction: column;
  transition: transform 0.35s cubic-bezier(.4,0,.2,1);
  z-index: 100;
  font-family: 'DM Sans', sans-serif;
}

.filter-panel.collapsed {
  transform: translateX(-178px);
}

.collapse-btn {
  position: absolute;
  top: 50%;
  right: -15px;
  transform: translateY(-50%);
  width: 30px;
  height: 30px;
  border-radius: 50%;
  background: #fff;
  border: 1px solid #d0b8e0;
  color: #7b2d8b;
  font-size: 18px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 2px 8px rgba(0,0,0,0.12);
  z-index: 101;
}
.collapse-btn:hover { background: #f0e8f8; }

.panel-inner {
  padding: 80px 14px 20px;
  overflow-y: auto;
  height: 100%;
  display: flex;
  flex-direction: column;
  gap: 4px;
  scrollbar-width: thin;
  scrollbar-color: #d0b8e0 transparent;
}

.filter-section {
  border-radius: 6px;
  overflow: hidden;
}

.section-header {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 10px;
  cursor: pointer;
  user-select: none;
  border-radius: 6px;
  transition: background 0.2s;
}
.section-header:hover { background: rgba(123,45,139,0.08); }

.section-icon { font-size: 12px; }
.section-label {
  flex: 1;
  font-family: 'Montserrat', sans-serif;
  font-weight: 700;
  font-size: 10px;
  color: #2a002a;
  letter-spacing: 0.06em;
  text-transform: uppercase;
}
.section-arrow {
  font-size: 10px;
  color: #7b2d8b;
  transition: transform 0.25s;
}
.section-arrow.open { transform: rotate(180deg); }

.section-options {
  max-height: 0;
  overflow: hidden;
  transition: max-height 0.3s cubic-bezier(.4,0,.2,1);
}
.section-options.open { max-height: 400px; }

.option-item {
  display: flex;
  align-items: center;
  gap: 7px;
  padding: 6px 10px 6px 24px;
  cursor: pointer;
  border-radius: 5px;
  transition: background 0.15s;
  margin: 1px 0;
}
.option-item:hover { background: rgba(123,45,139,0.07); }
.option-item.active { background: rgba(123,45,139,0.12); }

.option-check {
  width: 13px;
  height: 13px;
  border: 1.5px solid #c0a0d0;
  border-radius: 3px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 9px;
  color: #7b2d8b;
  flex-shrink: 0;
  background: #fff;
}
.option-item.active .option-check {
  background: #7b2d8b;
  color: #fff;
  border-color: #7b2d8b;
}

.option-text {
  flex: 1;
  font-size: 11px;
  color: #3a2040;
  font-family: 'DM Sans', sans-serif;
}
.option-item.active .option-text { color: #1a001a; font-weight: 500; }

.option-count {
  font-size: 9px;
  color: #7b2d8b;
  background: rgba(123,45,139,0.1);
  padding: 1px 5px;
  border-radius: 8px;
  font-family: 'Montserrat', sans-serif;
  font-weight: 700;
}

.clear-btn {
  margin-top: auto;
  padding: 8px;
  border: 1px solid #c0a0d0;
  border-radius: 7px;
  background: transparent;
  color: #7b2d8b;
  font-family: 'Montserrat', sans-serif;
  font-size: 10px;
  font-weight: 700;
  cursor: pointer;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  transition: all 0.2s;
}
.clear-btn:hover { background: rgba(123,45,139,0.08); }

/* ── Painel DIREITO (agências) ── */
.agencies-panel {
  position: fixed;
  top: 0;
  right: 0;
  height: 100vh;
  width: 220px;
  background: #f5f0f0;
  border-left: 1px solid #e0d8e8;
  display: flex;
  flex-direction: column;
  z-index: 100;
}

.agencies-title {
  font-family: 'Syne', sans-serif;
  font-weight: 800;
  font-size: 13px;
  color: #2a002a;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  padding: 80px 16px 12px;
  margin: 0;
  border-bottom: 1px solid #e0d0e8;
}

.agencies-list {
  flex: 1;
  overflow-y: auto;
  padding: 10px 10px;
  display: flex;
  flex-direction: column;
  gap: 8px;
  scrollbar-width: thin;
  scrollbar-color: #d0b8e0 transparent;
}

.agency-card {
  background: #ffffff;
  border: 1px solid #e8ddf0;
  border-radius: 8px;
  padding: 10px 12px;
  display: flex;
  flex-direction: column;
  gap: 5px;
  transition: box-shadow 0.2s, border-color 0.2s;
}
.agency-card:hover {
  box-shadow: 0 3px 12px rgba(123,45,139,0.1);
  border-color: #c0a0d0;
}

.agency-header {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.agency-name {
  font-family: 'Syne', sans-serif;
  font-weight: 800;
  font-size: 12px;
  color: #1a001a;
  text-transform: uppercase;
  letter-spacing: 0.04em;
}

.agency-stars {
  display: flex;
  gap: 1px;
}
.star { font-size: 11px; color: #ddd; }
.star.filled { color: #7b2d8b; }

.agency-desc {
  font-size: 10px;
  color: #5a3a6a;
  line-height: 1.5;
  margin: 0;
}

.agency-footer {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-top: 2px;
}

.agency-location {
  font-size: 9px;
  color: #8a5a9a;
  font-family: 'DM Sans', sans-serif;
}

.agency-btn {
  background: #7b2d8b;
  color: #fff;
  border: none;
  border-radius: 5px;
  padding: 4px 10px;
  font-family: 'Montserrat', sans-serif;
  font-size: 9px;
  font-weight: 700;
  cursor: pointer;
  transition: background 0.2s;
  letter-spacing: 0.04em;
  text-transform: uppercase;
}
.agency-btn:hover { background: #5a1f68; }

.no-agencies {
  font-size: 11px;
  color: #8a5a9a;
  text-align: center;
  padding: 20px 0;
  font-family: 'DM Sans', sans-serif;
}

/* ── Badge ── */
.active-badge {
  position: fixed;
  bottom: 20px;
  left: 50%;
  transform: translateX(-50%);
  background: #7b2d8b;
  color: #fff;
  font-family: 'Montserrat', sans-serif;
  font-size: 11px;
  font-weight: 700;
  padding: 5px 14px;
  border-radius: 20px;
  box-shadow: 0 4px 16px rgba(123,45,139,0.35);
  z-index: 150;
  animation: fadeIn 0.2s ease;
  letter-spacing: 0.03em;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateX(-50%) translateY(6px); }
  to   { opacity: 1; transform: translateX(-50%) translateY(0); }
}
</style>
