<template>
  <!-- Navbar -->
  <header class="navbar">
    <div class="navbar_logo">
      <!-- <img src="/logogb.png" alt="Global Bridge" /> -->
      <span class="navbar_brand">GlobalBridge</span>
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

  <!-- Filter Panel (left) -->
  <aside class="filter-panel" :class="{ collapsed: panelCollapsed }">
    <button class="collapse-btn" @click="panelCollapsed = !panelCollapsed">
      <svg width="16" height="16" viewBox="0 0 16 16" fill="none">
        <path :d="panelCollapsed ? 'M6 3l5 5-5 5' : 'M10 3L5 8l5 5'" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"/>
      </svg>
    </button>
    <div class="panel-inner">
      <!-- Botão voltar ao globo — aparece só quando o mapa está aberto -->
      <button v-if="mapMode" class="back-to-globe-btn" @click="backToGlobe">
        <svg width="14" height="14" viewBox="0 0 14 14" fill="none">
          <path d="M9 2L4 7l5 5" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
        </svg>
        Voltar ao Globo
      </button>

      <div class="panel-heading">
        <span class="panel-label">Filtros</span>
        <span class="panel-line"></span>
      </div>
      <div class="filter-section" v-for="section in filters" :key="section.id">
        <div class="section-header" @click="toggleSection(section.id)">
          <span class="section-label">{{ section.label }}</span>
          <svg class="section-arrow" :class="{ open: openSections[section.id] }" width="12" height="12" viewBox="0 0 12 12" fill="none">
            <path d="M2 4l4 4 4-4" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"/>
          </svg>
        </div>
        <div class="section-options" :class="{ open: openSections[section.id] }">
          <div v-for="opt in section.options" :key="opt.value" class="option-item"
            :class="{ active: activeFilters[section.id]?.includes(opt.value) }"
            @click="toggleFilter(section.id, opt.value)">
            <span class="option-check">
              <svg v-if="activeFilters[section.id]?.includes(opt.value)" width="8" height="8" viewBox="0 0 8 8" fill="none">
                <path d="M1 4l2.5 2.5L7 1.5" stroke="white" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
              </svg>
            </span>
            <span class="option-text">{{ opt.label }}</span>
            <span class="option-count" v-if="opt.count">{{ opt.count }}</span>
          </div>
        </div>
      </div>
      <button class="clear-btn" @click="clearFilters"><span>Limpar filtros</span></button>
    </div>
  </aside>

  <!-- Agencies Panel (right) -->
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
      <p v-if="filteredAgencies.length === 0" class="no-agencies">Nenhuma agência para os filtros selecionados.</p>
    </div>
  </aside>

  <!-- Active filters badge -->
  <div class="active-badge" v-if="totalActiveFilters > 0">
    {{ totalActiveFilters }} filtro(s) ativo(s)
  </div>

  <!-- ═══════════════════════════════════════════════
       GLOBE VIEW
  ══════════════════════════════════════════════════ -->
  <div ref="globeEl" class="globe-wrap" :class="{ 'globe-hidden': mapMode }"></div>

  <!-- ═══════════════════════════════════════════════
       LEAFLET MAP VIEW
  ══════════════════════════════════════════════════ -->
  <Transition name="map-slide">
    <div v-if="mapMode" class="map-overlay">

      <!-- Floating back button — always visible above panels -->
      <button class="back-btn-float" @click="backToGlobe">
        <svg width="16" height="16" viewBox="0 0 16 16" fill="none">
          <path d="M10 3L5 8l5 5" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round"/>
        </svg>
        <span>Voltar ao Globo</span>
      </button>

      <!-- Country info chip -->
      <div class="map-country-chip">
        <span class="map-country-flag">{{ selectedCountryData?.flag }}</span>
        <div class="map-country-text">
          <h2 class="map-country-name">{{ selectedCountryName }}</h2>
          <p class="map-country-stats">
            {{ filteredMapUniversities.length }} universidade(s)
            <span v-if="totalActiveFilters > 0"> · {{ totalActiveFilters }} filtro(s)</span>
          </p>
        </div>
      </div>

      <!-- Legend chip -->
      <div class="map-legend-chip">
        <span class="legend-dot legend-top100"></span> Top 100
        <span class="legend-dot legend-bolsas"></span> Bolsas
        <span class="legend-dot legend-intercambio"></span> Intercâmbio
        <span class="legend-dot legend-default"></span> Outras
      </div>

      <!-- Leaflet map container -->
      <div ref="leafletEl" class="leaflet-container"></div>

      <!-- University info panel (appears when marker clicked) -->
      <Transition name="info-slide">
        <div v-if="selectedUniversity" class="uni-info-panel">
          <button class="uni-close" @click="selectedUniversity = null">✕</button>
          <div class="uni-type-badge" :class="getUniBadgeClass(selectedUniversity)">
            {{ getUniTypeLabel(selectedUniversity) }}
          </div>
          <h3 class="uni-info-name">{{ selectedUniversity.name }}</h3>
          <p class="uni-info-location">📍 {{ selectedUniversity.city }}</p>
          <div class="uni-info-tags">
            <span v-for="tag in selectedUniversity.tags" :key="tag" class="uni-tag">{{ tag }}</span>
          </div>
          <div class="uni-info-stats">
            <div class="uni-stat">
              <span class="uni-stat-value">{{ selectedUniversity.ranking || '—' }}</span>
              <span class="uni-stat-label">Ranking mundial</span>
            </div>
            <div class="uni-stat">
              <span class="uni-stat-value">{{ selectedUniversity.students || '—' }}</span>
              <span class="uni-stat-label">Estudantes</span>
            </div>
            <div class="uni-stat">
              <span class="uni-stat-value">{{ selectedUniversity.founded || '—' }}</span>
              <span class="uni-stat-label">Fundação</span>
            </div>
          </div>
          <a :href="selectedUniversity.website || '#'" target="_blank" class="uni-visit-btn">
            Ver Universidade ↗
          </a>
        </div>
      </Transition>
    </div>
  </Transition>
</template>

<script setup>
import { ref, onMounted, onUnmounted, reactive, computed, watch, nextTick } from 'vue'
import Globe from 'globe.gl'
import * as d3 from 'd3'
import * as topojson from 'topojson-client'

// ─── Leaflet: import dinâmico para evitar SSR issues ───────────────────────
let L = null

// ─── Refs ──────────────────────────────────────────────────────────────────
const globeEl   = ref(null)
const leafletEl = ref(null)

const panelCollapsed    = ref(false)
const mapMode           = ref(false)
const selectedCountryName = ref('')
const selectedCountryData = ref(null)
const selectedUniversity  = ref(null)

let globe      = null
let leafletMap = null
let hoverD     = null

// ─── Filter state ───────────────────────────────────────────────────────────
const openSections  = reactive({ emprego: true, universidade: false, idioma: false, cultura: false })
const activeFilters = reactive({ emprego: [], universidade: [], idioma: [], cultura: [] })

