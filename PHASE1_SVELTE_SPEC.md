# Magyar Politika Idle — Fázis 1
## Svelte + Vite átállás + Pártalapítási képernyő
### Claude Code specifikáció

---

## Összefoglalás

**Cél:** Az egyetlen `magyar-politika-idle.html` fájlt átmigrálni egy Svelte + Vite projektbe, majd erre ráépíteni a pártalapítási wizard képernyőt. A játék logikája és megjelenése **változatlan** marad — csak az architektúra változik.

**Végeredmény:** `npm run dev` → fut a játék Vite dev serverrel. `npm run build` → egyetlen `dist/` mappa ami statikusan kiszolgálható.

---

## 1. Projekt létrehozása

```bash
npm create vite@latest magyar-politika-idle -- --template svelte
cd magyar-politika-idle
npm install
```

**Fájlstruktúra amit létre kell hozni:**

```
magyar-politika-idle/
├── index.html              # Vite entry point (minimális)
├── vite.config.js
├── package.json
├── src/
│   ├── main.js             # App mount
│   ├── App.svelte          # Root: routing founding ↔ game
│   ├── lib/
│   │   ├── gameData.js     # BUILDINGS, UPGRADES, MILESTONES, ACHIEVEMENTS, NEWS
│   │   ├── gameStore.js    # Svelte writable store — a game state
│   │   ├── gameLoop.js     # rAF loop, calcVps, bldCost, fmt
│   │   ├── partyData.js    # TOPICS, TOPIC_STANCES, getArchetype, calculateIdeology
│   │   └── partyStore.js   # Svelte writable store — a party config
│   ├── components/
│   │   ├── game/
│   │   │   ├── GameLayout.svelte    # A teljes 3-oszlopos játék layout
│   │   │   ├── Header.svelte        # Sticky header + stats
│   │   │   ├── Ticker.svelte        # Hírticker
│   │   │   ├── ClickerPanel.svelte  # Bal panel: kattintás, milestone
│   │   │   ├── BuildingsPanel.svelte # Közép panel
│   │   │   ├── UpgradesPanel.svelte  # Jobb panel fejlesztések
│   │   │   └── EventsPanel.svelte   # Jobb panel hírek
│   │   └── founding/
│   │       ├── FoundingWizard.svelte # Wrapper, step kezelés
│   │       ├── Step1Name.svelte      # Párt neve, rövidítés, szín
│   │       ├── Step2Topics.svelte    # 5 téma kiválasztása
│   │       ├── Step3Stances.svelte   # Álláspontok
│   │       └── Step4Summary.svelte   # Ideológiai térkép + indítás
│   └── styles/
│       └── global.css      # Az összes CSS a jelenlegi HTML-ből
```

---

## 2. CSS migráció

Az összes CSS tartalom a jelenlegi `<style>` blokkból kerüljön `src/styles/global.css`-be **változtatás nélkül**. A Google Fonts import is maradjon.

`index.html`-ben:
```html
<!DOCTYPE html>
<html lang="hu">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Magyar Politika Idle</title>
</head>
<body>
  <div id="app"></div>
  <script type="module" src="/src/main.js"></script>
</body>
</html>
```

`src/main.js`:
```javascript
import './styles/global.css'
import App from './App.svelte'

const app = new App({ target: document.getElementById('app') })
export default app
```

---

## 3. Game adatok migrációja

### `src/lib/gameData.js`
A meglévő konstansok **szó szerint** másolva, csak `export const` lesz belőlük:

```javascript
export const BUILDINGS = [
  { id:'poster', icon:'📋', name:'Plakátragasztó', desc:'Éjjel-nappal ragaszt',
    base:10, vps:0.1, flavorPre:'Valaki', flavorPost:'plakátot ragasztott.' },
  { id:'bus', icon:'🚌', name:'Kampánybusz', desc:'Falvakat jár be',
    base:100, vps:0.5, flavorPre:'A busz', flavorPost:'megérkezett.' },
  { id:'press', icon:'🎙', name:'Sajtótájékoztató', desc:'Nem kell kérdés',
    base:500, vps:4, flavorPre:'Sajtó', flavorPost:'tájékoztattak mindenkit.' },
  { id:'tvspot', icon:'📺', name:'TV-spot', desc:'Ismétlés, ismétlés',
    base:2000, vps:20, flavorPre:'Láttak', flavorPost:'egy TV-spotot.' },
  { id:'media', icon:'📡', name:'Kormányközeli médium', desc:'Tények? Mi az?',
    base:10000, vps:100, flavorPre:'Az igazság', flavorPost:'kiderült.' },
  { id:'stadium', icon:'🏟', name:'Stadionépítés', desc:'Örökség marad',
    base:50000, vps:500, flavorPre:'Átadták', flavorPost:'az új stadiont.' },
  { id:'eu', icon:'🇪🇺', name:'EU-s pénzek', desc:'Brüsszeltől',
    base:200000, vps:2500, flavorPre:'Érkezett', flavorPost:'némi forrás.' },
  { id:'constit', icon:'📜', name:'Alkotmánymódosítás', desc:'Két perces szavazás',
    base:1000000, vps:12000, flavorPre:'Megszavazták', flavorPost:'a módosítást.' },
  { id:'nat', icon:'🦅', name:'Nemzeti Konzultáció', desc:'Kérdés már van',
    base:5000000, vps:60000, flavorPre:'Beérkezett', flavorPost:'egy névtelen ív.' },
]

export const UPGRADES = [
  { id:'u1', bld:'poster', mult:2, name:'Foszforeszkáló plakát', desc:'Sötétben is látszik', cost:100, icon:'✨' },
  { id:'u2', bld:'bus', mult:2, name:'Dupladecker busz', desc:'Kétszer annyi ember', cost:500, icon:'🚌' },
  { id:'u3', bld:'poster', mult:2, name:'Időjárásálló ragasztó', desc:'Eső sem mossa le', cost:1000, icon:'🌧' },
  { id:'u4', bld:'press', mult:2, name:'Szónoki állvány', desc:'Tekintélyesebb megjelenés', cost:2000, icon:'🎤' },
  { id:'u5', bld:'bus', mult:2, name:'Kifőzde a buszon', desc:'Lángos menet közben', cost:5000, icon:'🥙' },
  { id:'u6', bld:'tvspot', mult:2, name:'Primetime spot', desc:'Mindenki nézi', cost:10000, icon:'⭐' },
  { id:'u7', bld:'media', mult:2, name:'Hatodik csatorna', desc:'Csak a mi hangunk', cost:40000, icon:'📻' },
  { id:'u8', bld:'press', mult:2, name:'Kérdés nélküli sajtó', desc:'Csak közlemény van', cost:25000, icon:'🚫' },
  { id:'u9', bld:'stadium', mult:2, name:'Névadói jog', desc:'Mindenki tudja', cost:150000, icon:'💰' },
  { id:'u10', bld:'eu', mult:2, name:'Kohéziós alap', desc:'Plusz csatorna', cost:500000, icon:'💶' },
  { id:'u11', bld:'constit', mult:2, name:'Sarkalatos törvény', desc:'Kétharmad kellene', cost:2000000, icon:'⚖' },
  { id:'u12', bld:'nat', mult:2, name:'Kérdések kérdése', desc:'Mit kérdezzünk?', cost:10000000, icon:'❓' },
  { id:'u13', bld:null, mult:2, name:'Össznépi összefogás', desc:'Mindenki kettőt kattint', cost:50, icon:'👐', clickMult:true },
  { id:'u14', bld:null, mult:5, name:'Parlamenti napidíj', desc:'Ötszörös szavazat kézfogásonként', cost:2000, icon:'💼', clickMult:true },
  { id:'u15', bld:null, mult:2, name:'Selfie a néppel', desc:'Viral lett', cost:200, icon:'🤳', clickMult:true },
]

export const MILESTONES = [
  { votes:0,         title:'Ismeretlen aktivista' },
  { votes:100,       title:'Falusi képviselő' },
  { votes:1000,      title:'Kerületi politikus' },
  { votes:10000,     title:'Parlamenti képviselő' },
  { votes:100000,    title:'Frakcióvezető' },
  { votes:1000000,   title:'Miniszter' },
  { votes:10000000,  title:'Miniszterelnök' },
  { votes:50000000,  title:'Kétharmados győztes' },
  { votes:200000000, title:'Örök kétharmad' },
  { votes:1e9,       title:'Alapító atya' },
]

export const ACHIEVEMENTS = [
  { id:'a1', label:'Első kézfogás',  cond: s => s.totalClicks >= 1 },
  { id:'a2', label:'100 kézfogás',   cond: s => s.totalClicks >= 100 },
  { id:'a3', label:'Első plakát',    cond: s => (s.owned.poster||0) >= 1 },
  { id:'a4', label:'Kampánybusz',    cond: s => (s.owned.bus||0) >= 1 },
  { id:'a5', label:'Milliomos',      cond: s => s.totalVotes >= 1000000 },
  { id:'a6', label:'TV-szereplő',    cond: s => (s.owned.tvspot||0) >= 1 },
  { id:'a7', label:'Stadionfoglaló', cond: s => (s.owned.stadium||0) >= 1 },
  { id:'a8', label:'Kétharmad!',     cond: s => s.totalVotes >= 50000000 },
]

export const NEWS = [
  { type:'bad',     text:'Az ellenzék összefog. Aztán szétesik.' },
  { type:'neutral', text:'Orbán: „A Soros-terv veszélyezteti a plakátjainkat."' },
  { type:'good',    text:'Brüsszel megint beavatkozik a belviszonyainkba.' },
  { type:'neutral', text:'DK: „Mi vagyunk az igazi ellenzék." Momentum: „Nem, mi."' },
  { type:'bad',     text:'A kérdőívek 94%-a visszaérkezett. A kérdések ismeretlenek.' },
  { type:'good',    text:'Felavatták az ötödik stadiont. A focipálya hamarosan épül.' },
  { type:'neutral', text:'Fidesz: „Ez megint Brüsszel." Brüsszel: ???' },
  { type:'bad',     text:'MSZP frakcióülés: öten vannak, három frakció.' },
  { type:'good',    text:'Közmédia: „Tárgyilagos tájékoztatás, mint mindig."' },
  { type:'neutral', text:'Momentum: „Eljön a változás." (2014-óta)' },
  { type:'bad',     text:'Az alkotmánymódosítást 7 perc alatt szavazták meg.' },
  { type:'good',    text:'Paks II. hamarosan kész. A tervek szerint 2043-ban.' },
  { type:'neutral', text:'Ellenzéki összefogás: 12 párt, 11 elnök, 0 program.' },
  { type:'bad',     text:'EU-s pénz érkezett. Elköltötték. Auditorok keresik.' },
  { type:'good',    text:'Karácsony: „Jövőre meglesz a főváros." Jövőre megint jövőre.' },
  { type:'neutral', text:'Közvélemény-kutatás: 40%-os támogatás minden párt mérése szerint.' },
  { type:'bad',     text:'Jogállamisági eljárás indul. Mi: „Irigység."' },
  { type:'good',    text:'Magyar siker: egy kézilabdás nyert valamit valahol.' },
  { type:'neutral', text:'Sajtótájékoztató kérdések nélkül. Válaszok kérdések nélkül.' },
  { type:'bad',     text:'A főpolgármester elmondja, mit tesz, ha… Budapest nem engedi.' },
]
```

