# Magyar Politika Idle — Fázis 1 fejlesztési specifikáció
## Pártalapítási képernyő

---

## Kontextus és fájlok

A projekt egyetlen önálló HTML fájl: `magyar-politika-idle.html`
Nincs build rendszer, nincs npm, nincs framework — pure vanilla JS + HTML + CSS.
A meglévő fájl stílusa: IBM Plex Mono + IBM Plex Sans, sötét téma CSS változókkal.

**Meglévő CSS változók (ne változtasd meg):**
```css
--bg: #0e0e12
--surface: #16161d
--surface2: #1e1e28
--border: #2a2a3a
--red: #ce2939
--green: #3a7d44
--white: #e8e8e8
--muted: #6b6b80
--accent: #f0c040
--text: #d0d0d8
--mono: 'IBM Plex Mono', monospace
--sans: 'IBM Plex Sans', sans-serif
```

---

## Mit kell megépíteni

**Egyetlen új funkció:** pártalapítási intro képernyő, ami a meglévő játék ELŐL fut le. Ha a játékos már alapított pártot (localStorage), a képernyő kihagyódik és a játék rögtön indul.

A képernyő 4 lépésből áll. Minden lépés egy `<div>` ami show/hide logikával váltakozik — nincs routing, nincs SPA framework.

---

## Adatstruktúra — amit a pártalapítás létrehoz

```javascript
// Ezt kell localStorage-ba menteni és a játék state-be integrálni
const partyConfig = {
  name: "",           // string, max 40 karakter
  abbreviation: "",   // string, max 6 karakter, auto-generált de szerkeszthető
  color: "#4a9eff",   // hex string, a játékos választja
  founded: Date.now(),
  topics: [],         // pontosan 5 string a TOPICS listából
  stances: {},        // { topicId: "progressive"|"neutral"|"conservative" }
  ideologyX: 0,       // -1.0 (bal) ... +1.0 (jobb), kiszámított
  ideologyY: 0,       // -1.0 (liberális) ... +1.0 (konzervatív), kiszámított
  archetypeName: "",  // kiszámított, pl. "Néppárti pragmatikus"
  startingBonuses: {} // kiszámított, pl. { urban: 1.2, young: 0.9 }
}
```

---

## A 4 lépés részletesen

---

### 1. lépés — Párt neve és jelszó

**UI elemek:**
- Nagy fejléc szöveg: `"Alapíts pártot"` (mono font, accent szín)
- Szöveg input: `"A párt neve"` — placeholder: `"Magyar Jövő Pártja"`, maxlength: 40
- Szöveg input: `"Rövidítés"` — placeholder: `"MJP"`, maxlength: 6, auto-fill a névből (első betűk)
- Szín-választó: 8 előre definiált szín gombok (kör alakú, 28px, border ha selected)
  ```
  Választható színek: #4a9eff, #e84545, #3db87a, #f0c040, #b06ef3, #ff7043, #26c6da, #ec407a
  ```
- Jelmondat input (opcionális): `"Jelszó (opcionális)"`, maxlength: 80, placeholder: `"Változást akarunk!"`
- Tovább gomb — csak ha a párt neve ki van töltve (min 3 karakter)

**Validáció:**
- Párt neve: min 3, max 40 karakter
- Rövidítés: min 2, max 6 karakter, csak nagybetű (auto-uppercase)
- Ha a párt neve változik, a rövidítés auto-frissül (de szerkeszthető marad)

**Auto-rövidítés logika:**
```javascript
function generateAbbreviation(name) {
  return name.split(' ')
    .filter(w => w.length > 2)
    .map(w => w[0].toUpperCase())
    .join('')
    .slice(0, 6) || name.slice(0, 3).toUpperCase()
}
```

---

### 2. lépés — 5 téma kiválasztása

**UI:** Rácsos elrendezés (3 oszlop desktop, 2 oszlop mobil). Minden kártya kattintható.