const filters = [
  {
    id: 'emprego', label: 'Emprego',
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
    id: 'universidade', label: 'Universidade',
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
    id: 'idioma', label: 'Idioma',
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
    id: 'cultura', label: 'Cultura',
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

// ─── Agencies data ──────────────────────────────────────────────────────────
const agencies = [
  { id: 1, name: 'Japan Express', stars: 5, description: 'Especializada em intercâmbio no Japão com suporte completo em português, visto e moradia.', location: 'Tokyo, Japão', tags: { idioma: ['japones'], universidade: ['intercambio', 'bolsas'] } },
  { id: 2, name: 'Living Japan', stars: 4, description: 'Programas de imersão cultural e linguística em diversas cidades japonesas.', location: 'Osaka, Japão', tags: { idioma: ['japones'], cultura: ['gastronomia', 'festivais'] } },
  { id: 3, name: 'MEXT', stars: 3, description: 'Bolsas governamentais japonesas para graduação e pós-graduação no exterior.', location: 'Kyoto, Japão', tags: { universidade: ['top100', 'bolsas', 'publicas'], idioma: ['japones'] } },
  { id: 4, name: 'EF Education', stars: 5, description: 'Cursos de idiomas em mais de 50 países com certificação internacional reconhecida.', location: 'Londres, Reino Unido', tags: { idioma: ['ingles', 'frances', 'alemao'], universidade: ['intercambio'] } },
  { id: 5, name: 'Campus France', stars: 4, description: 'Agência oficial francesa para estudantes internacionais com bolsas e vistos.', location: 'Paris, França', tags: { idioma: ['frances'], universidade: ['top100', 'bolsas', 'publicas'] } },
  { id: 6, name: 'DAAD Brasil', stars: 5, description: 'Serviço Alemão de Intercâmbio Acadêmico com bolsas para universidades alemãs.', location: 'Berlim, Alemanha', tags: { idioma: ['alemao'], universidade: ['top100', 'bolsas', 'publicas'] } },
  { id: 7, name: 'Study USA', stars: 4, description: 'Intercâmbio nos EUA: high school, college e programas de trabalho.', location: 'Nova York, EUA', tags: { idioma: ['ingles'], universidade: ['top100', 'intercambio', 'privadas'], emprego: ['tech', 'financas'] } },
  { id: 8, name: 'Ibero Intercâmbio', stars: 3, description: 'Programas na Espanha e América Latina com foco em cultura e gastronomia.', location: 'Madrid, Espanha', tags: { idioma: ['espanhol'], cultura: ['gastronomia', 'festivais', 'musica'] } },
]

// ─── Country metadata ────────────────────────────────────────────────────────
const countryMeta = {
  'United States of America': { emprego: ['tech','financas','saude','engenharia','educacao','artes'], idioma: ['ingles'], cultura: ['gastronomia','musica','esportes','festivais'], universidade: ['top100','privadas','intercambio'], flag: '🇺🇸', center: [38, -97], zoom: 4 },
  'United Kingdom':           { emprego: ['tech','financas','saude','educacao'], idioma: ['ingles'], cultura: ['musica','festivais','gastronomia'], universidade: ['top100','intercambio'], flag: '🇬🇧', center: [54, -2], zoom: 6 },
  'Canada':                   { emprego: ['tech','saude','engenharia','educacao'], idioma: ['ingles','frances'], cultura: ['natureza','esportes'], universidade: ['top100','intercambio','bolsas'], flag: '🇨🇦', center: [56, -96], zoom: 4 },
  'Australia':                { emprego: ['tech','saude','engenharia'], idioma: ['ingles'], cultura: ['natureza','esportes'], universidade: ['intercambio','bolsas'], flag: '🇦🇺', center: [-25, 133], zoom: 4 },
  'Germany':                  { emprego: ['engenharia','tech','saude'], idioma: ['alemao'], cultura: ['festivais','gastronomia','musica'], universidade: ['top100','publicas','bolsas'], flag: '🇩🇪', center: [51, 10], zoom: 6 },
  'France':                   { emprego: ['artes','educacao','financas'], idioma: ['frances'], cultura: ['gastronomia','festivais','musica'], universidade: ['top100','bolsas','publicas'], flag: '🇫🇷', center: [46, 2], zoom: 6 },
  'Japan':                    { emprego: ['tech','engenharia'], idioma: ['japones'], cultura: ['gastronomia','festivais','musica'], universidade: ['top100','intercambio'], flag: '🇯🇵', center: [36, 138], zoom: 5 },
  'China':                    { emprego: ['tech','engenharia','financas'], idioma: ['mandarin'], cultura: ['gastronomia','festivais'], universidade: ['top100','publicas'], flag: '🇨🇳', center: [35, 105], zoom: 4 },
  'India':                    { emprego: ['tech','engenharia','educacao'], idioma: ['ingles'], cultura: ['festivais','gastronomia','religiao'], universidade: ['publicas','privadas'], flag: '🇮🇳', center: [20, 77], zoom: 5 },
  'Brazil':                   { emprego: ['engenharia','educacao','artes'], idioma: ['portugues'], cultura: ['musica','esportes','festivais','gastronomia'], universidade: ['publicas','privadas','ead'], flag: '🇧🇷', center: [-14, -51], zoom: 4 },
  'Portugal':                 { emprego: ['educacao','artes'], idioma: ['portugues'], cultura: ['gastronomia','musica','festivais'], universidade: ['bolsas','intercambio'], flag: '🇵🇹', center: [39.5, -8], zoom: 7 },
  'Spain':                    { emprego: ['artes','educacao'], idioma: ['espanhol'], cultura: ['gastronomia','festivais','musica'], universidade: ['intercambio','bolsas'], flag: '🇪🇸', center: [40, -3.7], zoom: 6 },
  'Mexico':                   { emprego: ['engenharia','educacao'], idioma: ['espanhol'], cultura: ['gastronomia','festivais','musica'], universidade: ['publicas','privadas'], flag: '🇲🇽', center: [23, -102], zoom: 5 },
  'Argentina':                { emprego: ['educacao','artes'], idioma: ['espanhol'], cultura: ['esportes','gastronomia','musica'], universidade: ['publicas'], flag: '🇦🇷', center: [-38, -63], zoom: 4 },
  'Netherlands':              { emprego: ['tech','financas','engenharia'], idioma: ['ingles'], cultura: ['festivais'], universidade: ['top100','intercambio'], flag: '🇳🇱', center: [52.3, 5.3], zoom: 7 },
  'Sweden':                   { emprego: ['tech','engenharia','saude'], idioma: ['ingles'], cultura: ['natureza'], universidade: ['top100','bolsas','publicas'], flag: '🇸🇪', center: [62, 17], zoom: 5 },
  'Norway':                   { emprego: ['engenharia','saude'], idioma: ['ingles'], cultura: ['natureza'], universidade: ['bolsas','publicas'], flag: '🇳🇴', center: [64, 13], zoom: 5 },
  'Switzerland':              { emprego: ['financas','saude','tech'], idioma: ['alemao','frances'], universidade: ['top100'], cultura: [], flag: '🇨🇭', center: [46.8, 8.2], zoom: 7 },
  'South Korea':              { emprego: ['tech','engenharia'], idioma: ['ingles'], cultura: ['musica','gastronomia'], universidade: ['top100','intercambio'], flag: '🇰🇷', center: [36, 128], zoom: 7 },
  'New Zealand':              { emprego: ['saude','educacao'], idioma: ['ingles'], cultura: ['natureza','esportes'], universidade: ['intercambio','bolsas'], flag: '🇳🇿', center: [-41, 174], zoom: 6 },
  'Ireland':                  { emprego: ['tech','financas'], idioma: ['ingles'], cultura: ['musica','festivais'], universidade: ['intercambio'], flag: '🇮🇪', center: [53, -8], zoom: 7 },
  'Singapore':                { emprego: ['tech','financas','engenharia'], idioma: ['ingles'], universidade: ['top100'], cultura: [], flag: '🇸🇬', center: [1.35, 103.8], zoom: 11 },
  'Finland':                  { emprego: ['tech','educacao'], idioma: ['ingles'], cultura: ['natureza'], universidade: ['top100','bolsas','publicas'], flag: '🇫🇮', center: [64, 26], zoom: 5 },
  'Denmark':                  { emprego: ['tech','saude','engenharia'], idioma: ['ingles'], cultura: ['natureza'], universidade: ['bolsas','publicas'], flag: '🇩🇰', center: [56, 10], zoom: 7 },
  'Italy':                    { emprego: ['artes','educacao'], idioma: ['ingles'], cultura: ['gastronomia','musica','festivais'], universidade: ['top100','intercambio'], flag: '🇮🇹', center: [42, 12.5], zoom: 6 },
}

// ─── Universities dataset per country ──────────────────────────────────────
const universitiesData = {
  'Japan': [
    { id: 1,  name: 'University of Tokyo',    city: 'Tokyo',    lat: 35.7128, lng: 139.7613, tags: ['Top 100','Pesquisa','Intercâmbio'], type: 'top100',      ranking: '#23',  students: '28,000', founded: '1877', website: 'https://www.u-tokyo.ac.jp' },
    { id: 2,  name: 'Kyoto University',        city: 'Kyoto',    lat: 35.0262, lng: 135.7813, tags: ['Top 100','Pesquisa','Bolsas'],      type: 'top100',      ranking: '#46',  students: '22,000', founded: '1897' },
    { id: 3,  name: 'Osaka University',        city: 'Osaka',    lat: 34.8215, lng: 135.5242, tags: ['Top 100','Ciências'],               type: 'top100',      ranking: '#68',  students: '24,000', founded: '1931' },
    { id: 4,  name: 'Tohoku University',       city: 'Sendai',   lat: 38.2573, lng: 140.8468, tags: ['Top 100','Pesquisa'],               type: 'top100',      ranking: '#79',  students: '18,000', founded: '1907' },
    { id: 5,  name: 'Tokyo Institute of Tech', city: 'Tokyo',    lat: 35.6052, lng: 139.6859, tags: ['Top 100','Engenharia'],             type: 'top100',      ranking: '#91',  students: '10,000', founded: '1881' },
    { id: 6,  name: 'Nagoya University',       city: 'Nagoya',   lat: 35.1556, lng: 136.9678, tags: ['Pública','Engenharia'],             type: 'default',     ranking: '#111', students: '16,000', founded: '1939' },
    { id: 7,  name: 'Hokkaido University',     city: 'Sapporo',  lat: 43.0769, lng: 141.3458, tags: ['Pública','Bolsas','Natureza'],      type: 'bolsas',      ranking: '#118', students: '18,000', founded: '1876' },
    { id: 8,  name: 'Waseda University',       city: 'Tokyo',    lat: 35.7090, lng: 139.7195, tags: ['Intercâmbio','Privada'],            type: 'intercambio', ranking: '#201', students: '43,000', founded: '1882' },
    { id: 9,  name: 'Keio University',         city: 'Tokyo',    lat: 35.6483, lng: 139.7035, tags: ['Intercâmbio','Privada','Bolsas'],   type: 'bolsas',      ranking: '#195', students: '33,000', founded: '1858' },
    { id: 10, name: 'Kyushu University',       city: 'Fukuoka',  lat: 33.6264, lng: 130.4286, tags: ['Pública','Pesquisa'],               type: 'default',     ranking: '#135', students: '20,000', founded: '1911' },
    { id: 11, name: 'Kobe University',         city: 'Kobe',     lat: 34.7281, lng: 135.2360, tags: ['Pública','Intercâmbio'],            type: 'intercambio', ranking: '#251', students: '16,000', founded: '1902' },
    { id: 12, name: 'Hiroshima University',    city: 'Hiroshima',lat: 34.4017, lng: 132.7087, tags: ['Pública','Bolsas'],                 type: 'bolsas',      ranking: '#301', students: '14,000', founded: '1929' },
    { id: 13, name: 'Sophia University',       city: 'Tokyo',    lat: 35.6877, lng: 139.7312, tags: ['Privada','Intercâmbio'],            type: 'intercambio', ranking: '#401', students: '13,000', founded: '1913' },
    { id: 14, name: 'Ritsumeikan University',  city: 'Kyoto',    lat: 35.0299, lng: 135.7234, tags: ['Privada','Bolsas'],                 type: 'bolsas',      ranking: '#501', students: '35,000', founded: '1900' },
    { id: 15, name: 'Tsukuba University',      city: 'Tsukuba',  lat: 36.1063, lng: 140.1013, tags: ['Pública','Pesquisa','Intercâmbio'], type: 'intercambio', ranking: '#175', students: '17,000', founded: '1973' },
  ],
  'Germany': [
    { id: 1,  name: 'LMU München',              city: 'Munique',   lat: 48.1510, lng: 11.5800, tags: ['Top 100','Pública','Bolsas'],    type: 'top100',      ranking: '#54',  students: '50,000', founded: '1472' },
    { id: 2,  name: 'TU München',               city: 'Munique',   lat: 48.1474, lng: 11.5679, tags: ['Top 100','Engenharia','Bolsas'], type: 'top100',      ranking: '#37',  students: '48,000', founded: '1868' },
    { id: 3,  name: 'Heidelberg University',    city: 'Heidelberg',lat: 49.4097, lng:  8.6992, tags: ['Top 100','Pública'],            type: 'top100',      ranking: '#65',  students: '29,000', founded: '1386' },
    { id: 4,  name: 'Humboldt-Universität',     city: 'Berlim',    lat: 52.5190, lng: 13.3915, tags: ['Top 100','Pública'],            type: 'top100',      ranking: '#87',  students: '35,000', founded: '1810' },
    { id: 5,  name: 'FU Berlin',                city: 'Berlim',    lat: 52.4540, lng: 13.2957, tags: ['Intercâmbio','Pública'],        type: 'intercambio', ranking: '#98',  students: '33,000', founded: '1948' },
    { id: 6,  name: 'RWTH Aachen',              city: 'Aachen',    lat: 50.7793, lng:  6.0595, tags: ['Engenharia','Top 100'],         type: 'top100',      ranking: '#106', students: '45,000', founded: '1870' },
    { id: 7,  name: 'KIT Karlsruhe',            city: 'Karlsruhe', lat: 49.0130, lng:  8.4093, tags: ['Engenharia','Tech','Bolsas'],   type: 'bolsas',      ranking: '#119', students: '25,000', founded: '1825' },
    { id: 8,  name: 'TU Berlin',                city: 'Berlim',    lat: 52.5122, lng: 13.3270, tags: ['Engenharia','Intercâmbio'],     type: 'intercambio', ranking: '#154', students: '35,000', founded: '1879' },
    { id: 9,  name: 'Universität Hamburg',      city: 'Hamburgo',  lat: 53.5680, lng:  9.9747, tags: ['Pública','Pesquisa'],           type: 'default',     ranking: '#201', students: '40,000', founded: '1919' },
    { id: 10, name: 'Universität Frankfurt',    city: 'Frankfurt', lat: 50.1282, lng:  8.6656, tags: ['Pública','Bolsas'],             type: 'bolsas',      ranking: '#190', students: '46,000', founded: '1914' },
    { id: 11, name: 'Universität Freiburg',     city: 'Freiburg',  lat: 47.9960, lng:  7.8495, tags: ['Pública','Intercâmbio'],        type: 'intercambio', ranking: '#232', students: '25,000', founded: '1457' },
    { id: 12, name: 'Universität Tübingen',     city: 'Tübingen',  lat: 48.5216, lng:  9.0576, tags: ['Pública','Pesquisa'],           type: 'default',     ranking: '#210', students: '28,000', founded: '1477' },
    { id: 13, name: 'Universität Bonn',         city: 'Bonn',      lat: 50.7275, lng:  7.0849, tags: ['Pública','Bolsas'],             type: 'bolsas',      ranking: '#177', students: '35,000', founded: '1818' },
    { id: 14, name: 'TU Dresden',               city: 'Dresden',   lat: 51.0263, lng: 13.7226, tags: ['Engenharia','Pública'],         type: 'default',     ranking: '#245', students: '32,000', founded: '1828' },
    { id: 15, name: 'Universität Köln',         city: 'Colônia',   lat: 50.9283, lng:  6.9291, tags: ['Pública','Intercâmbio'],        type: 'intercambio', ranking: '#260', students: '50,000', founded: '1388' },
  ],
  'United Kingdom': [
    { id: 1,  name: 'University of Oxford',    city: 'Oxford',     lat: 51.7548, lng: -1.2544, tags: ['Top 100','Bolsas','Intercâmbio'], type: 'top100',      ranking: '#1',  students: '24,000', founded: '1096' },
    { id: 2,  name: 'University of Cambridge', city: 'Cambridge',  lat: 52.2043, lng:  0.1149, tags: ['Top 100','Pesquisa'],             type: 'top100',      ranking: '#2',  students: '22,000', founded: '1209' },
    { id: 3,  name: 'Imperial College London', city: 'Londres',    lat: 51.4988, lng: -0.1749, tags: ['Top 100','Tecnologia'],           type: 'top100',      ranking: '#6',  students: '17,000', founded: '1907' },
    { id: 4,  name: 'UCL',                     city: 'Londres',    lat: 51.5246, lng: -0.1340, tags: ['Top 100','Intercâmbio'],          type: 'top100',      ranking: '#9',  students: '42,000', founded: '1826' },
    { id: 5,  name: 'University of Edinburgh', city: 'Edimburgo',  lat: 55.9445, lng: -3.1892, tags: ['Top 100','Bolsas'],               type: 'top100',      ranking: '#22', students: '35,000', founded: '1583' },
    { id: 6,  name: 'University of Manchester',city: 'Manchester', lat: 53.4668, lng: -2.2339, tags: ['Top 100','Pesquisa'],             type: 'top100',      ranking: '#28', students: '40,000', founded: '1824' },
    { id: 7,  name: "King's College London",   city: 'Londres',    lat: 51.5115, lng: -0.1160, tags: ['Intercâmbio','Bolsas'],           type: 'bolsas',      ranking: '#40', students: '31,000', founded: '1829' },
    { id: 8,  name: 'LSE',                     city: 'Londres',    lat: 51.5144, lng: -0.1164, tags: ['Top 100','Ciências Sociais'],     type: 'top100',      ranking: '#45', students: '12,000', founded: '1895' },
    { id: 9,  name: 'University of Bristol',   city: 'Bristol',    lat: 51.4584, lng: -2.6032, tags: ['Pública','Intercâmbio'],          type: 'intercambio', ranking: '#55', students: '28,000', founded: '1909' },
    { id: 10, name: 'University of Warwick',   city: 'Coventry',   lat: 52.3837, lng: -1.5607, tags: ['Pública','Bolsas'],               type: 'bolsas',      ranking: '#67', students: '29,000', founded: '1965' },
    { id: 11, name: 'University of Glasgow',   city: 'Glasgow',    lat: 55.8721, lng: -4.2885, tags: ['Pública','Intercâmbio'],          type: 'intercambio', ranking: '#73', students: '32,000', founded: '1451' },
    { id: 12, name: 'University of Leeds',     city: 'Leeds',      lat: 53.8063, lng: -1.5550, tags: ['Pública','Bolsas'],               type: 'bolsas',      ranking: '#92', students: '38,000', founded: '1904' },
    { id: 13, name: 'University of Sheffield', city: 'Sheffield',  lat: 53.3811, lng: -1.4873, tags: ['Pública','Pesquisa'],             type: 'default',     ranking: '#95', students: '29,000', founded: '1905' },
    { id: 14, name: 'University of Nottingham',city: 'Nottingham', lat: 52.9378, lng: -1.1981, tags: ['Pública','Intercâmbio'],          type: 'intercambio', ranking: '#99', students: '33,000', founded: '1881' },
    { id: 15, name: 'Durham University',       city: 'Durham',     lat: 54.7751, lng: -1.5775, tags: ['Pública','Top 100'],              type: 'top100',      ranking: '#78', students: '20,000', founded: '1832' },
  ],
  'France': [
    { id: 1,  name: 'École Polytechnique',       city: 'Paris',    lat: 48.7145, lng:  2.2090, tags: ['Top 100','Engenharia','Bolsas'],          type: 'top100',      ranking: '#42',  students: '3,000',  founded: '1794' },
    { id: 2,  name: 'Sorbonne Université',       city: 'Paris',    lat: 48.8485, lng:  2.3443, tags: ['Top 100','Intercâmbio'],                  type: 'top100',      ranking: '#83',  students: '55,000', founded: '1257' },
    { id: 3,  name: 'ENS Paris',                 city: 'Paris',    lat: 48.8433, lng:  2.3448, tags: ['Top 100','Pesquisa'],                      type: 'top100',      ranking: '#44',  students: '2,500',  founded: '1794' },
    { id: 4,  name: 'Sciences Po',               city: 'Paris',    lat: 48.8540, lng:  2.3282, tags: ['Ciências Sociais','Bolsas','Intercâmbio'], type: 'bolsas',      ranking: '#150', students: '14,000', founded: '1872' },
    { id: 5,  name: 'Université Paris-Saclay',   city: 'Paris',    lat: 48.7085, lng:  2.1630, tags: ['Top 100','Pesquisa'],                      type: 'top100',      ranking: '#13',  students: '48,000', founded: '2019' },
    { id: 6,  name: 'HEC Paris',                 city: 'Paris',    lat: 48.7538, lng:  2.1717, tags: ['Negócios','Privada','Bolsas'],              type: 'bolsas',      ranking: '#27',  students: '4,000',  founded: '1881' },
    { id: 7,  name: 'Université de Lyon',        city: 'Lyon',     lat: 45.7640, lng:  4.8357, tags: ['Pública','Bolsas'],                        type: 'bolsas',      ranking: '#190', students: '40,000', founded: '1809' },
    { id: 8,  name: 'Université Grenoble Alpes', city: 'Grenoble', lat: 45.1885, lng:  5.7245, tags: ['Pública','Intercâmbio'],                   type: 'intercambio', ranking: '#201', students: '45,000', founded: '1339' },
    { id: 9,  name: 'Université de Bordeaux',    city: 'Bordeaux', lat: 44.8096, lng: -0.6057, tags: ['Pública','Intercâmbio'],                   type: 'intercambio', ranking: '#231', students: '52,000', founded: '1441' },
    { id: 10, name: 'Université de Strasbourg',  city: 'Estrasburgo',lat:48.5840,lng:  7.7672, tags: ['Pública','Bolsas'],                        type: 'bolsas',      ranking: '#220', students: '51,000', founded: '1538' },
    { id: 11, name: 'Université Aix-Marseille',  city: 'Marselha', lat: 43.3004, lng:  5.3922, tags: ['Pública','Pesquisa'],                      type: 'default',     ranking: '#245', students: '80,000', founded: '1409' },
    { id: 12, name: 'Université de Montpellier', city: 'Montpellier',lat:43.6117,lng:  3.8767, tags: ['Pública','Intercâmbio'],                   type: 'intercambio', ranking: '#270', students: '48,000', founded: '1220' },
  ],
  'Brazil': [
    { id: 1,  name: 'USP',           city: 'São Paulo',       lat: -23.5587, lng: -46.7317, tags: ['Top 100','Pública','Pesquisa'],        type: 'top100',      ranking: '#100', students: '90,000', founded: '1934' },
    { id: 2,  name: 'UNICAMP',       city: 'Campinas',        lat: -22.8172, lng: -47.0685, tags: ['Top 100','Pesquisa','Pública'],        type: 'top100',      ranking: '#201', students: '35,000', founded: '1966' },
    { id: 3,  name: 'UFRJ',          city: 'Rio de Janeiro',  lat: -22.8624, lng: -43.2235, tags: ['Pública','Pesquisa'],                 type: 'default',     ranking: '#300', students: '60,000', founded: '1920' },
    { id: 4,  name: 'UFMG',          city: 'Belo Horizonte',  lat: -19.8709, lng: -43.9674, tags: ['Pública','Pesquisa'],                 type: 'default',     ranking: '#400', students: '45,000', founded: '1927' },
    { id: 5,  name: 'PUC-Rio',       city: 'Rio de Janeiro',  lat: -22.9784, lng: -43.2349, tags: ['Privada','Intercâmbio','Bolsas'],     type: 'intercambio', ranking: '#500', students: '26,000', founded: '1941' },
    { id: 6,  name: 'FGV São Paulo', city: 'São Paulo',       lat: -23.5992, lng: -46.6756, tags: ['Privada','Negócios','EAD'],           type: 'default',     ranking: '#400', students: '20,000', founded: '1944' },
    { id: 7,  name: 'UNESP',         city: 'São Paulo',       lat: -23.4330, lng: -51.9380, tags: ['Pública','Pesquisa'],                 type: 'default',     ranking: '#401', students: '50,000', founded: '1976' },
    { id: 8,  name: 'UFSC',          city: 'Florianópolis',   lat: -27.5969, lng: -48.5495, tags: ['Pública','Bolsas','Intercâmbio'],     type: 'bolsas',      ranking: '#501', students: '40,000', founded: '1960' },
    { id: 9,  name: 'UFRGS',         city: 'Porto Alegre',    lat: -30.0346, lng: -51.2177, tags: ['Pública','Pesquisa'],                 type: 'default',     ranking: '#451', students: '30,000', founded: '1934' },
    { id: 10, name: 'UnB',           city: 'Brasília',        lat: -15.7626, lng: -47.8682, tags: ['Pública','Pesquisa'],                 type: 'default',     ranking: '#601', students: '40,000', founded: '1962' },
    { id: 11, name: 'UFC',           city: 'Fortaleza',       lat:  -3.7461, lng: -38.5751, tags: ['Pública','Bolsas'],                   type: 'bolsas',      ranking: '#601', students: '50,000', founded: '1954' },
    { id: 12, name: 'UFBA',          city: 'Salvador',        lat: -13.0013, lng: -38.5099, tags: ['Pública','Intercâmbio'],              type: 'intercambio', ranking: '#701', students: '40,000', founded: '1808' },
    { id: 13, name: 'PUC-SP',        city: 'São Paulo',       lat: -23.5279, lng: -46.6531, tags: ['Privada','Bolsas'],                   type: 'bolsas',      ranking: '#601', students: '30,000', founded: '1946' },
    { id: 14, name: 'INSPER',        city: 'São Paulo',       lat: -23.5973, lng: -46.6889, tags: ['Privada','Negócios','EAD'],           type: 'default',     ranking: '#701', students: '4,500',  founded: '2004' },
    { id: 15, name: 'UFPR',          city: 'Curitiba',        lat: -25.4496, lng: -49.2328, tags: ['Pública','Pesquisa'],                 type: 'default',     ranking: '#601', students: '28,000', founded: '1912' },
  ],
  'United States of America': [
    { id: 1,  name: 'MIT',                   city: 'Cambridge, MA',  lat: 42.3601, lng: -71.0942, tags: ['Top 100','Tech','Pesquisa'],   type: 'top100',      ranking: '#1',  students: '11,500', founded: '1861' },
    { id: 2,  name: 'Stanford University',   city: 'Stanford, CA',   lat: 37.4275, lng:-122.1697, tags: ['Top 100','Tech','Bolsas'],     type: 'top100',      ranking: '#3',  students: '17,000', founded: '1885' },
    { id: 3,  name: 'Harvard University',    city: 'Cambridge, MA',  lat: 42.3770, lng: -71.1167, tags: ['Top 100','Pesquisa','Bolsas'], type: 'top100',      ranking: '#4',  students: '21,000', founded: '1636' },
    { id: 4,  name: 'Caltech',               city: 'Pasadena, CA',   lat: 34.1377, lng:-118.1253, tags: ['Top 100','Ciências'],          type: 'top100',      ranking: '#6',  students: '2,300',  founded: '1891' },
    { id: 5,  name: 'Columbia University',   city: 'Nova York, NY',  lat: 40.8075, lng: -73.9626, tags: ['Top 100','Intercâmbio'],       type: 'top100',      ranking: '#11', students: '31,000', founded: '1754' },
    { id: 6,  name: 'UC Berkeley',           city: 'Berkeley, CA',   lat: 37.8719, lng:-122.2585, tags: ['Top 100','Pública','Bolsas'],  type: 'top100',      ranking: '#10', students: '42,000', founded: '1868' },
    { id: 7,  name: 'NYU',                   city: 'Nova York, NY',  lat: 40.7295, lng: -73.9965, tags: ['Intercâmbio','Privada'],       type: 'intercambio', ranking: '#58', students: '51,000', founded: '1831' },
    { id: 8,  name: 'University of Chicago', city: 'Chicago, IL',    lat: 41.7886, lng: -87.5987, tags: ['Top 100','Pesquisa'],          type: 'top100',      ranking: '#12', students: '16,000', founded: '1890' },
    { id: 9,  name: 'Princeton University',  city: 'Princeton, NJ',  lat: 40.3431, lng: -74.6551, tags: ['Top 100','Bolsas'],            type: 'top100',      ranking: '#13', students: '8,500',  founded: '1746' },
    { id: 10, name: 'Yale University',       city: 'New Haven, CT',  lat: 41.3163, lng: -72.9223, tags: ['Top 100','Bolsas'],            type: 'top100',      ranking: '#14', students: '13,000', founded: '1701' },
    { id: 11, name: 'UCLA',                  city: 'Los Angeles, CA',lat: 34.0689, lng:-118.4452, tags: ['Top 100','Pública'],            type: 'top100',      ranking: '#20', students: '46,000', founded: '1919' },
    { id: 12, name: 'University of Michigan',city: 'Ann Arbor, MI',  lat: 42.2780, lng: -83.7382, tags: ['Pública','Intercâmbio'],       type: 'intercambio', ranking: '#23', students: '47,000', founded: '1817' },
    { id: 13, name: 'Duke University',       city: 'Durham, NC',     lat: 35.9940, lng: -78.8986, tags: ['Privada','Bolsas'],            type: 'bolsas',      ranking: '#21', students: '16,000', founded: '1838' },
    { id: 14, name: 'Johns Hopkins',         city: 'Baltimore, MD',  lat: 39.3299, lng: -76.6205, tags: ['Top 100','Pesquisa'],          type: 'top100',      ranking: '#16', students: '24,000', founded: '1876' },
    { id: 15, name: 'Carnegie Mellon',       city: 'Pittsburgh, PA', lat: 40.4432, lng: -79.9428, tags: ['Top 100','Tech'],              type: 'top100',      ranking: '#22', students: '15,000', founded: '1900' },
    { id: 16, name: 'University of Texas',   city: 'Austin, TX',     lat: 30.2849, lng: -97.7341, tags: ['Pública','Bolsas'],            type: 'bolsas',      ranking: '#54', students: '51,000', founded: '1883' },
    { id: 17, name: 'Cornell University',    city: 'Ithaca, NY',     lat: 42.4534, lng: -76.4735, tags: ['Top 100','Intercâmbio'],       type: 'top100',      ranking: '#18', students: '23,000', founded: '1865' },
    { id: 18, name: 'U. of Washington',      city: 'Seattle, WA',    lat: 47.6553, lng:-122.3035, tags: ['Pública','Pesquisa'],          type: 'default',     ranking: '#27', students: '47,000', founded: '1861' },
  ],
  'Canada': [
    { id: 1,  name: 'University of Toronto', city: 'Toronto',   lat: 43.6629, lng: -79.3957, tags: ['Top 100','Pública','Bolsas'],  type: 'top100',      ranking: '#21',  students: '97,000', founded: '1827' },
    { id: 2,  name: 'McGill University',     city: 'Montreal',  lat: 45.5048, lng: -73.5772, tags: ['Top 100','Intercâmbio'],       type: 'top100',      ranking: '#46',  students: '40,000', founded: '1821' },
    { id: 3,  name: 'UBC',                   city: 'Vancouver', lat: 49.2606, lng:-123.2460, tags: ['Top 100','Pesquisa','Bolsas'], type: 'top100',      ranking: '#34',  students: '65,000', founded: '1908' },
    { id: 4,  name: 'University of Waterloo',city: 'Waterloo',  lat: 43.4723, lng: -80.5449, tags: ['Tech','Intercâmbio','Bolsas'], type: 'bolsas',      ranking: '#166', students: '40,000', founded: '1957' },
    { id: 5,  name: 'McMaster University',   city: 'Hamilton',  lat: 43.2609, lng: -79.9192, tags: ['Pública','Pesquisa'],          type: 'default',     ranking: '#152', students: '32,000', founded: '1887' },
    { id: 6,  name: 'Queens University',     city: 'Kingston',  lat: 44.2253, lng: -76.4951, tags: ['Pública','Intercâmbio'],       type: 'intercambio', ranking: '#209', students: '25,000', founded: '1841' },
    { id: 7,  name: 'University of Alberta', city: 'Edmonton',  lat: 53.5232, lng:-113.5263, tags: ['Pública','Bolsas'],            type: 'bolsas',      ranking: '#110', students: '40,000', founded: '1908' },
    { id: 8,  name: 'U. of Ottawa',          city: 'Ottawa',    lat: 45.4231, lng: -75.6831, tags: ['Pública','Intercâmbio'],       type: 'intercambio', ranking: '#279', students: '43,000', founded: '1848' },
    { id: 9,  name: 'Dalhousie University',  city: 'Halifax',   lat: 44.6368, lng: -63.5921, tags: ['Pública','Bolsas'],            type: 'bolsas',      ranking: '#301', students: '19,000', founded: '1818' },
    { id: 10, name: 'U. de Montréal',        city: 'Montreal',  lat: 45.5048, lng: -73.6159, tags: ['Pública','Pesquisa'],          type: 'default',     ranking: '#141', students: '66,000', founded: '1878' },
  ],
  'Australia': [
    { id: 1,  name: 'Australian Nat. Univ.', city: 'Canberra',  lat: -35.2777, lng: 149.1185, tags: ['Top 100','Pública','Bolsas'],  type: 'top100',      ranking: '#30',  students: '25,000', founded: '1946' },
    { id: 2,  name: 'Univ. of Melbourne',    city: 'Melbourne', lat: -37.7963, lng: 144.9614, tags: ['Top 100','Intercâmbio'],       type: 'top100',      ranking: '#33',  students: '52,000', founded: '1853' },
    { id: 3,  name: 'Univ. of Sydney',       city: 'Sydney',    lat: -33.8882, lng: 151.1873, tags: ['Top 100','Pesquisa'],          type: 'top100',      ranking: '#41',  students: '73,000', founded: '1850' },
    { id: 4,  name: 'Univ. of Queensland',   city: 'Brisbane',  lat: -27.4975, lng: 153.0137, tags: ['Top 100','Bolsas'],            type: 'top100',      ranking: '#47',  students: '54,000', founded: '1909' },
    { id: 5,  name: 'Monash University',      city: 'Melbourne', lat: -37.9105, lng: 145.1362, tags: ['Top 100','Intercâmbio'],       type: 'top100',      ranking: '#57',  students: '86,000', founded: '1958' },
    { id: 6,  name: 'UNSW Sydney',            city: 'Sydney',    lat: -33.9173, lng: 151.2313, tags: ['Top 100','Tech'],              type: 'top100',      ranking: '#67',  students: '65,000', founded: '1949' },
    { id: 7,  name: 'Univ. of Adelaide',      city: 'Adelaide',  lat: -34.9205, lng: 138.6010, tags: ['Pública','Bolsas'],            type: 'bolsas',      ranking: '#89',  students: '30,000', founded: '1874' },
    { id: 8,  name: 'Univ. of Western Aust.', city: 'Perth',     lat: -31.9800, lng: 115.8189, tags: ['Pública','Intercâmbio'],       type: 'intercambio', ranking: '#90',  students: '24,000', founded: '1911' },
  ],
  'South Korea': [
    { id: 1,  name: 'Seoul National Univ.',  city: 'Seul',     lat: 37.4601, lng: 126.9519, tags: ['Top 100','Pública','Pesquisa'], type: 'top100',      ranking: '#41',  students: '28,000', founded: '1946' },
    { id: 2,  name: 'KAIST',                city: 'Daejeon',  lat: 36.3719, lng: 127.3608, tags: ['Top 100','Tech','Bolsas'],      type: 'top100',      ranking: '#42',  students: '10,000', founded: '1971' },
    { id: 3,  name: 'Yonsei University',     city: 'Seul',     lat: 37.5640, lng: 126.9388, tags: ['Top 100','Intercâmbio'],       type: 'top100',      ranking: '#56',  students: '40,000', founded: '1885' },
    { id: 4,  name: 'Korea University',      city: 'Seul',     lat: 37.5894, lng: 127.0325, tags: ['Top 100','Pesquisa'],          type: 'top100',      ranking: '#69',  students: '37,000', founded: '1905' },
    { id: 5,  name: 'POSTECH',               city: 'Pohang',   lat: 36.0140, lng: 129.3221, tags: ['Engenharia','Bolsas'],         type: 'bolsas',      ranking: '#87',  students: '4,000',  founded: '1986' },
    { id: 6,  name: 'Sungkyunkwan Univ.',    city: 'Seul',     lat: 37.5873, lng: 126.9938, tags: ['Intercâmbio','Bolsas'],        type: 'bolsas',      ranking: '#95',  students: '33,000', founded: '1398' },
    { id: 7,  name: 'Hanyang University',    city: 'Seul',     lat: 37.5553, lng: 127.0448, tags: ['Engenharia','Privada'],        type: 'default',     ranking: '#145', students: '36,000', founded: '1939' },
  ],
  'China': [
    { id: 1,  name: 'Peking University',     city: 'Pequim',   lat: 39.9926, lng: 116.3108, tags: ['Top 100','Pesquisa','Bolsas'], type: 'top100',      ranking: '#17',  students: '45,000', founded: '1898' },
    { id: 2,  name: 'Tsinghua University',   city: 'Pequim',   lat: 40.0054, lng: 116.3244, tags: ['Top 100','Tech'],              type: 'top100',      ranking: '#16',  students: '47,000', founded: '1911' },
    { id: 3,  name: 'Fudan University',      city: 'Xangai',   lat: 31.2994, lng: 121.5021, tags: ['Top 100','Intercâmbio'],       type: 'top100',      ranking: '#42',  students: '46,000', founded: '1905' },
    { id: 4,  name: 'Shanghai Jiao Tong',    city: 'Xangai',   lat: 31.2000, lng: 121.4239, tags: ['Top 100','Engenharia'],        type: 'top100',      ranking: '#51',  students: '45,000', founded: '1896' },
    { id: 5,  name: 'Zhejiang University',   city: 'Hangzhou', lat: 30.2624, lng: 120.1242, tags: ['Top 100','Pesquisa'],          type: 'top100',      ranking: '#44',  students: '65,000', founded: '1897' },
    { id: 6,  name: 'U. of Science & Tech.', city: 'Hefei',    lat: 31.8418, lng: 117.2510, tags: ['Ciências','Bolsas'],           type: 'bolsas',      ranking: '#57',  students: '16,000', founded: '1958' },
    { id: 7,  name: 'Nankai University',      city: 'Tianjin',  lat: 39.1069, lng: 117.1690, tags: ['Pública','Intercâmbio'],       type: 'intercambio', ranking: '#80',  students: '25,000', founded: '1919' },
  ],
  'Spain': [
    { id: 1,  name: 'Univ. Autónoma de Madrid',city: 'Madrid',    lat: 40.5440, lng: -3.6943, tags: ['Top 100','Pública','Bolsas'], type: 'top100',      ranking: '#191', students: '28,000', founded: '1968' },
    { id: 2,  name: 'Univ. Complutense',       city: 'Madrid',    lat: 40.4511, lng: -3.7275, tags: ['Pública','Intercâmbio'],      type: 'intercambio', ranking: '#220', students: '85,000', founded: '1293' },
    { id: 3,  name: 'Univ. de Barcelona',       city: 'Barcelona', lat: 41.3869, lng:  2.1647, tags: ['Top 100','Pública'],          type: 'top100',      ranking: '#165', students: '63,000', founded: '1450' },
    { id: 4,  name: 'Univ. Pompeu Fabra',       city: 'Barcelona', lat: 41.3854, lng:  2.1899, tags: ['Top 100','Intercâmbio'],      type: 'top100',      ranking: '#194', students: '14,000', founded: '1990' },
    { id: 5,  name: 'Univ. Politécnica Madrid', city: 'Madrid',    lat: 40.3889, lng: -3.6297, tags: ['Engenharia','Pública'],       type: 'default',     ranking: '#301', students: '35,000', founded: '1971' },
    { id: 6,  name: 'Univ. de Sevilla',         city: 'Sevilha',   lat: 37.3887, lng: -5.9928, tags: ['Pública','Intercâmbio'],      type: 'intercambio', ranking: '#401', students: '65,000', founded: '1505' },
    { id: 7,  name: 'Univ. de Valencia',        city: 'Valência',  lat: 39.4779, lng: -0.3478, tags: ['Pública','Bolsas'],           type: 'bolsas',      ranking: '#351', students: '50,000', founded: '1499' },
    { id: 8,  name: 'Univ. de Granada',         city: 'Granada',   lat: 37.1773, lng: -3.5991, tags: ['Pública','Intercâmbio'],      type: 'intercambio', ranking: '#401', students: '60,000', founded: '1531' },
  ],
  'Italy': [
    { id: 1,  name: 'Univ. di Bologna',       city: 'Bolonha',  lat: 44.4958, lng: 11.3528, tags: ['Top 100','Pública','Intercâmbio'], type: 'top100',      ranking: '#154', students: '88,000', founded: '1088' },
    { id: 2,  name: 'Sapienza Roma',           city: 'Roma',     lat: 41.9027, lng: 12.5138, tags: ['Top 100','Pública'],               type: 'top100',      ranking: '#171', students: '110,000',founded: '1303' },
    { id: 3,  name: 'Politecnico di Milano',   city: 'Milão',    lat: 45.4782, lng:  9.2279, tags: ['Top 100','Engenharia'],            type: 'top100',      ranking: '#123', students: '50,000', founded: '1863' },
    { id: 4,  name: 'Univ. di Padova',         city: 'Pádua',    lat: 45.4064, lng: 11.8768, tags: ['Pública','Bolsas'],                type: 'bolsas',      ranking: '#214', students: '62,000', founded: '1222' },
    { id: 5,  name: 'Univ. di Firenze',        city: 'Florença', lat: 43.7696, lng: 11.2558, tags: ['Pública','Intercâmbio'],           type: 'intercambio', ranking: '#351', students: '52,000', founded: '1321' },
    { id: 6,  name: 'Univ. di Milano',         city: 'Milão',    lat: 45.4610, lng:  9.1935, tags: ['Pública','Pesquisa'],              type: 'default',     ranking: '#275', students: '64,000', founded: '1924' },
    { id: 7,  name: 'Univ. di Napoli',         city: 'Nápoles',  lat: 40.8518, lng: 14.2681, tags: ['Pública','Intercâmbio'],           type: 'intercambio', ranking: '#401', students: '80,000', founded: '1224' },
  ],
  'Portugal': [
    { id: 1, name: 'Univ. de Lisboa',          city: 'Lisboa',   lat: 38.7517, lng: -9.1574, tags: ['Top 100','Pública','Bolsas'],  type: 'top100',      ranking: '#301', students: '48,000', founded: '1290' },
    { id: 2, name: 'Univ. do Porto',            city: 'Porto',    lat: 41.1784, lng: -8.6071, tags: ['Pública','Intercâmbio'],       type: 'intercambio', ranking: '#351', students: '30,000', founded: '1911' },
    { id: 3, name: 'IST Lisboa',                city: 'Lisboa',   lat: 38.7367, lng: -9.1395, tags: ['Engenharia','Bolsas'],         type: 'bolsas',      ranking: '#301', students: '11,000', founded: '1911' },
    { id: 4, name: 'Univ. Nova de Lisboa',      city: 'Lisboa',   lat: 38.6607, lng: -9.2059, tags: ['Pública','Intercâmbio'],       type: 'intercambio', ranking: '#401', students: '20,000', founded: '1973' },
    { id: 5, name: 'Univ. de Coimbra',          city: 'Coimbra',  lat: 40.2094, lng: -8.4290, tags: ['Top 100','Pública'],           type: 'top100',      ranking: '#401', students: '25,000', founded: '1290' },
    { id: 6, name: 'Univ. do Minho',            city: 'Braga',    lat: 41.5637, lng: -8.3972, tags: ['Pública','Bolsas'],            type: 'bolsas',      ranking: '#501', students: '19,000', founded: '1973' },
  ],
  'Netherlands': [
    { id: 1, name: 'Delft Univ. of Tech.',     city: 'Delft',     lat: 51.9987, lng:  4.3736, tags: ['Top 100','Engenharia','Bolsas'], type: 'top100',      ranking: '#49',  students: '27,000', founded: '1842' },
    { id: 2, name: 'Leiden University',         city: 'Leiden',    lat: 52.1601, lng:  4.4970, tags: ['Top 100','Pesquisa'],            type: 'top100',      ranking: '#87',  students: '33,000', founded: '1575' },
    { id: 3, name: 'Univ. of Amsterdam',        city: 'Amsterdã',  lat: 52.3667, lng:  4.9041, tags: ['Top 100','Intercâmbio'],         type: 'top100',      ranking: '#53',  students: '40,000', founded: '1632' },
    { id: 4, name: 'Erasmus Univ. Rotterdam',   city: 'Rotterdam', lat: 51.9176, lng:  4.5225, tags: ['Negócios','Bolsas'],             type: 'bolsas',      ranking: '#133', students: '33,000', founded: '1913' },
    { id: 5, name: 'Eindhoven Univ. of Tech.',  city: 'Eindhoven', lat: 51.4484, lng:  5.4906, tags: ['Engenharia','Tech'],             type: 'default',     ranking: '#152', students: '16,000', founded: '1956' },
    { id: 6, name: 'Wageningen University',      city: 'Wageningen',lat: 51.9856, lng:  5.6648, tags: ['Ciências','Bolsas'],            type: 'bolsas',      ranking: '#62',  students: '14,000', founded: '1918' },
  ],
  'Sweden': [
    { id: 1, name: 'KTH Royal Inst. of Tech.',  city: 'Estocolmo', lat: 59.3498, lng: 18.0707, tags: ['Top 100','Engenharia','Bolsas'], type: 'top100',      ranking: '#89',  students: '15,000', founded: '1827' },
    { id: 2, name: 'Karolinska Institutet',      city: 'Estocolmo', lat: 59.3484, lng: 18.0222, tags: ['Top 100','Saúde'],               type: 'top100',      ranking: '#42',  students: '7,000',  founded: '1810' },
    { id: 3, name: 'Stockholm University',        city: 'Estocolmo', lat: 59.3653, lng: 18.0591, tags: ['Pública','Intercâmbio'],         type: 'intercambio', ranking: '#168', students: '33,000', founded: '1878' },
    { id: 4, name: 'Lund University',             city: 'Lund',      lat: 55.7109, lng: 13.2032, tags: ['Top 100','Bolsas'],              type: 'top100',      ranking: '#97',  students: '48,000', founded: '1666' },
    { id: 5, name: 'Uppsala University',          city: 'Uppsala',   lat: 59.8586, lng: 17.6389, tags: ['Top 100','Pesquisa'],            type: 'top100',      ranking: '#121', students: '52,000', founded: '1477' },
  ],
  'Switzerland': [
    { id: 1, name: 'ETH Zürich',           city: 'Zurique',  lat: 47.3763, lng:  8.5484, tags: ['Top 100','Engenharia','Bolsas'], type: 'top100', ranking: '#7',  students: '23,000', founded: '1855' },
    { id: 2, name: 'EPFL',                 city: 'Lausanne', lat: 46.5191, lng:  6.5667, tags: ['Top 100','Tech'],                type: 'top100', ranking: '#14', students: '17,000', founded: '1853' },
    { id: 3, name: 'Univ. of Zürich',      city: 'Zurique',  lat: 47.3739, lng:  8.5494, tags: ['Top 100','Pesquisa'],           type: 'top100', ranking: '#88', students: '28,000', founded: '1833' },
    { id: 4, name: 'Univ. of Bern',        city: 'Berna',    lat: 46.9503, lng:  7.4386, tags: ['Pública','Intercâmbio'],        type: 'intercambio', ranking: '#120', students: '19,000', founded: '1834' },
    { id: 5, name: 'Univ. of Basel',       city: 'Basileia', lat: 47.5596, lng:  7.5826, tags: ['Pública','Bolsas'],             type: 'bolsas',  ranking: '#110', students: '13,000', founded: '1460' },
  ],
  'India': [
    { id: 1, name: 'IIT Bombay',           city: 'Mumbai',    lat: 19.1334, lng:  72.9133, tags: ['Top 100','Engenharia','Bolsas'], type: 'top100',      ranking: '#149', students: '10,000', founded: '1958' },
    { id: 2, name: 'IIT Delhi',            city: 'Nova Delhi',lat: 28.5459, lng:  77.1926, tags: ['Top 100','Tech'],               type: 'top100',      ranking: '#150', students: '8,500',  founded: '1961' },
    { id: 3, name: 'Indian Inst. of Sci.', city: 'Bangalore', lat: 13.0219, lng:  77.5671, tags: ['Top 100','Pesquisa'],           type: 'top100',      ranking: '#155', students: '4,000',  founded: '1909' },
    { id: 4, name: 'IIT Madras',           city: 'Chennai',   lat: 12.9916, lng:  80.2336, tags: ['Top 100','Engenharia'],         type: 'top100',      ranking: '#227', students: '9,000',  founded: '1959' },
    { id: 5, name: 'IIT Kharagpur',        city: 'Kharagpur', lat: 22.3190, lng:  87.3091, tags: ['Engenharia','Bolsas'],          type: 'bolsas',      ranking: '#271', students: '13,000', founded: '1951' },
    { id: 6, name: 'Univ. of Delhi',       city: 'Nova Delhi',lat: 28.6888, lng:  77.2143, tags: ['Pública','Intercâmbio'],        type: 'intercambio', ranking: '#521', students: '300,000',founded: '1922' },
    { id: 7, name: 'Jawaharlal Nehru Univ.',city: 'Nova Delhi',lat: 28.5404, lng:  77.1656, tags: ['Pública','Bolsas'],             type: 'bolsas',      ranking: '#601', students: '8,000',  founded: '1969' },
  ],
}

// Fallback for countries without specific data
function getUniversitiesForCountry(name) {
  return universitiesData[name] || []
}

// ─── Computed ────────────────────────────────────────────────────────────────
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

const filteredMapUniversities = computed(() => {
  const all = getUniversitiesForCountry(selectedCountryName.value)
  if (totalActiveFilters.value === 0) return all

  const selectedUniFilters = activeFilters.universidade
  if (selectedUniFilters.length === 0) return all

  return all.filter(uni => {
    const typeMap = {
      top100: 'top100',
      bolsas: 'bolsas',
      intercambio: 'intercambio',
    }
    return selectedUniFilters.some(f => {
      if (typeMap[f]) return uni.type === typeMap[f]
      if (f === 'publicas') return uni.tags.some(t => t.toLowerCase().includes('pública') || t.toLowerCase().includes('publica'))
      if (f === 'privadas') return uni.tags.some(t => t.toLowerCase().includes('privada'))
      if (f === 'ead') return uni.tags.some(t => t.toLowerCase().includes('ead'))
      return false
    })
  })
})

// ─── Globe helpers ───────────────────────────────────────────────────────────
let colorScale

const universitiesByCountry = {
  'United States of America': 5765, 'India': 5288, 'Brazil': 2690, 'China': 2631,
  'Russia': 896, 'Mexico': 834, 'Japan': 780, 'Indonesia': 740, 'Turkey': 710,
  'Germany': 430, 'France': 400, 'Canada': 380, 'United Kingdom': 370,
  'Argentina': 300, 'Spain': 280, 'Australia': 220, 'Italy': 210,
  'South Korea': 200, 'Poland': 190, 'Portugal': 130, 'Netherlands': 80,
  'Sweden': 65, 'Norway': 45, 'Switzerland': 45, 'Belgium': 55,
  'Denmark': 40, 'Finland': 40, 'Ireland': 30, 'Singapore': 6,
  'New Zealand': 18, 'Austria': 70,
}

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
    .polygonCapColor(d => countryMatchesFilters(d.properties.name) ? colorScale(d.properties.universities) : 'rgba(20,5,30,0.35)')
    .polygonAltitude(d => {
      if (d === hoverD) return 0.12
      return countryMatchesFilters(d.properties.name) ? 0.02 + d.properties.universities / 50000 : 0.005
    })
    .polygonSideColor(d => countryMatchesFilters(d.properties.name) ? 'rgba(128,0,128,0.25)' : 'rgba(20,5,30,0.1)')
}

// ─── Leaflet helpers ─────────────────────────────────────────────────────────
let leafletMarkers = []

const markerColors = {
  top100:      { fill: '#9b30d0', border: '#7b2d8b' },
  bolsas:      { fill: '#2db88a', border: '#1a8a63' },
  intercambio: { fill: '#e8a020', border: '#c07010' },
  default:     { fill: '#4a7fc1', border: '#2a5aa0' },
}

function makeMarkerIcon(type) {
  const c = markerColors[type] || markerColors.default
  const svg = `
    <svg xmlns="http://www.w3.org/2000/svg" width="32" height="40" viewBox="0 0 32 40">
      <defs>
        <filter id="shadow" x="-30%" y="-20%" width="160%" height="160%">
          <feDropShadow dx="0" dy="2" stdDeviation="2" flood-color="rgba(0,0,0,0.35)"/>
        </filter>
      </defs>
      <path d="M16 2C9.37 2 4 7.37 4 14c0 9 12 24 12 24s12-15 12-24c0-6.63-5.37-12-12-12z"
            fill="${c.fill}" stroke="${c.border}" stroke-width="1.5" filter="url(#shadow)"/>
      <circle cx="16" cy="14" r="5" fill="white" opacity="0.9"/>
    </svg>
  `
  return L.divIcon({
    html: svg,
    className: '',
    iconSize: [32, 40],
    iconAnchor: [16, 40],
    popupAnchor: [0, -42],
  })
}

function clearLeafletMarkers() {
  leafletMarkers.forEach(m => m.remove())
  leafletMarkers = []
}

function addUniversityMarkers(unis) {
  if (!leafletMap || !L) return
  clearLeafletMarkers()
  unis.forEach(uni => {
    const marker = L.marker([uni.lat, uni.lng], { icon: makeMarkerIcon(uni.type) })
    marker.bindTooltip(`<strong>${uni.name}</strong><br/>${uni.city}`, {
      direction: 'top', offset: [0, -38], className: 'uni-tooltip'
    })
    marker.on('click', () => {
      selectedUniversity.value = uni
    })
    marker.addTo(leafletMap)
    leafletMarkers.push(marker)
  })
}

// ─── Transition: Globe → Leaflet ─────────────────────────────────────────────
async function openCountryMap(countryName, lat, lng) {
  selectedCountryName.value = countryName
  selectedCountryData.value  = countryMeta[countryName] || null
  selectedUniversity.value   = null
  mapMode.value = true

  await nextTick()

  // Import Leaflet on demand
  if (!L) {
    const leafletModule = await import('leaflet')
    L = leafletModule.default
    // Inject Leaflet CSS
    if (!document.querySelector('#leaflet-css')) {
      const link = document.createElement('link')
      link.id = 'leaflet-css'
      link.rel = 'stylesheet'
      link.href = 'https://unpkg.com/leaflet@1.9.4/dist/leaflet.css'
      document.head.appendChild(link)
      await new Promise(r => { link.onload = r })
    }
  }

  // Initialize or update map
  const meta = countryMeta[countryName]
  const center = meta ? meta.center : [lat, lng]
  const zoom   = meta ? meta.zoom  : 6

  if (!leafletMap) {
    leafletMap = L.map(leafletEl.value, {
      center,
      zoom,
      zoomControl: false,
    })

    // Custom zoom control (top-right)
    L.control.zoom({ position: 'bottomright' }).addTo(leafletMap)

    // Tile layer: CartoDB dark (matches the purple/dark globe theme)
    L.tileLayer('https://{s}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}{r}.png', {
      attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> &copy; <a href="https://carto.com/attributions">CARTO</a>',
      subdomains: 'abcd',
      maxZoom: 19,
    }).addTo(leafletMap)
  } else {
    leafletMap.setView(center, zoom, { animate: true, duration: 1.2 })
    // Force resize after visibility change
    setTimeout(() => leafletMap.invalidateSize(), 100)
  }

  // Add markers for filtered universities
  addUniversityMarkers(filteredMapUniversities.value)
}

function backToGlobe() {
  mapMode.value = false
  selectedUniversity.value = null
  // Small delay then resume rotation
  setTimeout(() => {
    if (globe) globe.controls().autoRotate = true
  }, 400)
}

// Watch filter changes to update markers on map
watch(filteredMapUniversities, (unis) => {
  if (mapMode.value && leafletMap && L) {
    addUniversityMarkers(unis)
  }
})

// ─── University badge helpers ─────────────────────────────────────────────
function getUniBadgeClass(uni) {
  const map = { top100: 'badge-top100', bolsas: 'badge-bolsas', intercambio: 'badge-intercambio', default: 'badge-default' }
  return map[uni.type] || 'badge-default'
}

function getUniTypeLabel(uni) {
  const map = { top100: 'Top 100 Mundial', bolsas: 'Oferece Bolsas', intercambio: 'Intercâmbio', default: 'Universidade' }
  return map[uni.type] || 'Universidade'
}

// ─── Filter logic ─────────────────────────────────────────────────────────
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

// ─── Mount ───────────────────────────────────────────────────────────────────
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
    .polygonLabel(d => {
      const unis = getUniversitiesForCountry(d.properties.name)
      return `
        <div style="font-family:Montserrat,sans-serif;background:rgba(30,0,50,0.92);border:1px solid #7b2d8b;border-radius:8px;padding:10px 14px;min-width:160px">
          <b style="color:#fff;font-size:13px">${d.properties.name}</b>
          <div style="color:#cba8dc;font-size:11px;margin-top:4px">
            ${d.properties.universities.toLocaleString()} universidades
          </div>
          ${unis.length > 0 ? `<div style="color:#9b6acb;font-size:10px;margin-top:2px">${unis.length} com dados detalhados</div>` : ''}
          <div style="color:#7b2d8b;font-size:10px;margin-top:6px;font-weight:700">Clique para explorar ↗</div>
        </div>
      `
    })
    .onPolygonHover(d => {
      hoverD = d
      globe.polygonAltitude(globe.polygonAltitude())
    })
    .onPolygonClick(d => {
      const [lng, lat] = d3.geoCentroid(d)
      globe.controls().autoRotate = false

      // Animate globe to country, then open map
      globe.pointOfView({ lat, lng, altitude: 1.4 }, 900)
      setTimeout(() => {
        openCountryMap(d.properties.name, lat, lng)
      }, 950)
    })

  globe.controls().autoRotate = true
  globe.controls().autoRotateSpeed = 0.3
  globe.controls().maxDistance = globe.getGlobeRadius() * 4
  globe.controls().minDistance = globe.getGlobeRadius() * 1.5

  let rotateTimer = null
  function stopAndScheduleRotation() {
    globe.controls().autoRotate = false
    clearTimeout(rotateTimer)
    rotateTimer = setTimeout(() => { globe.controls().autoRotate = true }, 3000)
  }
  globe.controls().addEventListener('start', stopAndScheduleRotation)

  window.addEventListener('resize', () => {
    globe.width(window.innerWidth).height(window.innerHeight)
    if (leafletMap) leafletMap.invalidateSize()
  })
})