---

## 4. Game Store — `src/lib/gameStore.js`

A state két részre bomlik: **belső state** (plain JS object, 60fps-en fut) és **display state** (Svelte store, ~10x/mp frissül a UI-hoz).

```javascript
import { writable, derived } from 'svelte/store'
import { BUILDINGS, UPGRADES, MILESTONES, ACHIEVEMENTS } from './gameData.js'

// ── BELSŐ STATE (plain JS, nem reaktív — a game loop ezen dolgozik) ──
export const internal = {
  votes: 0,
  totalVotes: 0,
  vps: 0,
  clickPower: 1,
  owned: {},        // { poster: 2, bus: 1, ... }
  boughtUpgrades: new Set(),
  boughtAchievements: new Set(),
  totalClicks: 0,
  startTime: Date.now(),
  bldMultipliers: Object.fromEntries(BUILDINGS.map(b => [b.id, 1])),
}

// ── DISPLAY STATE (Svelte store — UI figyeli) ──
export const display = writable({
  votes: 0,
  totalVotes: 0,
  vps: 0,
  clickPower: 1,
  owned: {},
  boughtUpgrades: [],   // Set-ből array a Svelte-hez
  events: [],
  elapsedSec: 0,
})

// ── SEGÉDFÜGGVÉNYEK ──
export function fmt(n) {
  if (n < 1000) return Math.floor(n).toString()
  if (n < 1e6)  return (n/1000).toFixed(1).replace('.0','') + 'E'
  if (n < 1e9)  return (n/1e6).toFixed(2).replace(/\.?0+$/, '') + 'M'
  if (n < 1e12) return (n/1e9).toFixed(2).replace(/\.?0+$/, '') + 'Mrd'
  return (n/1e12).toFixed(2) + 'T'
}

export function bldCost(buildingId) {
  const b = BUILDINGS.find(b => b.id === buildingId)
  const owned = internal.owned[buildingId] || 0
  return Math.floor(b.base * Math.pow(1.15, owned))
}

export function calcVps() {
  let total = 0
  BUILDINGS.forEach(b => {
    const cnt = internal.owned[b.id] || 0
    total += cnt * b.vps * internal.bldMultipliers[b.id]
  })
  internal.vps = total
}

// Display state szinkronizálás — hívd ~10x/mp
export function syncDisplay() {
  display.update(() => ({
    votes: internal.votes,
    totalVotes: internal.totalVotes,
    vps: internal.vps,
    clickPower: internal.clickPower,
    owned: { ...internal.owned },
    boughtUpgrades: [...internal.boughtUpgrades],
    events: [...internal.events],
    elapsedSec: Math.floor((Date.now() - internal.startTime) / 1000),
  }))
}

// ── AKCIÓK ──
export function buyBuilding(buildingId) {
  const cost = bldCost(buildingId)
  if (internal.votes < cost) return false
  const b = BUILDINGS.find(b => b.id === buildingId)
  internal.votes -= cost
  internal.owned[buildingId] = (internal.owned[buildingId] || 0) + 1
  calcVps()
  addEvent(b.flavorPre + ' ' + b.flavorPost + ' (' + b.name + ')')
  syncDisplay()
  return true
}

export function buyUpgrade(upgradeId) {
  const u = UPGRADES.find(u => u.id === upgradeId)
  if (!u || internal.boughtUpgrades.has(upgradeId)) return false
  if (internal.votes < u.cost) return false
  internal.votes -= u.cost
  internal.boughtUpgrades.add(upgradeId)
  if (u.clickMult) {
    internal.clickPower *= u.mult
  } else if (u.bld) {
    internal.bldMultipliers[u.bld] *= u.mult
  }
  calcVps()
  addEvent('Fejlesztés megvásárolva: ' + u.name, 'good')
  syncDisplay()
  return true
}

export function doClick() {
  internal.votes += internal.clickPower
  internal.totalVotes += internal.clickPower
  internal.totalClicks++
  // Click esetén azonnal szinkronizálunk
  syncDisplay()
  // achievement check
  checkAchievements()
}

// ── EVENTS ──
export const events = writable([])

export function addEvent(text, type = 'neutral') {
  const now = new Date()
  const time = String(now.getHours()).padStart(2,'0') + ':' +
               String(now.getMinutes()).padStart(2,'0') + ':' +
               String(now.getSeconds()).padStart(2,'0')
  internal.events = [{ text, type, time }, ...internal.events].slice(0, 20)
  events.set(internal.events)
}

// ── ACHIEVEMENTS ──
export function checkAchievements() {
  ACHIEVEMENTS.forEach(a => {
    if (!internal.boughtAchievements.has(a.id) && a.cond(internal)) {
      internal.boughtAchievements.add(a.id)
      addEvent('Eredmény: ' + a.label + ' 🏅', 'good')
    }
  })
}

// ── RANDOM EVENTS ──
export function triggerRandomEvent() {
  if (internal.totalVotes < 10) return
  const which = Math.floor(Math.random() * 5)
  if (which === 0) {
    const bonus = Math.floor(internal.vps * 30 + 50)
    internal.votes += bonus
    internal.totalVotes += bonus
    addEvent('🎉 Spontán tüntetés! +' + fmt(bonus) + ' szavazat', 'good')
  } else if (which === 1) {
    const loss = Math.floor(internal.votes * 0.05 + 10)
    internal.votes = Math.max(0, internal.votes - loss)
    addEvent('😤 Botrány! -' + fmt(loss) + ' szavazat', 'bad')
  } else if (which === 2) {
    internal.clickPower *= 1.5
    setTimeout(() => { internal.clickPower /= 1.5 }, 15000)
    addEvent('⚡ Viral pillanat! 15mp: 1.5x kattintás erő!', 'good')
  } else if (which === 3) {
    // csak hír
    addEvent('📰 Hírek...', 'neutral')
  } else {
    const bonus = Math.floor(internal.vps * 60 + 100)
    internal.votes += bonus
    internal.totalVotes += bonus
    addEvent('🗳 Sikeres kampányrendezvény! +' + fmt(bonus) + ' szavazat', 'good')
  }
  syncDisplay()
}

// ── PERSISTENCE (localStorage) ──
export function saveGame() {
  const saveData = {
    votes: internal.votes,
    totalVotes: internal.totalVotes,
    clickPower: internal.clickPower,
    owned: internal.owned,
    boughtUpgrades: [...internal.boughtUpgrades],
    boughtAchievements: [...internal.boughtAchievements],
    totalClicks: internal.totalClicks,
    startTime: internal.startTime,
    bldMultipliers: internal.bldMultipliers,
    events: internal.events,
  }
  localStorage.setItem('gameState', JSON.stringify(saveData))
}

export function loadGame() {
  const raw = localStorage.getItem('gameState')
  if (!raw) return false
  try {
    const data = JSON.parse(raw)
    Object.assign(internal, {
      ...data,
      boughtUpgrades: new Set(data.boughtUpgrades || []),
      boughtAchievements: new Set(data.boughtAchievements || []),
    })
    calcVps()
    syncDisplay()
    return true
  } catch { return false }
}
```