**Instrukció szöveg:** `"Válassz 5 témát — ezek határozzák meg a pártod ideológiai pozícióját"`

**Téma kártya megjelenése:**
- Ikon + Név (félkövér) + rövid alcím
- Alapállapot: `--surface` háttér, `--border` keret
- Kiválasztott: `--accent` keret (2px), halvány accent háttér (`rgba(240,192,64,0.08)`)
- Ha már 5 van kiválasztva és ez nincs köztük: `opacity: 0.4`, `pointer-events: none` (de meglévő kijelölés törölhető)
- Számláló a tetején: `"3 / 5 téma kiválasztva"`
- Tovább gomb: csak ha pontosan 5 van kiválasztva

**A 20 téma teljes listája:**

```javascript
const TOPICS = [
  // GAZDASÁGI
  { id: "adorendszer",       icon: "💰", name: "Adórendszer",          desc: "Progresszív vs. lapos adó",         axis: "LR", weight: 0.8 },
  { id: "kulföldi_toke",     icon: "🏭", name: "Külföldi tőke",        desc: "Multik vs. hazai KKV-k",            axis: "LR", weight: 0.6 },
  { id: "minimalbér",        icon: "👷", name: "Minimálbér",           desc: "Szakszervezetek és bérek",          axis: "LR", weight: 0.7 },
  { id: "lakhatás",          icon: "🏠", name: "Lakhatás",             desc: "Szociális bérlakás vs. CSOK",       axis: "LR", weight: 0.6 },
  { id: "eu_penzek",         icon: "🇪🇺", name: "EU-s pénzek",          desc: "Átláthatóság vs. szuverén döntés",  axis: "LR", weight: 0.4 },

  // TÁRSADALMI
  { id: "csaladpolitika",    icon: "👨‍👩‍👧", name: "Családpolitika",      desc: "Univerzális vs. házassághoz kötött", axis: "CL", weight: 0.9 },
  { id: "lmbtq",             icon: "🏳️‍🌈", name: "LMBTQ-jogok",          desc: "Egyenlő jogok vs. hagyomány",       axis: "CL", weight: 1.0 },
  { id: "egyhaz_allam",      icon: "⛪", name: "Egyház és állam",      desc: "Szétválasztás vs. keresztény gyökerek", axis: "CL", weight: 0.8 },
  { id: "egeszsegugy",       icon: "🏥", name: "Egészségügy",          desc: "Állami vs. vegyes finanszírozás",   axis: "LR", weight: 0.7 },
  { id: "oktatas",           icon: "📚", name: "Oktatás",              desc: "Tanári autonómia vs. centralizáció",axis: "LR", weight: 0.6 },

  // KÜLPOLITIKA
  { id: "ukrajna_haboru",    icon: "🕊️", name: "Ukrajna / Béke",       desc: "Béke vs. Ukrajna-támogatás",        axis: "SPECIAL", weight: 1.0 },
  { id: "eu_integráció",     icon: "🌐", name: "EU-integráció",        desc: "Föderáció vs. nemzetek Európája",   axis: "LR", weight: 0.9 },
  { id: "migráció",          icon: "🚧", name: "Migráció",             desc: "Kerítés vs. befogadás",             axis: "CL", weight: 1.0 },
  { id: "nato",              icon: "🛡️", name: "NATO / Védelem",       desc: "Szuverenista vs. atlantista",       axis: "SPECIAL", weight: 0.7 },

  // KÖZÉLET
  { id: "sajto",             icon: "📰", name: "Sajtószabadság",       desc: "Médiapiac szabályozása",            axis: "CL", weight: 0.8 },
  { id: "korrupcio",         icon: "⚖️", name: "Korrupcióellenes",     desc: "Független intézmények",             axis: "LR", weight: 0.7 },
  { id: "valasztasi_reform", icon: "🗳️", name: "Választási reform",    desc: "Rendszer megváltoztatása",          axis: "LR", weight: 0.6 },
  { id: "energiapolitika",   icon: "⚡", name: "Energiapolitika",      desc: "Atomerőmű vs. megújuló",            axis: "CL", weight: 0.7 },
  { id: "kivandorlas",       icon: "✈️", name: "Kivándorlás",          desc: "Megtartó vs. strukturális reform",  axis: "LR", weight: 0.5 },
  { id: "drog_politika",     icon: "💊", name: "Drogpolitika",         desc: "Dekriminalizáció vs. tilalom",      axis: "CL", weight: 0.8 },
]
```