onUnmounted(() => {
  if (leafletMap) { leafletMap.remove(); leafletMap = null }
})
</script>

<style>
@import url('https://fonts.googleapis.com/css2?family=Syne:wght@400;600;700;800&family=DM+Sans:wght@300;400;500&family=Montserrat:wght@400;600;700&display=swap');

/* ─── Navbar ──────────────────────────────────────────────────────── */
.navbar {
  position: fixed;
  top: 12px;
  left: 50%;
  transform: translateX(-50%);
  width: calc(100% - 480px);
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
  z-index: 300;
}
.navbar_brand {
  font-family: 'Syne', sans-serif;
  font-weight: 800;
  font-size: 15px;
  color: #7b2d8b;
  letter-spacing: 0.04em;
}
.navbar_links { display: flex; gap: 28px; }
.navbar_links a { text-decoration: none; color: #1a1a1a; font-size: 13px; font-weight: 600; transition: color 0.2s; }
.navbar_links a:hover { color: #7b2d8b; }
.navbar_cta { display: flex; align-items: center; gap: 8px; background: #7b2d8b; color: #fff; border: none; border-radius: 8px; padding: 8px 16px; font-family: 'Montserrat', sans-serif; font-size: 12px; font-weight: 700; cursor: pointer; transition: background 0.2s; white-space: nowrap; }
.navbar_cta:hover { background: #5a1f68; }
.cta-icon { display: inline-flex; align-items: center; justify-content: center; width: 22px; height: 22px; background: rgba(255,255,255,0.2); border-radius: 50%; font-size: 12px; }

/* ─── Globe ──────────────────────────────────────────────────────── */
.globe-wrap { position: fixed; inset: 0; z-index: 0; cursor: grab; transition: opacity 0.5s ease; }
.globe-wrap:active { cursor: grabbing; }
.globe-hidden { opacity: 0; pointer-events: none; }

/* ─── Filter Panel (left) ────────────────────────────────────────── */
.filter-panel { position: fixed; top: 0; left: 0; height: 100vh; width: 220px; background: #faf8fc; border-right: 1px solid rgba(80,20,100,0.1); display: flex; flex-direction: column; transition: transform 0.4s cubic-bezier(.4,0,.2,1); z-index: 200; font-family: 'DM Sans', sans-serif; }
.filter-panel.collapsed { transform: translateX(-192px); }
.collapse-btn { position: absolute; top: 50%; right: -14px; transform: translateY(-50%); width: 28px; height: 28px; border-radius: 50%; background: #faf8fc; border: 1px solid rgba(80,20,100,0.15); color: #7b2d8b; cursor: pointer; display: flex; align-items: center; justify-content: center; box-shadow: 2px 0 12px rgba(0,0,0,0.08); z-index: 201; transition: background 0.2s, box-shadow 0.2s; }
.collapse-btn:hover { background: #f0e6f8; box-shadow: 2px 0 16px rgba(123,45,139,0.15); }
.panel-inner { padding: 88px 16px 24px; overflow-y: auto; height: 100%; display: flex; flex-direction: column; gap: 2px; scrollbar-width: thin; scrollbar-color: #ddd transparent; }
.panel-heading { display: flex; align-items: center; gap: 10px; margin-bottom: 16px; }

/* ─── Botão voltar ao globo (dentro do painel) ───────────────────── */
.back-to-globe-btn {
  display: flex;
  align-items: center;
  gap: 8px;
  width: 100%;
  margin-bottom: 16px;
  padding: 10px 12px;
  background: #f0e6f8;
  border: 1.5px solid #c9a0dc;
  border-radius: 8px;
  color: #5a1f68;
  font-family: 'Montserrat', sans-serif;
  font-size: 11px;
  font-weight: 700;
  letter-spacing: 0.05em;
  text-transform: uppercase;
  cursor: pointer;
  transition: background 0.2s, border-color 0.2s, color 0.2s;
}
.back-to-globe-btn:hover {
  background: #7b2d8b;
  border-color: #7b2d8b;
  color: #fff;
}
.back-to-globe-btn svg {
  flex-shrink: 0;
  transition: transform 0.2s;
}
.back-to-globe-btn:hover svg {
  transform: translateX(-2px);
}
.panel-label { font-family: 'Syne', sans-serif; font-weight: 800; font-size: 11px; letter-spacing: 0.18em; text-transform: uppercase; color: #1a001a; white-space: nowrap; }
.panel-line { flex: 1; height: 1px; background: linear-gradient(to right, rgba(123,45,139,0.3), transparent); }
.filter-section { overflow: hidden; border-bottom: 1px solid rgba(80,20,100,0.06); }
.section-header { display: flex; align-items: center; justify-content: space-between; padding: 11px 4px; cursor: pointer; user-select: none; transition: color 0.2s; }
.section-header:hover .section-label { color: #7b2d8b; }
.section-header .section-label { font-family: 'Montserrat', sans-serif; font-weight: 700; font-size: 10px; color: #2a002a; letter-spacing: 0.1em; text-transform: uppercase; transition: color 0.2s; }
.section-arrow { color: #bba0cc; transition: transform 0.25s cubic-bezier(.4,0,.2,1), color 0.2s; flex-shrink: 0; }
.section-arrow.open { transform: rotate(180deg); color: #7b2d8b; }
.section-options { max-height: 0; overflow: hidden; transition: max-height 0.35s cubic-bezier(.4,0,.2,1); }
.section-options.open { max-height: 400px; }
.option-item { display: flex; align-items: center; gap: 9px; padding: 7px 4px 7px 8px; cursor: pointer; border-radius: 4px; transition: background 0.15s; margin-bottom: 1px; }
.option-item:hover { background: rgba(123,45,139,0.05); }
.option-check { width: 14px; height: 14px; border: 1.5px solid #cbb8d8; border-radius: 3px; display: flex; align-items: center; justify-content: center; flex-shrink: 0; background: #fff; transition: all 0.15s; }
.option-item.active .option-check { background: #7b2d8b; border-color: #7b2d8b; }
.option-text { flex: 1; font-size: 12px; color: #4a2a5a; font-weight: 400; transition: color 0.15s, font-weight 0.15s; }
.option-item.active .option-text { color: #1a001a; font-weight: 500; }
.option-count { font-size: 9px; color: #9a6aaa; font-family: 'Montserrat', sans-serif; font-weight: 600; letter-spacing: 0.03em; }
.clear-btn { margin-top: auto; padding: 10px; border: 1px solid rgba(123,45,139,0.25); border-radius: 6px; background: transparent; color: #7b2d8b; font-family: 'Montserrat', sans-serif; font-size: 9px; font-weight: 700; cursor: pointer; text-transform: uppercase; letter-spacing: 0.12em; transition: color 0.2s, border-color 0.2s; position: relative; overflow: hidden; }
.clear-btn::after { content: ''; position: absolute; inset: 0; background: #7b2d8b; opacity: 0; transition: opacity 0.2s; }
.clear-btn:hover { color: #fff; border-color: #7b2d8b; }
.clear-btn:hover::after { opacity: 1; }
.clear-btn span { position: relative; z-index: 1; }

/* ─── Agencies Panel (right) ─────────────────────────────────────── */
.agencies-panel { position: fixed; top: 0; right: 0; height: 100vh; width: 220px; background: #f5f0f0; border-left: 1px solid #e0d8e8; display: flex; flex-direction: column; z-index: 200; }
.agencies-title { font-family: 'Syne', sans-serif; font-weight: 800; font-size: 13px; color: #2a002a; text-transform: uppercase; letter-spacing: 0.1em; padding: 80px 16px 12px; margin: 0; border-bottom: 1px solid #e0d0e8; }
.agencies-list { flex: 1; overflow-y: auto; padding: 10px; display: flex; flex-direction: column; gap: 8px; scrollbar-width: thin; scrollbar-color: #d0b8e0 transparent; }
.agency-card { background: #fff; border: 1px solid #e8ddf0; border-radius: 8px; padding: 10px 12px; display: flex; flex-direction: column; gap: 5px; transition: box-shadow 0.2s, border-color 0.2s; }
.agency-card:hover { box-shadow: 0 3px 12px rgba(123,45,139,0.1); border-color: #c0a0d0; }
.agency-header { display: flex; flex-direction: column; gap: 2px; }
.agency-name { font-family: 'Syne', sans-serif; font-weight: 800; font-size: 12px; color: #1a001a; text-transform: uppercase; letter-spacing: 0.04em; }
.agency-stars { display: flex; gap: 1px; }
.star { font-size: 11px; color: #ddd; }
.star.filled { color: #7b2d8b; }
.agency-desc { font-size: 10px; color: #5a3a6a; line-height: 1.5; margin: 0; }
.agency-footer { display: flex; align-items: center; justify-content: space-between; margin-top: 2px; }
.agency-location { font-size: 9px; color: #8a5a9a; font-family: 'DM Sans', sans-serif; }
.agency-btn { background: #7b2d8b; color: #fff; border: none; border-radius: 5px; padding: 4px 10px; font-family: 'Montserrat', sans-serif; font-size: 9px; font-weight: 700; cursor: pointer; transition: background 0.2s; letter-spacing: 0.04em; text-transform: uppercase; }
.agency-btn:hover { background: #5a1f68; }
.no-agencies { font-size: 11px; color: #8a5a9a; text-align: center; padding: 20px 0; font-family: 'DM Sans', sans-serif; }

/* ─── Active badge ───────────────────────────────────────────────── */
.active-badge { position: fixed; bottom: 20px; left: 50%; transform: translateX(-50%); background: #7b2d8b; color: #fff; font-family: 'Montserrat', sans-serif; font-size: 11px; font-weight: 700; padding: 5px 14px; border-radius: 20px; box-shadow: 0 4px 16px rgba(123,45,139,0.35); z-index: 300; animation: fadeIn 0.2s ease; letter-spacing: 0.03em; }

/* ─── Map overlay ────────────────────────────────────────────────── */
.map-overlay { position: fixed; inset: 0; z-index: 150; display: flex; flex-direction: column; }

.map-slide-enter-active { transition: opacity 0.45s ease, transform 0.45s cubic-bezier(0.22,1,0.36,1); }
.map-slide-leave-active { transition: opacity 0.35s ease, transform 0.35s ease; }
.map-slide-enter-from { opacity: 0; transform: scale(1.04); }
.map-slide-leave-to   { opacity: 0; transform: scale(0.97); }

/* ─── Floating back button ───────────────────────────────────────── */
.back-btn-float {
  position: absolute;
  top: 20px;
  left: calc(220px + 20px);
  z-index: 400;
  display: flex;
  align-items: center;
  gap: 8px;
  background: rgba(10, 0, 24, 0.88);
  backdrop-filter: blur(14px);
  border: 1px solid rgba(123, 45, 139, 0.6);
  color: #e0c0f0;
  border-radius: 10px;
  padding: 10px 18px;
  font-family: 'Montserrat', sans-serif;
  font-size: 12px;
  font-weight: 700;
  cursor: pointer;
  letter-spacing: 0.04em;
  transition: background 0.2s, border-color 0.2s, color 0.2s, transform 0.15s;
  box-shadow: 0 4px 20px rgba(0,0,0,0.4);
}
.back-btn-float:hover {
  background: rgba(123, 45, 139, 0.55);
  border-color: #a050c0;
  color: #fff;
  transform: translateY(-1px);
}
.back-btn-float:active { transform: translateY(0); }

/* ─── Country chip ───────────────────────────────────────────────── */
.map-country-chip {
  position: absolute;
  top: 20px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 400;
  display: flex;
  align-items: center;
  gap: 12px;
  background: rgba(10, 0, 24, 0.88);
  backdrop-filter: blur(14px);
  border: 1px solid rgba(123, 45, 139, 0.4);
  border-radius: 12px;
  padding: 10px 20px;
  box-shadow: 0 4px 20px rgba(0,0,0,0.4);
}
.map-country-flag { font-size: 26px; }
.map-country-name { font-family: 'Syne', sans-serif; font-weight: 800; font-size: 16px; color: #fff; margin: 0; }
.map-country-stats { font-size: 11px; color: #9b6acb; margin: 2px 0 0; font-family: 'DM Sans', sans-serif; }

/* ─── Legend chip ────────────────────────────────────────────────── */
.map-legend-chip {
  position: absolute;
  top: 20px;
  right: calc(220px + 20px);
  z-index: 400;
  display: flex;
  align-items: center;
  gap: 10px;
  background: rgba(10, 0, 24, 0.88);
  backdrop-filter: blur(14px);
  border: 1px solid rgba(123, 45, 139, 0.4);
  border-radius: 10px;
  padding: 10px 16px;
  font-size: 10px;
  color: #c0a0dc;
  font-family: 'DM Sans', sans-serif;
  box-shadow: 0 4px 20px rgba(0,0,0,0.4);
  white-space: nowrap;
}
.legend-dot { width: 9px; height: 9px; border-radius: 50%; display: inline-block; margin-right: 4px; flex-shrink: 0; }
.legend-top100      { background: #9b30d0; }
.legend-bolsas      { background: #2db88a; }
.legend-intercambio { background: #e8a020; }
.legend-default     { background: #4a7fc1; }

/* ─── Leaflet container ──────────────────────────────────────────── */
.leaflet-container { flex: 1; width: 100%; }

/* Override Leaflet styles to match theme */
.leaflet-container .leaflet-tile-pane { filter: saturate(0.9) brightness(0.95); }
.uni-tooltip { background: rgba(20,0,40,0.95) !important; border: 1px solid #7b2d8b !important; border-radius: 6px !important; color: #e0c0f0 !important; font-family: 'DM Sans', sans-serif !important; font-size: 11px !important; padding: 6px 10px !important; }
.uni-tooltip::before { border-top-color: #7b2d8b !important; }
.leaflet-control-zoom a { background: rgba(30,0,50,0.9) !important; color: #e0c0f0 !important; border-color: rgba(123,45,139,0.4) !important; }
.leaflet-control-zoom a:hover { background: rgba(123,45,139,0.5) !important; }
.leaflet-bar { border: 1px solid rgba(123,45,139,0.35) !important; }
.leaflet-attribution-flag { display: none !important; }
.leaflet-control-attribution { background: rgba(10,0,20,0.7) !important; color: #666 !important; font-size: 9px !important; }

/* ─── University info panel ──────────────────────────────────────── */
.uni-info-panel { position: absolute; bottom: 32px; left: calc(220px + 24px); width: 300px; background: rgba(12,0,24,0.96); backdrop-filter: blur(20px); border: 1px solid rgba(123,45,139,0.4); border-radius: 16px; padding: 20px 20px 18px; z-index: 400; font-family: 'DM Sans', sans-serif; }

.info-slide-enter-active { transition: all 0.3s cubic-bezier(0.22,1,0.36,1); }
.info-slide-leave-active { transition: all 0.2s ease; }
.info-slide-enter-from { opacity: 0; transform: translateY(16px); }
.info-slide-leave-to   { opacity: 0; transform: translateY(8px); }

.uni-close { position: absolute; top: 12px; right: 12px; background: rgba(123,45,139,0.2); border: 1px solid rgba(123,45,139,0.3); color: #9b6acb; border-radius: 50%; width: 26px; height: 26px; cursor: pointer; font-size: 11px; display: flex; align-items: center; justify-content: center; transition: background 0.2s; }
.uni-close:hover { background: rgba(123,45,139,0.4); color: #fff; }

.uni-type-badge { display: inline-block; padding: 3px 10px; border-radius: 20px; font-family: 'Montserrat', sans-serif; font-size: 9px; font-weight: 700; letter-spacing: 0.08em; text-transform: uppercase; margin-bottom: 10px; }
.badge-top100     { background: rgba(155,48,208,0.2); color: #c070e8; border: 1px solid rgba(155,48,208,0.35); }
.badge-bolsas     { background: rgba(45,184,138,0.15); color: #4cd8a0; border: 1px solid rgba(45,184,138,0.3); }
.badge-intercambio{ background: rgba(232,160,32,0.15); color: #f0c060; border: 1px solid rgba(232,160,32,0.3); }
.badge-default    { background: rgba(74,127,193,0.15); color: #80b0e0; border: 1px solid rgba(74,127,193,0.3); }

.uni-info-name { font-family: 'Syne', sans-serif; font-weight: 800; font-size: 16px; color: #fff; margin: 0 0 6px; line-height: 1.2; }
.uni-info-location { font-size: 12px; color: #9b6acb; margin: 0 0 12px; }

.uni-info-tags { display: flex; flex-wrap: wrap; gap: 5px; margin-bottom: 14px; }
.uni-tag { background: rgba(123,45,139,0.15); border: 1px solid rgba(123,45,139,0.25); color: #cba0e0; border-radius: 12px; padding: 3px 9px; font-size: 10px; font-family: 'Montserrat', sans-serif; font-weight: 600; }

.uni-info-stats { display: grid; grid-template-columns: repeat(3,1fr); gap: 10px; margin-bottom: 16px; padding: 12px; background: rgba(255,255,255,0.04); border-radius: 10px; }
.uni-stat { text-align: center; }
.uni-stat-value { display: block; font-family: 'Syne', sans-serif; font-weight: 700; font-size: 14px; color: #e0c0ff; }
.uni-stat-label { display: block; font-size: 9px; color: #7a4a9a; font-family: 'DM Sans', sans-serif; margin-top: 2px; text-transform: uppercase; letter-spacing: 0.05em; }

.uni-visit-btn { display: block; width: 100%; padding: 10px; background: linear-gradient(135deg, #7b2d8b, #5a1f68); color: #fff; border: none; border-radius: 8px; font-family: 'Montserrat', sans-serif; font-size: 11px; font-weight: 700; text-align: center; text-decoration: none; cursor: pointer; transition: opacity 0.2s; letter-spacing: 0.04em; }
.uni-visit-btn:hover { opacity: 0.85; }

/* ─── Animations ─────────────────────────────────────────────────── */
@keyframes fadeIn {
  from { opacity: 0; transform: translateX(-50%) translateY(6px); }
  to   { opacity: 1; transform: translateX(-50%) translateY(0); }
}
</style>