---

## 5. Game Loop — `src/lib/gameLoop.js`

```javascript
import { internal, syncDisplay, triggerRandomEvent, saveGame, checkAchievements } from './gameStore.js'

let lastTick = Date.now()
let frameCount = 0
let lastRandomEvent = Date.now()
let lastSave = Date.now()
let rafId = null

export function startGameLoop() {
  if (rafId) return // már fut
  lastTick = Date.now()

  function tick() {
    const now = Date.now()
    const dt = (now - lastTick) / 1000
    lastTick = now
    frameCount++

    // Core idle tick
    const gain = internal.vps * dt
    internal.votes += gain
    internal.totalVotes += gain

    // UI szinkronizálás ~10x/mp (minden 6. frame ~60fps esetén)
    if (frameCount % 6 === 0) {
      syncDisplay()
      checkAchievements()
    }

    // Random event ~8mp-ként
    if (now - lastRandomEvent > 8000) {
      lastRandomEvent = now
      if (Math.random() < 0.3) triggerRandomEvent()
    }

    // Automatikus mentés 30mp-ként
    if (now - lastSave > 30000) {
      lastSave = now
      saveGame()
    }

    rafId = requestAnimationFrame(tick)
  }

  rafId = requestAnimationFrame(tick)
}

export function stopGameLoop() {
  if (rafId) cancelAnimationFrame(rafId)
  rafId = null
}
```

---

## 6. Party adatok — `src/lib/partyData.js`