---

### 3. lépés — Álláspontok megadása

**UI:** Az 5 kiválasztott téma egymás alatt, mindegyik mellett egy 3-opciós toggle.

**Instrukció szöveg:** `"Határozd meg a pártod álláspontját minden témában"`

Minden témának van egy bal és jobb felirata és egy semleges középső opció:

```javascript
const TOPIC_STANCES = {
  adorendszer:     { left: "Progresszív adó",    right: "Lapos adó",           leftLabel: "🔴 Bal",    rightLabel: "🔵 Jobb" },
  kulföldi_toke:   { left: "Hazai KKV-k",        right: "Külföldi tőke",       leftLabel: "🔴 Bal",    rightLabel: "🔵 Jobb" },
  minimalbér:      { left: "Erős szakszervezet",  right: "Piac dönt",           leftLabel: "🔴 Bal",    rightLabel: "🔵 Jobb" },
  lakhatás:        { left: "Szociális bérlakás",  right: "CSOK és piac",        leftLabel: "🔴 Bal",    rightLabel: "🔵 Jobb" },
  eu_penzek:       { left: "Átláthatóság",        right: "Szuverén döntés",     leftLabel: "🔵 Lib",    rightLabel: "🔴 Konz" },
  csaladpolitika:  { left: "Univerzális juttatás",right: "Házassághoz kötött",  leftLabel: "🔵 Lib",    rightLabel: "🔴 Konz" },
  lmbtq:           { left: "Egyenlő jogok",       right: "Hagyományos értékek", leftLabel: "🔵 Lib",    rightLabel: "🔴 Konz" },
  egyhaz_allam:    { left: "Szétválasztás",       right: "Keresztény gyökerek", leftLabel: "🔵 Lib",    rightLabel: "🔴 Konz" },
  egeszsegugy:     { left: "Állami rendszer",     right: "Vegyes finanszírozás",leftLabel: "🔴 Bal",    rightLabel: "🔵 Jobb" },
  oktatas:         { left: "Tanári autonómia",    right: "Centralizált tanterv",leftLabel: "🔵 Lib",    rightLabel: "🔴 Konz" },
  ukrajna_haboru:  { left: "Ukrajna-támogatás",   right: "Béke / Semlegesség",  leftLabel: "🌐 Atl",    rightLabel: "🏠 Szuv" },
  eu_integráció:   { left: "Mélyebb integráció",  right: "Nemzetek Európája",   leftLabel: "🌐 Pro-EU", rightLabel: "🏠 Szuv" },
  migráció:        { left: "Befogadás",           right: "Szigorú határvédelem",leftLabel: "🔵 Lib",    rightLabel: "🔴 Konz" },
  nato:            { left: "Atlantista",          right: "Szuverenista",        leftLabel: "🌐 NATO",   rightLabel: "🏠 Szuv" },
  sajto:           { left: "Sajtószabadság",      right: "Állami szabályozás",  leftLabel: "🔵 Lib",    rightLabel: "🔴 Konz" },
  korrupcio:       { left: "Független intézmények",right: "Nemzeti érdek",      leftLabel: "🔵 Lib",    rightLabel: "🔴 Konz" },
  valasztasi_reform:{ left: "Reform szükséges",   right: "Rendszer jól működik",leftLabel: "🔴 Bal",    rightLabel: "🔵 Jobb" },
  energiapolitika: { left: "Megújuló energia",    right: "Atomerőmű / Fosszilis",leftLabel: "🔵 Zöld",  rightLabel: "🔴 Konz" },
  kivandorlas:     { left: "Strukturális reform", right: "Megtartó program",    leftLabel: "🔴 Bal",    rightLabel: "🔵 Jobb" },
  drog_politika:   { left: "Dekriminalizáció",    right: "Szigorú tilalom",     leftLabel: "🔵 Lib",    rightLabel: "🔴 Konz" },
}
```

