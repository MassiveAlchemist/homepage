# Plan: Gräsroots webbplats – allt bakom frontend

En portfolio-sajt för produktionsbyrån **Gräsroot** (arbetsnamn) som samlar projekt från alla medlemmar – med bilder, texter, typer och taggar. Krav:

1. **Lätt att uppdatera** – alla medlemmar kan lägga upp ett projekt på några minuter, utan att kunna koda, och det publiceras direkt.
2. **Lätt att söka** – besökare hittar projekt efter typ, tagg, person, år eller fritext.
3. **Svenska och engelska.**
4. **Namnet kan ändras** – inget får vara hårdkodat till "Gräsroot" eller till en domän.

### Ansvarsfördelning

| Del | Vem/vad |
|---|---|
| **Frontend: stil, färger, typsnitt, layout** | Designas i **Canva** (Alice). Implementeras som komponenter + CSS ovanpå backend. |
| **Allt bakom frontend** (den här planen) | Innehållsmodell, CMS/publicering, datalager, routing, tvåspråkighet, sök, bilder, SEO, bygg och hosting. |

Backend levereras med en **ostylad skelett-frontend** (ren, semantisk HTML) så att allt går att testa från dag ett, och så att Canva-designen sedan kan "kläs på" utan att något bakom behöver ändras. Se [§ 8 Gränssnittet mot frontend](#8-gränssnittet-mot-frontend).

---

## 1. Teknik

| Del | Val | Varför |
|---|---|---|
| Sajtbyggare | **Astro** (statisk sajt) | Snabb, inbyggt stöd för innehållssamlingar med schemavalidering, i18n-routing och bildoptimering. Frontend kan skrivas i vanlig HTML/CSS. |
| Innehåll | **Filer i Git-repot** (en mapp per projekt) | Ingen databas, ingen server. Full historik → allt går att ångra. |
| Redigering | **Pages CMS** (pagescms.org) | Webbformulär, inloggning med GitHub, bilduppladdning. Sparar direkt till repot. Gratis. |
| Sök | **Pagefind** | Statiskt sökindex som byggs vid publicering, körs i webbläsaren, stöder filter och flera språk. Används "headless" (via JS-API) så att sökgränssnittet kan designas fritt. |
| Hosting | **Cloudflare Pages** | Bygger automatiskt vid varje ändring. Gratis, HTTPS, förhandsversioner, behåller senaste fungerande version om ett bygge misslyckas. |
| Video | **Vimeo / YouTube-inbäddning** | Stora videofiler hör inte hemma i Git. |

---

## 2. Publiceringsflöde – alla publicerar direkt

```
Medlem i Pages CMS  →  "Spara"  →  commit till main  →  Cloudflare bygger (≈1 min)  →  live
                                                         │
                                                         └─ bygget validerar innehållet först
```

- **Ingen granskning:** alla medlemmar bjuds in som medarbetare på GitHub-repot och sparar direkt till `main`.
- **Skyddsnät i stället för granskning:**
  - **Validering vid bygget:** saknad titel, okänd typ, okänd person, trasig bildlänk m.m. stoppar bygget med ett tydligt felmeddelande – och **den tidigare versionen ligger kvar live**. Ingen kan "ta sönder" sajten.
  - **Historik:** varje ändring är en commit med namn på den som gjorde den. Allt kan återställas.
  - **Notis vid fel:** misslyckat bygge → e-post/Discord-notis till gruppen (Cloudflare-notis eller GitHub Action).
- **Dolt-läge (valfritt):** en kryssruta `dold` för den som vill spara ett halvfärdigt projekt utan att det syns. Ingen godkännandeprocess.
- **Bildstorlek:** CMS:et begränsar uppladdningar (t.ex. max 10 MB), och en GitHub Action skalar automatiskt ner originalbilder över ~3000 px så att repot inte växer i onödan. Astro skapar sedan rätt storlekar och format (AVIF/WebP) för webben.

---

## 3. Innehållsmodell

Alla texter som besökare läser finns på **sv** och **en**. Allt annat (datum, bilder, typer, personer) delas mellan språken och fylls bara i en gång.

### Projekt – `src/content/projekt/<slug>/index.md`

```yaml
---
sv:
  titel: "Lysande väv"
  ingress: "Ett vävt tyg med inbyggd fiberoptik som reagerar på rörelse."
  text: |
    Brödtext på svenska (Markdown) ...
en:                                   # valfritt – saknas den visas svenskan med en notis
  titel: "Luminous Weave"
  ingress: "A woven fabric with embedded fiber optics that responds to movement."
  text: |
    Body text in English ...
datum: 2026-06-21
typ: [textil, fiberoptik, elektronik] # en eller flera, från typlistan
medverkande:
  - { person: alice, roll: { sv: Idé och väv, en: Concept and weaving } }
  - { person: kalle, roll: { sv: Elektronik, en: Electronics } }
taggar: [arduino, ull, utställning]
kund: "Hälsinge Kulturförening"       # valfritt
plats: "Hudiksvall"                   # valfritt
omslag: { bild: ./omslag.jpg, alt: { sv: "...", en: "..." } }
galleri:
  - { bild: ./detalj1.jpg, alt: { sv: "...", en: "..." } }
video: https://vimeo.com/123456       # valfritt
lankar: [{ etikett: "Utställningen", url: "https://..." }]   # valfritt
utvald: false                         # visas på startsidan
dold: false
---
```

- **Slug** (mappnamnet) är gemensam för båda språken och genereras från titeln. Den ändras inte om titeln ändras → länkar går aldrig sönder.
- Bilder ligger i samma mapp som texten.
- **Alt-text krävs** (minst svenska) – bra för tillgänglighet och för sök.

### Typer – `src/content/typer.yml`

Typ = *vad projektet är gjort av/med*. Styrd, utbyggbar lista; ett projekt kan ha flera.

```yaml
- id: textil
  sv: { namn: Textil, beskrivning: "Plagg, vävar, broderi, mjuka skulpturer" }
  en: { namn: Textile, beskrivning: "Garments, weaving, embroidery, soft sculpture" }
- id: elektronik
  sv: { namn: Elektronik, beskrivning: "Kretsar, sensorer, mikrokontroller, ljud" }
  en: { namn: Electronics, beskrivning: "Circuits, sensors, microcontrollers, sound" }
- id: fiberoptik
  sv: { namn: Fiberoptik, beskrivning: "Ljusledare, belysning, ljusinstallationer" }
  en: { namn: Fiber optics, beskrivning: "Light guides, lighting, light installations" }
- id: 3d-print
  sv: { namn: 3D-print, beskrivning: "Printade objekt, prototyper, formar" }
  en: { namn: 3D printing, beskrivning: "Printed objects, prototypes, moulds" }
- id: odling
  sv: { namn: Odling, beskrivning: "Växter, mycel, levande material" }
  en: { namn: Growing, beskrivning: "Plants, mycelium, living materials" }
```

Ny typ = ny post (går att göra i CMS:et). Bygget kontrollerar att alla projekt bara använder typer som finns.

### Taggar – fria, med valfri översättning

- Taggar skrivs fritt (på svenska) i projektet. CMS:et föreslår befintliga taggar för att undvika dubbletter.
- Normaliseras vid bygget (gemener, trimmade blanksteg).
- `src/content/taggar.yml` är en **valfri** översättningstabell (`ull: wool`). Saknas en översättning visas taggen som den är på engelska sidan. Bygget listar oöversatta taggar som en påminnelse (inte ett fel).

### Personer – `src/content/personer/<id>.md`

`namn`, `portratt` (+alt), `bio: { sv, en }`, `kompetenser`, `lankar` (Instagram, webb, e-post), `aktiv` (för tidigare medlemmar).

### Sajtinställningar – `src/content/sajt.yml` (redigerbar i CMS)

```yaml
namn: "Gräsroot"                 # byt här → byts överallt
domän: "https://grasroot.pages.dev"
sv: { slogan: "...", om: "...", kontakt: "..." }
en: { slogan: "...", about: "...", contact: "..." }
epost: "hej@..."
sociala: { instagram: "...", ... }
```

**Namnet och domänen finns bara här.** Inga filnamn, URL:er, mappnamn eller kod innehåller namnet. Byte av namn = en rad; byte av domän = en rad + koppla domänen i Cloudflare.

### Gränssnittstexter – `src/i18n/sv.json`, `src/i18n/en.json`

Knappar, rubriker och meddelanden ("Sök", "Visa alla", "Inga träffar", "Endast på svenska") – frontend hämtar alltid text härifrån, aldrig hårdkodat.

---

## 4. Tvåspråkighet (i18n)

| | Svenska (standard) | Engelska |
|---|---|---|
| Start | `/` | `/en/` |
| Projektlista | `/projekt` | `/en/projects` |
| Projekt | `/projekt/<slug>` | `/en/projects/<slug>` |
| Typ | `/typ/<id>` | `/en/type/<id>` |
| Tagg | `/tagg/<tagg>` | `/en/tag/<tagg>` |
| Personer | `/personer/<id>` | `/en/people/<id>` |
| Om | `/om` | `/en/about` |

- **Språkväxlaren** leder alltid till *samma* sida på andra språket.
- **Fallback:** saknas engelsk text visas svenskan med en tydlig notis, och sidan får `lang="sv"` på innehållet. Sidan finns alltså alltid på båda språken.
- **SEO:** `hreflang`-länkar mellan språkversionerna, `<html lang>` rätt per sida, separat sitemap per språk.
- **Förstagångsbesök:** ingen automatisk omdirigering (stör sökmotorer). I stället en diskret "In English?"-länk om webbläsaren är engelskspråkig.
- **Översättningsstatus:** en byggrapport listar projekt som saknar engelsk text.
- *Valfritt senare:* en GitHub Action som tar fram ett engelskt utkast med AI när engelska saknas, märkt "maskinöversatt" tills någon granskat det.

---

## 5. Sök

- **Pagefind** indexerar den byggda sajten. Ett separat index per språk (styrs av `lang`), med rätt ordstamning för svenska och engelska ("vävar" hittar "väv", "weaving" hittar "weave").
- **Fritext** över titel, ingress, text, kund, plats, typer, taggar, medverkande och alt-texter.
- **Filter:** typ (flerval), tagg, medverkande, år. Går att kombinera (*textil + elektronik*, *Alice + 2025*).
- **Sortering:** relevans eller datum.
- **Headless:** sök används via en liten modul `src/lib/sok.ts` (`sok(fråga, filter) → träffar`) så att Canva-designen helt bestämmer hur sökrutan och resultaten ser ut.
- **Delbara sökningar:** fråga och filter sparas i URL:en (`/projekt?typ=textil&typ=elektronik&q=ljus`).
- **Utan JavaScript:** typ-, tagg- och personsidor är vanliga länkade sidor och fungerar alltid.
- Indexet byggs automatiskt vid varje publicering – inget att underhålla.

---

## 6. Övrigt bakom kulisserna

- **Bilder:** Astro `<Image>`/`<Picture>` → responsiva storlekar, AVIF/WebP, lazy loading, fasta dimensioner (ingen layoutförskjutning). Fokuspunkt per bild (valfritt fält) för snygga beskärningar.
- **SEO:** titlar och beskrivningar per sida och språk, Open Graph-bilder från omslaget, strukturerad data (`CreativeWork`, `Person`, `Organization`), sitemap, `robots.txt`.
- **RSS-flöde** per språk med nya projekt (gratis att lägga till).
- **Statistik:** Cloudflare Web Analytics (cookiefri → ingen cookiebanner).
- **Kontakt:** e-postlänk från start; ev. formulär via Cloudflare Pages Function eller Formspree senare.
- **Domän:** tills namnet är bestämt körs sajten på `*.pages.dev`. När domän finns: lägg in den i `sajt.yml` och koppla den i Cloudflare (≈10 min).
- **Kvalitetskontroller i CI** (GitHub Actions vid varje ändring): innehållsvalidering, typkontroll, trasiga interna länkar, oöversatta texter (varning), bildstorlek.

---

## 7. Mappstruktur

```
/
├── .pages.yml                    # CMS: formulär för projekt, personer, typer, taggar, sajt
├── .github/workflows/            # validering, bildnedskalning, byggnotiser
├── astro.config.mjs              # i18n, integrationer
├── src/
│   ├── content/
│   │   ├── projekt/<slug>/index.md + bilder
│   │   ├── personer/<id>.md + porträtt
│   │   ├── typer.yml
│   │   ├── taggar.yml
│   │   └── sajt.yml
│   ├── content.config.ts         # scheman + validering (Zod)
│   ├── i18n/                     # sv.json, en.json, hjälpfunktioner för språk och URL:er
│   ├── lib/
│   │   ├── data.ts               # ← datalagret frontend använder
│   │   └── sok.ts                # ← sök-API för frontend
│   ├── pages/                    # routing (tunna filer som hämtar data och skickar till layouts)
│   ├── layouts/                  # ← FRONTEND (Canva)
│   ├── components/               # ← FRONTEND (Canva)
│   └── styles/tokens.css         # ← FRONTEND: färger, typsnitt, avstånd från Canva
└── docs/
    ├── lagga-upp-projekt.md      # guide för medlemmar (sv)
    └── frontend.md               # datakontraktet för den som bygger frontend
```

---

## 8. Gränssnittet mot frontend

Det som gör att backend och Canva-designen kan tas fram parallellt.

**Datalagret** (`src/lib/data.ts`) ger färdiga, språkanpassade objekt – frontend behöver aldrig läsa filer eller hantera fallback själv:

```ts
hamtaProjekt(lang, { typ?, tagg?, person?, utvalda?, gräns? }): ProjektKort[]
hamtaProjektSida(lang, slug): ProjektSida        // inkl. liknande projekt, prev/next
hamtaTyper(lang): Typ[]                           // med antal projekt per typ
hamtaTaggar(lang): Tagg[]                         // med antal
hamtaPersoner(lang): Person[]
hamtaSajt(lang): Sajt                             // namn, slogan, navigation, sociala länkar
t(lang, nyckel): string                           // gränssnittstexter
url(lang, sida, param?): string                   // rätt URL på rätt språk
```

**Exempel på datatyp:**

```ts
type ProjektKort = {
  slug: string; url: string; titel: string; ingress: string;
  datum: Date; år: number;
  omslag: BildData;              // färdig för <Picture>
  typer: { id: string; namn: string; url: string }[];
  taggar: { namn: string; url: string }[];
  medverkande: { namn: string; url: string; roll?: string }[];
  fallbackSprak: boolean;        // true = texten visas på svenska
};
```

**Komponentlista** som Canva-designen behöver täcka (skelettet har en ostylad version av varje):
Sidhuvud (logga, nav, språkväxlare, sök) · Sidfot · Projektkort · Projektrutnät · Filterpanel (typ/tagg/person/år) · Sökruta + resultatlista · Typetikett · Taggetikett · Projektsida (omslag, text, galleri/lightbox, video, medverkande, liknande) · Personkort + personsida · Typsida · Notis "endast på svenska" · Tomt läge ("inga träffar") · 404.

**Från Canva till kod:**
1. Designa minst: startsida, projektlista med filter, projektsida, personsida – i mobil- och desktopbredd.
2. Definiera färger, typsnitt och avstånd i Canvas varumärkeskit → översätts till `tokens.css` (CSS-variabler, inkl. ev. mörkt läge).
3. Exportera eller dela designen; den implementeras komponent för komponent ovanpå skelettet. (Canva-designen kan läsas direkt härifrån via Canva-kopplingen om du delar länken.)

---

## 9. Faser (backend)

### Fas 1 – Grund och innehåll
- [ ] Rensa repot, sätt upp Astro + TypeScript.
- [ ] Scheman med validering: projekt, personer, typer, taggar, sajt.
- [ ] i18n: routing sv/en, språkväxlare, fallback, gränssnittstexter.
- [ ] Datalagret `data.ts` + `docs/frontend.md`.
- [ ] Skelett-frontend (ostylad) för alla sidor.
- [ ] 3–5 riktiga testprojekt med bilder, på båda språken.
- [ ] Cloudflare Pages: automatisk publicering på `*.pages.dev`.

### Fas 2 – Sök och navigering
- [ ] Pagefind med två språkindex och filter.
- [ ] `sok.ts` (headless) + filter i URL.
- [ ] Typ-, tagg- och personsidor, liknande projekt.

### Fas 3 – Publicering för alla
- [ ] Pages CMS-formulär (`.pages.yml`) med sv/en-fält sida vid sida.
- [ ] GitHub Actions: validering, bildnedskalning, notis vid misslyckat bygge.
- [ ] Bjud in alla medlemmar.
- [ ] Guide: "Så lägger du upp ett projekt".

### Fas 4 – Canva-design in
- [ ] `tokens.css` från Canva-varumärkeskitet.
- [ ] Implementera komponenterna enligt designen.
- [ ] Tillgänglighetsgenomgång (kontrast, tangentbord, skärmläsare).

### Fas 5 – Lansering
- [ ] SEO, Open Graph, sitemap, RSS, statistik.
- [ ] Namn och domän bestämda → uppdatera `sajt.yml` och koppla domänen.

---

## 10. Kostnad

| Post | Kostnad |
|---|---|
| Hosting, CMS, sök, GitHub, statistik | 0 kr |
| Domän (när namnet är klart) | ca 100–200 kr/år |
| Video (Vimeo/YouTube) | 0 kr på gratisnivå |

---

## 11. Beslut och öppna frågor

**Beslutat**
- ✅ Typ är en sökbar, utbyggbar lista. Start: textil, elektronik, fiberoptik, 3d-print, odling.
- ✅ Alla medlemmar publicerar direkt (skyddsnät: validering + historik).
- ✅ Svenska (standard) och engelska.
- ✅ Namn och domän är inte bestämda → finns bara i `sajt.yml`, tillfällig adress på `*.pages.dev`.
- ✅ Frontend designas i Canva; den här planen täcker allt bakom.

**Öppet**
1. **Hur många medlemmar** – och har alla GitHub-konton? (Behövs för att logga in i CMS:et; det är gratis.)
2. **Var ligger videorna idag?** (Vimeo, YouTube, Drive …)
3. **Kontakt** – räcker e-post, eller formulär?
4. **Notiser vid fel** – e-post, Discord eller något annat?
5. **Canva** – när ungefär finns en första design? Fas 1–3 kan byggas klart innan dess.