```javascript
export const TOPICS = [
  { id:'adorendszer',      icon:'💰', name:'Adórendszer',          desc:'Progresszív vs. lapos adó',             axis:'LR',      weight:0.8 },
  { id:'kulföldi_toke',    icon:'🏭', name:'Külföldi tőke',        desc:'Multik vs. hazai KKV-k',                axis:'LR',      weight:0.6 },
  { id:'minimalbér',       icon:'👷', name:'Minimálbér',           desc:'Szakszervezetek és bérek',              axis:'LR',      weight:0.7 },
  { id:'lakhatás',         icon:'🏠', name:'Lakhatás',             desc:'Szociális bérlakás vs. CSOK',           axis:'LR',      weight:0.6 },
  { id:'eu_penzek',        icon:'🇪🇺', name:'EU-s pénzek',          desc:'Átláthatóság vs. szuverén döntés',      axis:'LR',      weight:0.4 },
  { id:'csaladpolitika',   icon:'👨‍👩‍👧', name:'Családpolitika',      desc:'Univerzális vs. házassághoz kötött',   axis:'CL',      weight:0.9 },
  { id:'lmbtq',            icon:'🏳️‍🌈', name:'LMBTQ-jogok',         desc:'Egyenlő jogok vs. hagyomány',          axis:'CL',      weight:1.0 },
  { id:'egyhaz_allam',     icon:'⛪', name:'Egyház és állam',      desc:'Szétválasztás vs. keresztény gyökerek', axis:'CL',      weight:0.8 },
  { id:'egeszsegugy',      icon:'🏥', name:'Egészségügy',          desc:'Állami vs. vegyes finanszírozás',       axis:'LR',      weight:0.7 },
  { id:'oktatas',          icon:'📚', name:'Oktatás',              desc:'Tanári autonómia vs. centralizáció',    axis:'LR',      weight:0.6 },
  { id:'ukrajna_haboru',   icon:'🕊️', name:'Ukrajna / Béke',       desc:'Béke vs. Ukrajna-támogatás',            axis:'SPECIAL', weight:1.0 },
  { id:'eu_integráció',    icon:'🌐', name:'EU-integráció',        desc:'Föderáció vs. nemzetek Európája',       axis:'LR',      weight:0.9 },
  { id:'migráció',         icon:'🚧', name:'Migráció',             desc:'Kerítés vs. befogadás',                 axis:'CL',      weight:1.0 },
  { id:'nato',             icon:'🛡️', name:'NATO / Védelem',       desc:'Szuverenista vs. atlantista',           axis:'SPECIAL', weight:0.7 },
  { id:'sajto',            icon:'📰', name:'Sajtószabadság',       desc:'Médiapiac szabályozása',                axis:'CL',      weight:0.8 },
  { id:'korrupcio',        icon:'⚖️', name:'Korrupcióellenes',     desc:'Független intézmények',                 axis:'LR',      weight:0.7 },
  { id:'valasztasi_reform',icon:'🗳️', name:'Választási reform',    desc:'Rendszer megváltoztatása',              axis:'LR',      weight:0.6 },
  { id:'energiapolitika',  icon:'⚡', name:'Energiapolitika',      desc:'Atomerőmű vs. megújuló',                axis:'CL',      weight:0.7 },
  { id:'kivandorlas',      icon:'✈️', name:'Kivándorlás',          desc:'Megtartó vs. strukturális reform',      axis:'LR',      weight:0.5 },
  { id:'drog_politika',    icon:'💊', name:'Drogpolitika',         desc:'Dekriminalizáció vs. tilalom',          axis:'CL',      weight:0.8 },
]

// Bal és jobb felirat minden témánál
// stance: 'progressive' | 'neutral' | 'conservative'
export const TOPIC_STANCES = {
  adorendszer:      { left:'Progresszív adó',      right:'Lapos adó' },
  kulföldi_toke:    { left:'Hazai KKV-k',           right:'Külföldi tőke' },
  minimalbér:       { left:'Erős szakszervezet',    right:'Piac dönt' },
  lakhatás:         { left:'Szociális bérlakás',    right:'CSOK és piac' },
  eu_penzek:        { left:'Átláthatóság',          right:'Szuverén döntés' },
  csaladpolitika:   { left:'Univerzális juttatás',  right:'Házassághoz kötött' },
  lmbtq:            { left:'Egyenlő jogok',         right:'Hagyományos értékek' },
  egyhaz_allam:     { left:'Szétválasztás',         right:'Keresztény gyökerek' },
  egeszsegugy:      { left:'Állami rendszer',       right:'Vegyes finanszírozás' },
  oktatas:          { left:'Tanári autonómia',      right:'Centralizált tanterv' },
  ukrajna_haboru:   { left:'Ukrajna-támogatás',     right:'Béke / Semlegesség' },
  eu_integráció:    { left:'Mélyebb integráció',    right:'Nemzetek Európája' },
  migráció:         { left:'Befogadás',             right:'Szigorú határvédelem' },
  nato:             { left:'Atlantista',            right:'Szuverenista' },
  sajto:            { left:'Sajtószabadság',        right:'Állami szabályozás' },
  korrupcio:        { left:'Független intézmények', right:'Nemzeti érdek' },
  valasztasi_reform:{ left:'Reform szükséges',      right:'Rendszer jól működik' },
  energiapolitika:  { left:'Megújuló energia',      right:'Atomerőmű / Fosszilis' },
  kivandorlas:      { left:'Strukturális reform',   right:'Megtartó program' },
  drog_politika:    { left:'Dekriminalizáció',      right:'Szigorú tilalom' },
}

export const REFERENCE_PARTIES = [
  { name:'Fidesz',    x: 0.75, y: 0.80, color:'#ff6600' },
  { name:'Tisza',     x: 0.20, y: 0.10, color:'#005ea5' },
  { name:'Mi Hazánk', x: 0.85, y: 0.90, color:'#555555' },
  { name:'DK',        x:-0.50, y:-0.20, color:'#e41e1e' },
  { name:'MKKP',      x:-0.10, y:-0.60, color:'#ff69b4' },
]

export const PARTY_COLORS = [
  '#4a9eff','#e84545','#3db87a','#f0c040',
  '#b06ef3','#ff7043','#26c6da','#ec407a',
]

// ── SZÁMÍTÁSOK ──

export function generateAbbreviation(name) {
  const abbr = name.split(' ')
    .filter(w => w.length > 2)
    .map(w => w[0].toUpperCase())
    .join('')
    .slice(0, 6)
  return abbr || name.slice(0, 3).toUpperCase()
}

// stances: { topicId: 'progressive'|'neutral'|'conservative' }
export function calculateIdeology(selectedTopics, stances) {
  const stanceVal = { progressive: -1, neutral: 0, conservative: 1 }
  let lrSum = 0, lrW = 0, clSum = 0, clW = 0

  selectedTopics.forEach(topicId => {
    const topic = TOPICS.find(t => t.id === topicId)
    if (!topic) return
    const val = stanceVal[stances[topicId] || 'neutral']
    if (topic.axis === 'LR' || topic.axis === 'SPECIAL') {
      lrSum += val * topic.weight
      lrW   += topic.weight
    }
    if (topic.axis === 'CL') {
      clSum += val * topic.weight
      clW   += topic.weight
    }
  })

  return {
    ideologyX: lrW > 0 ? lrSum / lrW : 0,  // -1 bal ... +1 jobb
    ideologyY: clW > 0 ? clSum / clW : 0,  // -1 liberális ... +1 konzervatív
  }
}

export function getArchetype(x, y) {
  if (x < -0.4 && y < -0.3) return { name:'Zöld szociáldemokrata',  desc:'Baloldali, progresszív értékekkel' }
  if (x < -0.4 && y >  0.3) return { name:'Nemzeti baloldali',       desc:'Szociális programok, konzervatív értékek' }
  if (x < -0.4)              return { name:'Szociáldemokrata',        desc:'Gazdasági baloldal, mérsékelt értékek' }
  if (x >  0.4 && y < -0.3) return { name:'Liberális jobboldal',     desc:'Piaci gazdaság, progresszív társadalom' }
  if (x >  0.4 && y >  0.3) return { name:'Nemzeti konzervatív',     desc:'Jobboldali gazdaság, hagyományos értékek' }
  if (x >  0.4)              return { name:'Kereszténydemokrata',     desc:'Jobbközép, mérsékelt értékek' }
  if (y < -0.4)              return { name:'Liberális centrista',     desc:'Középen, progresszív társadalomképpel' }
  if (y >  0.4)              return { name:'Konzervatív centrista',   desc:'Középen, hagyományos értékrenddel' }
  return                            { name:'Néppárti pragmatikus',    desc:'Középutas, gyűjtőpárti jelleg' }
}

export function calculateBonuses(selectedTopics, stances, ideologyX, ideologyY) {
  const b = { urban:1.0, rural:1.0, young:1.0, elderly:1.0, worker:1.0, intellectual:1.0 }
  const s = stances

  if (selectedTopics.includes('lakhatás')   && s.lakhatás   === 'progressive') { b.young  *= 1.25 }
  if (selectedTopics.includes('lmbtq')      && s.lmbtq      === 'progressive') { b.young  *= 1.20; b.urban *= 1.15 }
  if (selectedTopics.includes('migráció')   && s.migráció   === 'conservative'){ b.rural  *= 1.20 }
  if (selectedTopics.includes('minimalbér') && s.minimalbér === 'progressive') { b.worker *= 1.25 }
  if (selectedTopics.includes('egeszsegugy'))                                   { b.elderly*= 1.15 }
  if (selectedTopics.includes('korrupcio'))                                     { b.intellectual *= 1.20 }
  if (selectedTopics.includes('oktatas'))                                       { b.intellectual *= 1.15 }

  if (ideologyX < -0.3) { b.worker *= 1.1;  b.young   *= 1.05 }
  if (ideologyX >  0.3) { b.rural  *= 1.1;  b.elderly *= 1.05 }
  if (ideologyY < -0.3) { b.urban  *= 1.1;  b.intellectual *= 1.1 }
  if (ideologyY >  0.3) { b.rural  *= 1.1;  b.elderly *= 1.1 }

  return b
}

// Emberi leírás a bónuszokhoz
export const BONUS_LABELS = {
  urban:        'Városi szavazók',
  rural:        'Vidéki szavazók',
  young:        'Fiatal szavazók (18–35)',
  elderly:      'Idős szavazók (60+)',
  worker:       'Munkás szavazók',
  intellectual: 'Értelmiségi szavazók',
}
```

---

## 7. Party Store — `src/lib/partyStore.js`

```javascript
import { writable } from 'svelte/store'

const STORAGE_KEY = 'partyConfig'

function createPartyStore() {
  const saved = localStorage.getItem(STORAGE_KEY)
  const initial = saved ? JSON.parse(saved) : null

  const { subscribe, set, update } = writable(initial)

  return {
    subscribe,
    save(config) {
      localStorage.setItem(STORAGE_KEY, JSON.stringify(config))
      set(config)
    },
    reset() {
      localStorage.removeItem(STORAGE_KEY)
      set(null)
    }
  }
}

export const partyStore = createPartyStore()
```

---

## 8. App.svelte — root routing