**Toggle megjelenése** minden témánál:
```
[Bal opció]  [Semleges]  [Jobb opció]
```
3 gomb egymás mellett. Az aktív gomb `--accent` háttér. Alapállapot: Semleges.
Tovább gomb: azonnal aktív (semleges is valid álláspont).

---

### 4. lépés — Ideológiai összefoglaló és indítás

Ez a lépés a pártalapítás alapján **kiszámítja** az ideológiai pozíciót és megmutatja.

**Az ideológiai számítás:**

```javascript
function calculateIdeology(topics, stances) {
  let lrSum = 0, lrCount = 0
  let clSum = 0, clCount = 0

  topics.forEach(topicId => {
    const topic = TOPICS.find(t => t.id === topicId)
    const stance = stances[topicId] // "progressive" | "neutral" | "conservative"
    const stanceValue = { progressive: -1, neutral: 0, conservative: 1 }[stance]

    if (topic.axis === "LR" || topic.axis === "SPECIAL") {
      lrSum += stanceValue * topic.weight
      lrCount += topic.weight
    }
    if (topic.axis === "CL") {
      clSum += stanceValue * topic.weight
      clCount += topic.weight
    }
  })

  return {
    ideologyX: lrCount > 0 ? lrSum / lrCount : 0,  // -1 bal, +1 jobb
    ideologyY: clCount > 0 ? clSum / clCount : 0   // -1 liberális, +1 konzervatív
  }
}
```

**Archetípus meghatározás:**

```javascript
function getArchetype(x, y) {
  // x: bal-jobb (-1..+1), y: liberális-konzervatív (-1..+1)
  if (x < -0.4 && y < -0.3) return { name: "Zöld szociáldemokrata",   desc: "Baloldali, progresszív értékekkel" }
  if (x < -0.4 && y > 0.3)  return { name: "Nemzeti baloldali",        desc: "Szociális programok, konzervatív értékek" }
  if (x < -0.4)              return { name: "Szociáldemokrata",         desc: "Gazdasági baloldal, mérsékelt értékek" }
  if (x > 0.4  && y < -0.3) return { name: "Liberális jobboldal",      desc: "Piaci gazdaság, progresszív társadalom" }
  if (x > 0.4  && y > 0.3)  return { name: "Nemzeti konzervatív",      desc: "Jobboldali gazdaság, hagyományos értékek" }
  if (x > 0.4)               return { name: "Kereszténydemokrata",      desc: "Jobbközép, mérsékelt értékek" }
  if (y < -0.4)              return { name: "Liberális centrista",      desc: "Középen, progresszív társadalomképpel" }
  if (y > 0.4)               return { name: "Konzervatív centrista",    desc: "Középen, hagyományos értékrenddel" }
  return                            { name: "Néppárti pragmatikus",     desc: "Középutas, gyűjtőpárti jelleg" }
}
```

**Kezdő bónuszok kiszámítása:**