```svelte
<script>
  import { partyStore } from './lib/partyStore.js'
  import { loadGame, saveGame } from './lib/gameStore.js'
  import { startGameLoop } from './lib/gameLoop.js'
  import FoundingWizard from './components/founding/FoundingWizard.svelte'
  import GameLayout from './components/game/GameLayout.svelte'

  let phase = $partyStore ? 'game' : 'founding'
  let transitioning = false

  function onFoundingComplete(config) {
    partyStore.save(config)
    transitioning = true
    setTimeout(() => {
      phase = 'game'
      transitioning = false
      loadGame()
      startGameLoop()
    }, 400)
  }

  // Ha már van mentett párt, rögtön játék indul
  if (phase === 'game') {
    loadGame()
    startGameLoop()
  }

  // Mentés oldal bezáráskor
  if (typeof window !== 'undefined') {
    window.addEventListener('beforeunload', saveGame)
  }
</script>

{#if phase === 'founding'}
  <div class="screen" class:hiding={transitioning}>
    <FoundingWizard on:complete={e => onFoundingComplete(e.detail)} />
  </div>
{:else}
  <div class="screen" class:fading-in={true}>
    <GameLayout />
  </div>
{/if}

<style>
  .screen { min-height: 100vh; transition: opacity 0.4s ease; }
  .hiding  { opacity: 0; pointer-events: none; }
  .fading-in { animation: fadeIn 0.4s ease; }
  @keyframes fadeIn { from { opacity: 0; } to { opacity: 1; } }
</style>
```

---

## 9. Founding Wizard komponensek

### `FoundingWizard.svelte`

```svelte
<script>
  import { createEventDispatcher } from 'svelte'
  import { PARTY_COLORS, generateAbbreviation, calculateIdeology,
           getArchetype, calculateBonuses } from '../../lib/partyData.js'
  import Step1Name    from './Step1Name.svelte'
  import Step2Topics  from './Step2Topics.svelte'
  import Step3Stances from './Step3Stances.svelte'
  import Step4Summary from './Step4Summary.svelte'

  const dispatch = createEventDispatcher()

  let step = 1
  let draft = {
    name: '',
    abbreviation: '',
    color: PARTY_COLORS[0],
    slogan: '',
    topics: [],
    stances: {},
  }

  function next() { step++ }
  function back() { step-- }

  function complete() {
    const { ideologyX, ideologyY } = calculateIdeology(draft.topics, draft.stances)
    const archetype = getArchetype(ideologyX, ideologyY)
    const startingBonuses = calculateBonuses(draft.topics, draft.stances, ideologyX, ideologyY)

    dispatch('complete', {
      ...draft,
      ideologyX,
      ideologyY,
      archetypeName: archetype.name,
      archetypeDesc: archetype.desc,
      startingBonuses,
      founded: Date.now(),
    })
  }
</script>

<div class="wizard-container">
  <!-- Progress dots -->
  <div class="progress-dots">
    {#each [1,2,3,4] as s}
      <div class="dot" class:active={s === step} class:done={s < step}></div>
    {/each}
  </div>

  {#if step === 1}
    <Step1Name bind:draft on:next={next} />
  {:else if step === 2}
    <Step2Topics bind:draft on:next={next} on:back={back} />
  {:else if step === 3}
    <Step3Stances bind:draft on:next={next} on:back={back} />
  {:else}
    <Step4Summary {draft} on:back={back} on:complete={complete} />
  {/if}
</div>

<style>
  .wizard-container {
    max-width: 700px;
    margin: 0 auto;
    padding: 40px 20px;
    min-height: 100vh;
    display: flex;
    flex-direction: column;
    gap: 32px;
  }
  .progress-dots {
    display: flex;
    justify-content: center;
    gap: 10px;
  }
  .dot {
    width: 8px;
    height: 8px;
    border-radius: 50%;
    background: var(--border);
    transition: background 0.2s;
  }
  .dot.done   { background: var(--muted); }
  .dot.active { background: var(--accent); }
</style>
```

### `Step1Name.svelte`

```svelte
<script>
  import { createEventDispatcher } from 'svelte'
  import { PARTY_COLORS, generateAbbreviation } from '../../lib/partyData.js'

  export let draft
  const dispatch = createEventDispatcher()

  $: if (draft.name) {
    const auto = generateAbbreviation(draft.name)
    // Csak ha a user még nem módosította manuálisan
    if (!draft._abbreviationManual) draft.abbreviation = auto
  }

  $: canNext = draft.name.trim().length >= 3 && draft.abbreviation.trim().length >= 2

  function onAbbrInput(e) {
    draft._abbreviationManual = true
    draft.abbreviation = e.target.value.toUpperCase()
  }
</script>

<div class="step">
  <h1 class="step-title">Alapíts pártot</h1>
  <p class="step-desc">Add meg a pártod nevét és alapvető arculatát.</p>

  <div class="field-group">
    <label class="field-label">Párt neve</label>
    <input
      class="field-input"
      type="text"
      bind:value={draft.name}
      maxlength="40"
      placeholder="Magyar Jövő Pártja"
    />
  </div>

  <div class="field-row">
    <div class="field-group flex-1">
      <label class="field-label">Rövidítés</label>
      <input
        class="field-input mono"
        type="text"
        value={draft.abbreviation}
        on:input={onAbbrInput}
        maxlength="6"
        placeholder="MJP"
      />
    </div>
  </div>

  <div class="field-group">
    <label class="field-label">Párszín</label>
    <div class="color-grid">
      {#each PARTY_COLORS as c}
        <button
          class="color-swatch"
          class:selected={draft.color === c}
          style="background: {c}"
          on:click={() => draft.color = c}
          aria-label={c}
        />
      {/each}
    </div>
  </div>

  <div class="field-group">
    <label class="field-label">Jelszó <span class="optional">(opcionális)</span></label>
    <input
      class="field-input"
      type="text"
      bind:value={draft.slogan}
      maxlength="80"
      placeholder="Változást akarunk!"
    />
  </div>

  <div class="step-actions">
    <button class="btn-primary" disabled={!canNext} on:click={() => dispatch('next')}>
      Tovább →
    </button>
  </div>
</div>

<style>
  .step { display: flex; flex-direction: column; gap: 20px; }
  .step-title { font-family: var(--mono); color: var(--accent); font-size: 22px; font-weight: 600; }
  .step-desc { color: var(--muted); font-size: 14px; }
  .field-group { display: flex; flex-direction: column; gap: 6px; }
  .field-row { display: flex; gap: 16px; }
  .flex-1 { flex: 1; }
  .field-label { font-family: var(--mono); font-size: 11px; color: var(--muted); text-transform: uppercase; letter-spacing: 0.08em; }
  .optional { font-size: 10px; color: var(--border); text-transform: none; }
  .field-input {
    background: var(--surface2);
    border: 1px solid var(--border);
    border-radius: 6px;
    color: var(--white);
    font-family: var(--sans);
    font-size: 14px;
    padding: 10px 12px;
    outline: none;
    transition: border-color 0.2s;
  }
  .field-input:focus { border-color: var(--accent); }
  .field-input.mono { font-family: var(--mono); text-transform: uppercase; }
  .color-grid { display: flex; gap: 8px; flex-wrap: wrap; }
  .color-swatch {
    width: 28px; height: 28px; border-radius: 50%;
    border: 2px solid transparent;
    cursor: pointer; transition: transform 0.15s, border-color 0.15s;
  }
  .color-swatch:hover { transform: scale(1.15); }
  .color-swatch.selected { border-color: var(--white); transform: scale(1.2); }
  .step-actions { display: flex; justify-content: flex-end; padding-top: 8px; }
  .btn-primary {
    background: var(--accent); color: var(--bg);
    font-family: var(--mono); font-weight: 600;
    border: none; border-radius: 6px;
    padding: 10px 24px; cursor: pointer;
    font-size: 14px; transition: opacity 0.15s;
  }
  .btn-primary:disabled { opacity: 0.35; cursor: not-allowed; }
</style>
```

### `Step2Topics.svelte`

```svelte
<script>
  import { createEventDispatcher } from 'svelte'
  import { TOPICS } from '../../lib/partyData.js'

  export let draft
  const dispatch = createEventDispatcher()

  $: canNext = draft.topics.length === 5

  function toggleTopic(id) {
    if (draft.topics.includes(id)) {
      draft.topics = draft.topics.filter(t => t !== id)
    } else if (draft.topics.length < 5) {
      draft.topics = [...draft.topics, id]
    }
  }
</script>

<div class="step">
  <h2 class="step-title">Válassz 5 témát</h2>
  <p class="step-desc">Ezek határozzák meg a pártod ideológiai pozícióját és szavazóbázisát.</p>

  <div class="counter" class:full={canNext}>
    {draft.topics.length} / 5 téma kiválasztva
  </div>

  <div class="topics-grid">
    {#each TOPICS as topic}
      {@const selected = draft.topics.includes(topic.id)}
      {@const disabled = !selected && draft.topics.length >= 5}
      <button
        class="topic-card"
        class:selected
        class:disabled
        on:click={() => toggleTopic(topic.id)}
        {disabled}
      >
        <span class="topic-icon">{topic.icon}</span>
        <span class="topic-name">{topic.name}</span>
        <span class="topic-desc">{topic.desc}</span>
      </button>
    {/each}
  </div>

  <div class="step-actions">
    <button class="btn-secondary" on:click={() => dispatch('back')}>← Vissza</button>
    <button class="btn-primary" disabled={!canNext} on:click={() => dispatch('next')}>Tovább →</button>
  </div>
</div>

<style>
  .step { display: flex; flex-direction: column; gap: 20px; }
  .step-title { font-family: var(--mono); color: var(--accent); font-size: 20px; font-weight: 600; }
  .step-desc  { color: var(--muted); font-size: 14px; }
  .counter { font-family: var(--mono); font-size: 13px; color: var(--muted); }
  .counter.full { color: var(--accent); }
  .topics-grid {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 10px;
  }
  @media (max-width: 540px) { .topics-grid { grid-template-columns: repeat(2, 1fr); } }
  .topic-card {
    background: var(--surface);
    border: 1px solid var(--border);
    border-radius: 8px;
    padding: 12px;
    cursor: pointer;
    text-align: left;
    display: flex; flex-direction: column; gap: 4px;
    transition: border-color 0.15s, background 0.15s;
  }
  .topic-card:hover:not(.disabled) { border-color: var(--muted); }
  .topic-card.selected { border-color: var(--accent); background: rgba(240,192,64,0.08); }
  .topic-card.disabled { opacity: 0.4; cursor: not-allowed; }
  .topic-icon { font-size: 20px; }
  .topic-name { font-family: var(--mono); font-size: 12px; font-weight: 600; color: var(--white); }
  .topic-desc { font-size: 11px; color: var(--muted); }
  .step-actions { display: flex; justify-content: space-between; padding-top: 8px; }
  .btn-primary  { background: var(--accent); color: var(--bg); font-family: var(--mono); font-weight: 600; border: none; border-radius: 6px; padding: 10px 24px; cursor: pointer; font-size: 14px; }
  .btn-primary:disabled { opacity: 0.35; cursor: not-allowed; }
  .btn-secondary { background: var(--surface2); color: var(--muted); font-family: var(--mono); border: 1px solid var(--border); border-radius: 6px; padding: 10px 20px; cursor: pointer; font-size: 14px; }
</style>
```

### `Step3Stances.svelte`

```svelte
<script>
  import { createEventDispatcher } from 'svelte'
  import { TOPICS, TOPIC_STANCES } from '../../lib/partyData.js'

  export let draft
  const dispatch = createEventDispatcher()

  // Alapértelmezett: minden semleges
  draft.topics.forEach(id => {
    if (!draft.stances[id]) draft.stances[id] = 'neutral'
  })

  function setStance(topicId, stance) {
    draft.stances = { ...draft.stances, [topicId]: stance }
  }

  $: selectedTopics = TOPICS.filter(t => draft.topics.includes(t.id))
</script>

<div class="step">
  <h2 class="step-title">Álláspontok</h2>
  <p class="step-desc">Határozd meg a pártod pozícióját minden kiválasztott témában.</p>

  <div class="stances-list">
    {#each selectedTopics as topic}
      {@const stances = TOPIC_STANCES[topic.id]}
      {@const current = draft.stances[topic.id] || 'neutral'}
      <div class="stance-row">
        <div class="stance-topic">
          <span class="stance-icon">{topic.icon}</span>
          <span class="stance-name">{topic.name}</span>
        </div>
        <div class="stance-toggle">
          <button
            class="toggle-btn"
            class:active={current === 'progressive'}
            on:click={() => setStance(topic.id, 'progressive')}
          >{stances.left}</button>
          <button
            class="toggle-btn neutral-btn"
            class:active={current === 'neutral'}
            on:click={() => setStance(topic.id, 'neutral')}
          >Semleges</button>
          <button
            class="toggle-btn"
            class:active={current === 'conservative'}
            on:click={() => setStance(topic.id, 'conservative')}
          >{stances.right}</button>
        </div>
      </div>
    {/each}
  </div>

  <div class="step-actions">
    <button class="btn-secondary" on:click={() => dispatch('back')}>← Vissza</button>
    <button class="btn-primary" on:click={() => dispatch('next')}>Tovább →</button>
  </div>
</div>

<style>
  .step { display: flex; flex-direction: column; gap: 20px; }
  .step-title { font-family: var(--mono); color: var(--accent); font-size: 20px; font-weight: 600; }
  .step-desc  { color: var(--muted); font-size: 14px; }
  .stances-list { display: flex; flex-direction: column; gap: 14px; }
  .stance-row {
    background: var(--surface);
    border: 1px solid var(--border);
    border-radius: 8px;
    padding: 14px;
    display: flex; flex-direction: column; gap: 10px;
  }
  .stance-topic { display: flex; align-items: center; gap: 8px; }
  .stance-icon  { font-size: 18px; }
  .stance-name  { font-family: var(--mono); font-size: 13px; font-weight: 600; color: var(--white); }
  .stance-toggle { display: flex; gap: 6px; }
  .toggle-btn {
    flex: 1; padding: 6px 4px;
    background: var(--surface2); border: 1px solid var(--border); border-radius: 5px;
    color: var(--muted); font-size: 11px; cursor: pointer;
    transition: background 0.15s, border-color 0.15s, color 0.15s;
    text-align: center;
  }
  .toggle-btn:hover { border-color: var(--muted); color: var(--text); }
  .toggle-btn.active { background: rgba(240,192,64,0.15); border-color: var(--accent); color: var(--accent); }
  .neutral-btn.active { background: rgba(107,107,128,0.2); border-color: var(--muted); color: var(--muted); }
  .step-actions { display: flex; justify-content: space-between; padding-top: 8px; }
  .btn-primary  { background: var(--accent); color: var(--bg); font-family: var(--mono); font-weight: 600; border: none; border-radius: 6px; padding: 10px 24px; cursor: pointer; font-size: 14px; }
  .btn-secondary { background: var(--surface2); color: var(--muted); font-family: var(--mono); border: 1px solid var(--border); border-radius: 6px; padding: 10px 20px; cursor: pointer; font-size: 14px; }
</style>
```

### `Step4Summary.svelte`