```javascript
function calculateStartingBonuses(topics, stances, ideologyX, ideologyY) {
  const bonuses = { urban: 1.0, rural: 1.0, young: 1.0, elderly: 1.0, worker: 1.0, intellectual: 1.0 }

  // Téma alapú bónuszok
  if (topics.includes("lakhatás") && stances.lakhatás === "progressive")  bonuses.young  *= 1.25
  if (topics.includes("lmbtq")    && stances.lmbtq   === "progressive")  bonuses.young  *= 1.20
  if (topics.includes("lmbtq")    && stances.lmbtq   === "progressive")  bonuses.urban  *= 1.15
  if (topics.includes("migráció") && stances.migráció === "conservative") bonuses.rural  *= 1.20
  if (topics.includes("minimalbér") && stances.minimalbér === "progressive") bonuses.worker *= 1.25
  if (topics.includes("egeszsegugy"))                                     bonuses.elderly*= 1.15
  if (topics.includes("korrupcio"))                                       bonuses.intellectual *= 1.20
  if (topics.includes("oktatas"))                                         bonuses.intellectual *= 1.15

  // Ideológia alapú bónuszok
  if (ideologyX < -0.3) { bonuses.worker *= 1.1; bonuses.young *= 1.05 }
  if (ideologyX >  0.3) { bonuses.rural  *= 1.1; bonuses.elderly *= 1.05 }
  if (ideologyY < -0.3) { bonuses.urban  *= 1.1; bonuses.intellectual *= 1.1 }
  if (ideologyY >  0.3) { bonuses.rural  *= 1.1; bonuses.elderly *= 1.1 }

  return bonuses
}
```

**Az összefoglaló képernyő UI elemei:**

1. **2D ideológiai térkép** — 200×200px SVG vagy Canvas
   - 4 negyed: Bal-Liberális (kék), Jobb-Liberális (zöld), Bal-Konzervatív (lila), Jobb-Konzervatív (piros)
   - Tengely feliratok: "Bal" ↔ "Jobb", "Liberális" ↔ "Konzervatív"
   - Ismert pártok pontjai (nem kattinthatók, csak referencia):
     ```javascript
     const REFERENCE_PARTIES = [
       { name: "Fidesz",    x:  0.75, y:  0.80, color: "#ff6600" },
       { name: "Tisza",     x:  0.20, y:  0.10, color: "#005ea5" },
       { name: "Mi Hazánk", x:  0.85, y:  0.90, color: "#333333" },
       { name: "DK",        x: -0.50, y: -0.20, color: "#e41e1e" },
       { name: "MKKP",      x: -0.10, y: -0.60, color: "#ff69b4" },
     ]
     ```
   - A te pontod: `--accent` szín, kicsit nagyobb, animated pulse

2. **Archetípus kártya** — `"Pártod típusa: [archetypeName]"` + leírás

3. **Kezdő bónusz lista** — csak azok ahol a szorzó > 1.0:
   ```
   ✓ Fiatal szavazók konverziója +25%
   ✓ Városi szavazók konverziója +15%
   ```

4. **"Párt alapítása" gomb** — nagy, accent szín, egyszer kattintható

---

## Integrációs pont — hogyan kapcsolódik a meglévő játékhoz

### Képernyőváltás logika

```javascript
// A játék indulásakor:
function init() {
  const saved = localStorage.getItem('partyConfig')
  if (saved) {
    partyConfig = JSON.parse(saved)
    showGame()  // rögtön a meglévő idle
  } else {
    showFoundingScreen()  // az új képernyő
  }
}

function completeFoundingScreen(config) {
  partyConfig = config
  localStorage.setItem('partyConfig', JSON.stringify(config))
  
  // Átadja a bónuszokat a meglévő state-nek
  state.clickPower = 1 * config.startingBonuses.urban  // példa
  
  // Frissíti a header párt-nevét
  document.getElementById('party-name').value = config.name
  
  showGame()
}
```

### Mit módosít a meglévő játékban

**Minimális változtatások a meglévő kódban:**

1. A `<body>` elé kerül egy `<div id="founding-screen">` — a játék fő layout-ja kap egy `id="game-screen"` attributumot
2. Az `init()` függvény hívás elé bekerül a fenti ellenőrzés
3. A `party-name` input kezdeti értéke a `partyConfig.name`-ből jön
4. A header-ben megjelenik az archetípus neve kis badge-ként (opcionális)

**Semmi más nem változik a meglévő játékban.**

---

## Képernyőváltás animáció