```svelte
<script>
  import { createEventDispatcher } from 'svelte'
  import { TOPICS, REFERENCE_PARTIES, calculateIdeology, getArchetype,
           calculateBonuses, BONUS_LABELS } from '../../lib/partyData.js'

  export let draft
  const dispatch = createEventDispatcher()

  $: ({ ideologyX, ideologyY } = calculateIdeology(draft.topics, draft.stances))
  $: archetype = getArchetype(ideologyX, ideologyY)
  $: bonuses   = calculateBonuses(draft.topics, draft.stances, ideologyX, ideologyY)
  $: activeBonuses = Object.entries(bonuses).filter(([, v]) => v > 1.005)

  // SVG ideológiai térkép — koordináta konverzió
  // ideologyX: -1..+1 → SVG x: 10..190
  // ideologyY: -1..+1 → SVG y: 10..190 (fordítva: -1 liberális = fent)
  function toSvgX(x) { return 10 + (x + 1) / 2 * 180 }
  function toSvgY(y) { return 10 + (y + 1) / 2 * 180 }  // +1 konzervatív = lent

  let founded = false
  function handleComplete() {
    if (founded) return
    founded = true
    dispatch('complete')
  }
</script>

<div class="step">
  <h2 class="step-title">A pártod</h2>

  <div class="party-header" style="border-left: 4px solid {draft.color}">
    <div class="party-abbr" style="color: {draft.color}">{draft.abbreviation}</div>
    <div>
      <div class="party-full-name">{draft.name}</div>
      {#if draft.slogan}<div class="party-slogan">„{draft.slogan}"</div>{/if}
    </div>
  </div>

  <div class="summary-grid">
    <!-- Ideológiai térkép -->
    <div class="map-card">
      <div class="card-label">Ideológiai pozíció</div>
      <svg viewBox="0 0 200 200" class="ideology-map">
        <!-- Negyedek -->
        <rect x="10" y="10" width="90" height="90" fill="rgba(74,158,255,0.08)" rx="2"/>
        <rect x="100" y="10" width="90" height="90" fill="rgba(61,184,122,0.08)" rx="2"/>
        <rect x="10" y="100" width="90" height="90" fill="rgba(176,110,243,0.08)" rx="2"/>
        <rect x="100" y="100" width="90" height="90" fill="rgba(232,69,69,0.08)" rx="2"/>
        <!-- Tengelyek -->
        <line x1="100" y1="10" x2="100" y2="190" stroke="#2a2a3a" stroke-width="1"/>
        <line x1="10" y1="100" x2="190" y2="100" stroke="#2a2a3a" stroke-width="1"/>
        <!-- Feliiratok -->
        <text x="20"  y="8"   fill="#6b6b80" font-size="8" font-family="monospace">Bal</text>
        <text x="165" y="8"   fill="#6b6b80" font-size="8" font-family="monospace">Jobb</text>
        <text x="12"  y="22"  fill="#6b6b80" font-size="7" font-family="monospace">Lib.</text>
        <text x="12"  y="196" fill="#6b6b80" font-size="7" font-family="monospace">Konz.</text>
        <!-- Referencia pártok -->
        {#each REFERENCE_PARTIES as rp}
          <circle cx={toSvgX(rp.x)} cy={toSvgY(rp.y)} r="4" fill={rp.color} opacity="0.7"/>
          <text x={toSvgX(rp.x) + 6} y={toSvgY(rp.y) + 3} fill={rp.color} font-size="7" opacity="0.8">{rp.name}</text>
        {/each}
        <!-- A te pontod -->
        <circle cx={toSvgX(ideologyX)} cy={toSvgY(ideologyY)} r="7"
          fill={draft.color} stroke="white" stroke-width="1.5"/>
      </svg>
    </div>

    <!-- Archetípus + bónuszok -->
    <div class="info-col">
      <div class="archetype-card">
        <div class="card-label">Párttípus</div>
        <div class="archetype-name">{archetype.name}</div>
        <div class="archetype-desc">{archetype.desc}</div>
      </div>

      {#if activeBonuses.length > 0}
        <div class="bonuses-card">
          <div class="card-label">Kezdő előnyök</div>
          {#each activeBonuses as [key, val]}
            <div class="bonus-row">
              <span class="bonus-label">{BONUS_LABELS[key]}</span>
              <span class="bonus-val">+{Math.round((val - 1) * 100)}%</span>
            </div>
          {/each}
        </div>
      {/if}
    </div>
  </div>

  <!-- Választott témák összefoglalója -->
  <div class="topics-summary">
    <div class="card-label">Kampánytémák</div>
    <div class="topic-pills">
      {#each draft.topics as topicId}
        {@const t = TOPICS.find(x => x.id === topicId)}
        {#if t}
          <span class="topic-pill">{t.icon} {t.name}</span>
        {/if}
      {/each}
    </div>
  </div>

  <div class="step-actions">
    <button class="btn-secondary" on:click={() => dispatch('back')}>← Vissza</button>
    <button class="btn-found" disabled={founded} on:click={handleComplete}>
      🇭🇺 Párt megalapítása
    </button>
  </div>
</div>

<style>
  .step { display: flex; flex-direction: column; gap: 20px; }
  .step-title { font-family: var(--mono); color: var(--accent); font-size: 20px; font-weight: 600; }
  .party-header { display: flex; align-items: center; gap: 14px; padding: 14px; background: var(--surface); border-radius: 8px; }
  .party-abbr { font-family: var(--mono); font-size: 28px; font-weight: 600; min-width: 56px; text-align: center; }
  .party-full-name { font-size: 16px; font-weight: 600; color: var(--white); }
  .party-slogan { font-size: 12px; color: var(--muted); font-style: italic; margin-top: 2px; }
  .summary-grid { display: grid; grid-template-columns: 200px 1fr; gap: 14px; }
  @media (max-width: 480px) { .summary-grid { grid-template-columns: 1fr; } }
  .map-card { background: var(--surface); border: 1px solid var(--border); border-radius: 8px; padding: 12px; }
  .ideology-map { width: 100%; max-width: 200px; display: block; }
  .info-col { display: flex; flex-direction: column; gap: 10px; }
  .archetype-card, .bonuses-card {
    background: var(--surface); border: 1px solid var(--border); border-radius: 8px; padding: 12px;
    display: flex; flex-direction: column; gap: 6px;
  }
  .card-label { font-family: var(--mono); font-size: 10px; color: var(--muted); text-transform: uppercase; letter-spacing: 0.1em; margin-bottom: 4px; }
  .archetype-name { font-family: var(--mono); font-size: 14px; font-weight: 600; color: var(--accent); }
  .archetype-desc { font-size: 12px; color: var(--muted); }
  .bonus-row { display: flex; justify-content: space-between; align-items: center; font-size: 12px; }
  .bonus-label { color: var(--text); }
  .bonus-val { color: var(--green); font-family: var(--mono); font-weight: 600; }
  .topics-summary { background: var(--surface); border: 1px solid var(--border); border-radius: 8px; padding: 12px; }
  .topic-pills { display: flex; flex-wrap: wrap; gap: 6px; margin-top: 8px; }
  .topic-pill { background: var(--surface2); border: 1px solid var(--border); border-radius: 20px; padding: 3px 10px; font-size: 12px; color: var(--text); }
  .step-actions { display: flex; justify-content: space-between; padding-top: 8px; }
  .btn-secondary { background: var(--surface2); color: var(--muted); font-family: var(--mono); border: 1px solid var(--border); border-radius: 6px; padding: 10px 20px; cursor: pointer; font-size: 14px; }
  .btn-found {
    background: var(--accent); color: var(--bg);
    font-family: var(--mono); font-weight: 600; font-size: 15px;
    border: none; border-radius: 6px; padding: 12px 28px; cursor: pointer;
    transition: opacity 0.15s;
  }
  .btn-found:disabled { opacity: 0.4; cursor: not-allowed; }
</style>
```

---

## 10. Game komponensek migrációja

### `GameLayout.svelte`

A meglévő 3-oszlopos HTML layout Svelte-be emelve. A `<Header>`, `<Ticker>`, `<ClickerPanel>`, `<BuildingsPanel>`, `<UpgradesPanel>`, `<EventsPanel>` komponensek mindegyike a megfelelő részt kapja.

**Fontos:** A meglévő CSS osztályok (`layout`, `panel`, `left-panel`, stb.) **változatlanul** maradnak — csak a HTML kerül `.svelte` fájlokba.

### `ClickerPanel.svelte` — a kattintás javítása

A Svelte-ben a kattintás problémája **megoldódik** természetesen, mert nem rebuild-eljük a DOM-ot tick-enként — a reaktivitás csak a változott értékeket frissíti.

```svelte
<script>
  import { display, doClick, fmt } from '../../lib/gameStore.js'
  import { MILESTONES } from '../../lib/gameData.js'
  import { partyStore } from '../../lib/partyStore.js'

  let floats = []
  let nextId = 0

  function handleClick(e) {
    doClick()
    // Float text
    const id = nextId++
    floats = [...floats, { id, x: e.clientX - 20, y: e.clientY - 10 }]
    setTimeout(() => { floats = floats.filter(f => f.id !== id) }, 900)
  }

  $: milestone = (() => {
    let cur = MILESTONES[0], nxt = MILESTONES[1]
    for (let i = 0; i < MILESTONES.length; i++) {
      if ($display.totalVotes >= MILESTONES[i].votes) {
        cur = MILESTONES[i]
        nxt = MILESTONES[i+1] || null
      }
    }
    const pct = nxt
      ? Math.min(100, (($display.totalVotes - cur.votes) / (nxt.votes - cur.votes)) * 100)
      : 100
    return { cur, nxt, pct }
  })()
</script>

<!-- Float texts (fixed overlay) -->
{#each floats as f (f.id)}
  <div class="float-text" style="left:{f.x}px; top:{f.y}px">
    +{fmt($display.clickPower)} 🗳
  </div>
{/each}

<div class="left-panel panel">
  <input class="party-name-input" type="text"
    value={$partyStore?.name || 'Pártom'}
    readonly
  />

  <div class="votes-big">
    <div class="votes-num">{fmt($display.votes)}</div>
    <div class="votes-sub">szavazat</div>
    <div class="votes-sub" style="color:var(--accent); margin-top:4px">
      {fmt($display.vps)} szavazat/mp
    </div>
  </div>

  <button class="clicker-btn" on:click={handleClick}>
    <span class="clicker-icon">🤝</span>
    <span class="clicker-label">Kezet fog<br>a választóval</span>
  </button>

  <div class="milestone-box">
    <div class="milestone-label">JELENLEGI TITULUS</div>
    <div class="milestone-title">{milestone.cur.title}</div>
    <div class="milestone-bar-wrap">
      <div class="milestone-bar" style="width:{milestone.pct}%"></div>
    </div>
    <div class="votes-sub" style="margin-top:5px; font-size:11px; color:var(--muted)">
      {milestone.nxt
        ? 'Következő: ' + fmt(milestone.nxt.votes) + ' szavazat → ' + milestone.nxt.title
        : 'Maximum elérve!'}
    </div>
  </div>
</div>

<style>
  /* Float text — fixed overlay */
  :global(.float-text) {
    position: fixed;
    font-family: var(--mono);
    font-size: 13px;
    font-weight: 600;
    color: var(--accent);
    pointer-events: none;
    z-index: 999;
    animation: floatUp 0.9s ease-out forwards;
  }
  @keyframes floatUp {
    from { opacity: 1; transform: translateY(0); }
    to   { opacity: 0; transform: translateY(-60px); }
  }
</style>
```

### `BuildingsPanel.svelte`

```svelte
<script>
  import { display, buyBuilding, fmt, bldCost } from '../../lib/gameStore.js'
  import { BUILDINGS } from '../../lib/gameData.js'
</script>

<div class="panel mid-panel">
  <div class="panel-title">🏗 Kampányinfrastruktúra</div>
  {#each BUILDINGS as b}
    {@const cost = bldCost(b.id)}
    {@const owned = $display.owned[b.id] || 0}
    {@const canBuy = $display.votes >= cost}
    <div
      class="building-row"
      class:can-buy={canBuy}
      class:disabled={!canBuy}
      on:click={() => canBuy && buyBuilding(b.id)}
    >
      <div class="building-icon">{b.icon}</div>
      <div class="building-info">
        <div class="building-name">{b.name}</div>
        <div class="building-desc">{b.desc}</div>
      </div>
      <div>
        <div class="building-cost">🗳 {fmt(cost)}</div>
        <div class="building-owned">Van: {owned}</div>
      </div>
    </div>
  {/each}
</div>
```

### `UpgradesPanel.svelte` és `EventsPanel.svelte`

Ugyanilyen mintát követnek — a meglévő renderUpgrades() és renderEvents() logika `{#each}` loopokká válik, az event handler-ek `on:click`-ké, a className-ek `class:` direktívákká.

---

## 11. Persistence — mentés/betöltés integrálása

A `partyStore` és a `gameStore` egyaránt localStorage-ba ment:

| Key | Tartalom |
|---|---|
| `partyConfig` | A pártalapítás eredménye |
| `gameState` | A játék állása (votes, buildings, stb.) |

Az `App.svelte` indulásakor:
1. `partyStore`-t ellenőrzi → van-e már párt?
2. Ha igen: `loadGame()` → `startGameLoop()`
3. Ha nem: `FoundingWizard` jelenik meg

**"Új játék" reset:**
```svelte
<!-- GameLayout.svelte-be kerül valahova diszkréten -->
<button on:click={() => { partyStore.reset(); localStorage.removeItem('gameState'); location.reload() }}>
  Új játék
</button>
```

---

## 12. Sorrend amit Claude Code-nak követni kell

**1.** Vite + Svelte projekt létrehozása (`npm create vite@latest`)

**2.** CSS átmásolása `src/styles/global.css`-be a HTML-ből — semmi nem változik

**3.** `src/lib/gameData.js` létrehozása — a konstansok szó szerint másolva

**4.** `src/lib/gameStore.js` létrehozása — a fenti specifikáció szerint

**5.** `src/lib/gameLoop.js` létrehozása

**6.** `src/lib/partyData.js` és `src/lib/partyStore.js`

**7.** `src/App.svelte` — routing logika

**8.** Founding wizard komponensek (Step1–Step4 + FoundingWizard wrapper)

**9.** Game komponensek migrálása — `GameLayout`, `Header`, `Ticker`, `ClickerPanel`, `BuildingsPanel`, `UpgradesPanel`, `EventsPanel`

**10.** Tesztelés: `npm run dev` → játék fut, pártalapítás működik, kattintás bug nincs

**11.** `npm run build` → `dist/` mappa létrejön, `dist/index.html` megnyitható

---

## 13. Amit NEM szabad megváltoztatni

- Az összes CSS class neve (`.building-row`, `.can-buy`, `.panel-title`, stb.)
- A játék logikája (bldCost képlet, vps számítás, upgrade szorzók)
- A meglévő adatok (BUILDINGS, UPGRADES, MILESTONES, NEWS, ACHIEVEMENTS)
- A vizuális megjelenés — pixel-pontosan ugyanolyannak kell kinéznie mint a jelenlegi HTML

---

## 14. Tesztelési ellenőrzőlista

**Svelte migráció:**
- [ ] `npm run dev` hibátlanul indul
- [ ] `npm run build` hibátlanul lefut
- [ ] A játék vizuálisan identikus a régi HTML-lel
- [ ] Épület vásárlás működik (kattintás regisztrál, szavazat csökken, owned nő)
- [ ] Fejlesztés vásárlás működik
- [ ] Szavazat/mp automatikusan növekszik
- [ ] Random események megjelennek a hírek panelben
- [ ] Achievements unlockol

**Pártalapítás:**
- [ ] Első betöltés: wizard jelenik meg
- [ ] Wizard: 4 lépés váltakozik
- [ ] Step 1: validáció (min 3 karakter), rövidítés auto-generálódik
- [ ] Step 2: pontosan 5 téma választható
- [ ] Step 3: minden témánál toggle működik, semleges az alap
- [ ] Step 4: ideológiai pont helyes pozícióban az SVG-n
- [ ] Step 4: referencia pártok láthatók a térképen
- [ ] Step 4: archetípus helyes
- [ ] Step 4: csak > 1.0 bónuszok jelennek meg
- [ ] Alapítás gomb → animált átmenet → játék indul
- [ ] LocalStorage: partyConfig mentve, gameState mentve
- [ ] Visszatöltés: wizard nem jelenik meg, játék folytatódik
- [ ] Párt neve megjelenik a headerben / clicker panelben