```css
#founding-screen {
  position: fixed;
  inset: 0;
  background: var(--bg);
  z-index: 1000;
  transition: opacity 0.4s ease;
}

#founding-screen.hiding {
  opacity: 0;
  pointer-events: none;
}

#game-screen {
  opacity: 0;
  transition: opacity 0.4s ease;
}

#game-screen.visible {
  opacity: 1;
}
```

```javascript
function showGame() {
  document.getElementById('founding-screen').classList.add('hiding')
  setTimeout(() => {
    document.getElementById('founding-screen').style.display = 'none'
    document.getElementById('game-screen').classList.add('visible')
    gameTick() // a meglévő game loop indul
  }, 400)
}
```

---

## Lépésváltás a pártalapítási wizard-on belül

```javascript
let currentStep = 1  // 1, 2, 3, 4

function goToStep(n) {
  document.querySelectorAll('.founding-step').forEach(el => el.style.display = 'none')
  document.getElementById(`step-${n}`).style.display = 'block'
  currentStep = n
  updateProgressDots()
}

// Minden step egy <div id="step-N" class="founding-step"> — nincs router
```

**Progress indicator:** 4 kis pont a tetején (● ● ○ ○ stílusban), nem kattintható.

---

## Amit NEM kell megcsinálni ebben a fázisban

- ❌ Semmilyen módosítás a meglévő idle mechanikában (épületek, termelés, stb.)
- ❌ Térkép
- ❌ Karakterek / dispatch rendszer
- ❌ NPC pártok
- ❌ Médiaelérés valuta
- ❌ Feladatok / missziók
- ❌ Mentés/betöltés bármire vonatkozóan KIVÉVE a partyConfig localStorage mentése

---

## Elvárt viselkedés összefoglalva

1. **Első indításkor:** Pártalapítási wizard jelenik meg (4 lépés)
2. **Wizard végén:** Animált átmenet a meglévő játékba, párt neve bekerül a headerbe
3. **Visszatéréskor (oldal újratöltés):** Wizard nem jelenik meg, rögtön a játék indul
4. **"Új játék" gomb** (opcionális, a meglévő játékban valahol): `localStorage.removeItem('partyConfig')` + oldal újratöltés

---

## Stíluskövetelmény

Az új képernyő **pontosan kövesse a meglévő játék stílusát:**
- Azonos CSS változók
- Azonos font (IBM Plex Mono fejlécekhez, IBM Plex Sans szövegekhez)
- Azonos sötét háttér, border stílusok
- Gombok: `background: var(--surface2)`, hover: `background: var(--border)`, active/selected: `background: rgba(240,192,64,0.15)`, border: `1px solid var(--accent)`
- Nincs rounded corner nagyobb mint 8px
- Nincs box-shadow, nincs gradient — flat design

---

## Tesztelési ellenőrzőlista

- [ ] Első betöltés: wizard jelenik meg
- [ ] Visszatöltés: wizard nem jelenik meg, játék indul
- [ ] Párt neve: validáció (min 3 karakter), tovább gomb inaktív ha üres
- [ ] Rövidítés: auto-generálódik, szerkeszthető
- [ ] Szín: 8 opció, kiválasztott jelölve
- [ ] Témák: pontosan 5 kiválasztható, a 6. nem választható (a már meglevők törölhetők)
- [ ] Álláspontok: mind 5 témának van álláspontja (semleges alapértelmezett)
- [ ] Ideológiai térkép: pont a helyes pozícióban jelenik meg
- [ ] Referencia pártok a térképen láthatók
- [ ] Archetípus helyes a pozíció alapján
- [ ] Kezdő bónuszok megjelennek (csak amik > 1.0)
- [ ] Alapítás gomb → animált átmenet → meglévő játék fut
- [ ] A meglévő játék összes funkciója változatlanul működik (épületek, kattintás, fejlesztések)
- [ ] LocalStorage: partyConfig mentve
- [ ] LocalStorage törlés után: wizard újra megjelenik
