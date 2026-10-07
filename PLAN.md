# Plan: Gräsroots webbplats

En portfolio-sajt för produktionsbyrån **Gräsroot** som samlar projekt från alla medlemmar – med bilder, texter och taggar. Två huvudkrav:

1. **Lätt att uppdatera** – en medlem ska kunna lägga upp ett nytt projekt på några minuter, utan att kunna koda.
2. **Lätt att söka** – en besökare ska snabbt hitta projekt efter ämne, person, år eller fritext.

---

## 1. Rekommenderad teknik (kort)

| Del | Val | Varför |
|---|---|---|
| Sajtbyggare | **Astro** (statisk sajt) | Snabb, enkel, inbyggt stöd för innehåll i Markdown och automatisk bildoptimering. |
| Innehåll | **Markdown-filer i repot** (en mapp per projekt) | Inget databas- eller serverkrångel. Allt versionshanteras i Git, gratis backup. |
| Redigering | **Pages CMS** (pagescms.org) | Webbgränssnitt där man loggar in med GitHub, fyller i ett formulär och laddar upp bilder. Kräver ingen egen server. |
| Sök | **Pagefind** | Fritextsök som byggs vid publicering och körs helt i webbläsaren. Stöder filter (taggar, person, år). Gratis, ingen extern tjänst. |
| Hosting | **Cloudflare Pages** eller **Netlify** (gratisnivå) | Bygger och publicerar automatiskt vid varje ändring i GitHub. Egen domän (t.ex. grasroot.se) + HTTPS ingår. |
| Video | **Vimeo / YouTube-inbäddning** | Stora videofiler hör inte hemma i Git. |

**Flödet för att lägga upp nytt material:**
Logga in i Pages CMS → "Nytt projekt" → fyll i titel, text, taggar, välj medverkande, dra in bilder → Spara.
→ GitHub får en ny commit → hostingen bygger om sajten → live efter ca 1 minut.

*Alternativ:* Vill ni hellre slippa Git helt kan Sanity eller Notion användas som innehållskälla, men det ger fler rörliga delar och beroenden. Börja med lösningen ovan.

---

## 2. Innehållsmodell

### Projekt (`src/content/projekt/<slug>/index.md`)

```yaml
---
titel: "Sommarnatt i Hälsingland"
datum: 2026-06-21
typ: [textil, elektronik, fiberoptik]   # en eller flera, se "Typer" nedan
medverkande: [alice, kalle]   # kopplas till personsidor
roller:                   # valfritt, vem gjorde vad
  - { person: alice, roll: Regi }
  - { person: kalle, roll: Foto }
kund: "Hälsinge Kulturförening"   # valfritt
taggar: [natur, drama, 16mm]
omslag: ./omslag.jpg
galleri: [./bild1.jpg, ./bild2.jpg]
video: https://vimeo.com/123456   # valfritt
utvald: true             # visas på startsidan
utkast: false            # true = syns inte på sajten än
---
Brödtext om projektet: idé, process, resultat ...
```

Bilderna ligger i samma mapp som texten → lätt att hålla ordning, och Astro optimerar dem automatiskt (rätt storlek, WebP/AVIF, lazy loading).

### Typer (`src/content/typer.yml`)
Gräsroot arbetar med alla sorters medier och material, så **typ** är en egen, sökbar och filtrerbar dimension – skild från de fria taggarna. Ett projekt kan ha **flera typer** (t.ex. ett plagg med inbyggd fiberoptik = `textil` + `fiberoptik` + `elektronik`).

Startlista:

| Typ | Exempel |
|---|---|
| `textil` | Plagg, vävar, broderi, mjuka skulpturer |
| `elektronik` | Kretsar, sensorer, mikrokontroller, ljud |
| `fiberoptik` | Ljusledare, belysning, ljusinstallationer |
| `3d-print` | Printade objekt, prototyper, formar |
| `odling` | Växter, mycel, levande material |

- Listan ligger i **en** fil (`typer.yml`) med namn, kort beskrivning och ev. ikon/färg per typ. Ny typ = en ny rad (går att göra i CMS:et), så listan kan växa fritt.
- I CMS:et väljs typer från listan (flerval) – inga stavfel eller dubbletter.
- Bygget validerar att varje projekts typer finns i listan.
- Varje typ får en egen sida, t.ex. `/typ/fiberoptik`, med beskrivning och alla projekt.
- Typerna visas som tydliga "etiketter" på projektkorten och är sökbara i fritextsöket (söker man "odling" hittar man både projekt av typen odling och projekt som nämner odling i texten).

**Typ vs. taggar:** typ = *vad det är gjort av/med* (styrd lista). Taggar = allt annat, fritt (tema, plats, teknik i detalj, t.ex. `arduino`, `ull`, `utställning`).

### Personer (`src/content/personer/<namn>.md`)
Namn, porträtt, kort bio, roller/kompetenser, länkar (Instagram, egen sajt, e-post).

### Taggar
Fri text, men CMS:et visar befintliga taggar som förslag för att undvika dubbletter som "Musikvideo"/"musikvideo"/"MV". Taggar normaliseras till gemener vid bygget.

---

## 3. Sidor

| Sida | Innehåll |
|---|---|
| **Start** `/` | Kort intro om Gräsroot, utvalda projekt, senaste projekt, sökruta. |
| **Projekt** `/projekt` | Rutnät med alla projekt. Filter: typ (flerval), tagg, person, år. Sortering: nyast först. |
| **Projektsida** `/projekt/<slug>` | Omslag, text, galleri (klicka för helskärm), video, medverkande med roller, taggar (klickbara), "fler liknande projekt". |
| **Typ** `/typ/<typ>` | Beskrivning av typen + alla projekt av den typen. Översikt på `/typ`. |
| **Tagg** `/taggar/<tagg>` | Alla projekt med en viss tagg. |
| **Personer** `/personer` + `/personer/<namn>` | Medlemmarna och var och ens projekt. |
| **Om / Kontakt** `/om` | Vilka ni är, vad ni gör, kontaktuppgifter/formulär. |
| **Sök** (överallt) | Sökruta i sidhuvudet, öppnas med `/` eller ⌘K. Resultat visas direkt medan man skriver. |

---

## 4. Sökfunktionen i detalj

- **Fritext** över titel, brödtext, kund, typer, taggar och namn på medverkande.
- **Filter** (Pagefind filters): typ, tagg, medverkande, år – går att kombinera (t.ex. *textil + elektronik*).
- **Typ först:** typerna visas som klickbara knappar direkt under sökrutan, så man kan bläddra utan att veta vad man letar efter.
- **Snabbt:** indexet delas upp i små bitar, så bara det som behövs laddas ner.
- **Svenska:** sätt `lang="sv"` så att ord stamhanteras rätt ("filmer" hittar "film"). Å, ä, ö fungerar.
- **Delbara sökningar:** filterval sparas i URL:en (`/projekt?tagg=natur&person=alice`).
- **Fallback:** taggsidor och personsidor fungerar även utan JavaScript.

---

## 5. Design (riktlinjer)

- Bilderna i fokus: stora omslag, lugnt typografiskt ramverk, mycket luft.
- Mobil först – de flesta besökare kommer via Instagram-länkar på mobilen.
- Ljust/mörkt läge.
- Tillgänglighet: alt-texter krävs i CMS:et, tydliga fokusmarkeringar, god kontrast.
- En egen logga/ordbild för Gräsroot + en accentfärg (t.ex. gräsgrönt) är allt som behövs från start.

---

## 6. Faser

### Fas 1 – Grund (≈1 helg)
- [ ] Rensa repot och sätt upp Astro-projektet.
- [ ] Definiera innehållsscheman (projekt, personer, typer) med validering.
- [ ] Grundlayout: sidhuvud, sidfot, startsida, projektlista, projektsida.
- [ ] Lägg in 3–5 riktiga projekt som testmaterial.
- [ ] Koppla till Cloudflare Pages/Netlify → automatisk publicering.

### Fas 2 – Sök och navigering (≈1 kväll–1 dag)
- [ ] Pagefind-integration med filter.
- [ ] Typsidor, taggsidor, personsidor, "liknande projekt" (delad typ/tagg).
- [ ] Filter i projektlistan, sparade i URL.

### Fas 3 – Enkel redigering (≈1 kväll)
- [ ] Konfigurera Pages CMS (`.pages.yml`) med formulär för projekt och personer.
- [ ] Bjud in alla kompisar som medarbetare på GitHub-repot.
- [ ] Skriv en kort guide: "Så lägger du upp ett projekt" (med skärmdumpar).

### Fas 4 – Finputs (löpande)
- [ ] Design: logga, typsnitt, färger, animationer.
- [ ] SEO: metabeskrivningar, delningsbilder (Open Graph) per projekt, sitemap.
- [ ] Integritetsvänlig statistik (t.ex. Plausible eller Cloudflare Web Analytics).
- [ ] Kontaktformulär.
- [ ] Egen domän.

---

## 7. Mappstruktur (förslag)

```
homepage/
├── .pages.yml                 # CMS-konfiguration
├── astro.config.mjs
├── public/                    # favicon, logga
└── src/
    ├── content/
    │   ├── projekt/
    │   │   └── sommarnatt/
    │   │       ├── index.md
    │   │       ├── omslag.jpg
    │   │       └── bild1.jpg
    │   ├── personer/
    │   │   └── alice.md
    │   └── typer.yml          # textil, elektronik, fiberoptik, 3d-print, odling
    ├── content.config.ts      # scheman och validering
    ├── components/            # ProjektKort, Galleri, Sök, Filter ...
    ├── layouts/
    └── pages/
        ├── index.astro
        ├── om.astro
        ├── projekt/
        ├── typ/
        ├── taggar/
        └── personer/
```

---

## 8. Kostnad

| Post | Kostnad |
|---|---|
| Hosting, CMS, sök, GitHub | 0 kr |
| Domän (.se) | ca 100–200 kr/år |
| Video (Vimeo/YouTube) | 0 kr på gratisnivå |

---

## 9. Frågor att besluta innan vi bygger

1. ~~Vilka typer av projekt gör ni?~~ **Beslutat:** typ är en sökbar, utbyggbar lista. Start: textil, elektronik, fiberoptik, 3d-print, odling.
2. **Hur många medlemmar** och ska alla kunna publicera direkt, eller ska någon granska först? (Pages CMS kan spara som utkast.)
3. **Domän** – finns `grasroot.se` eller liknande?
4. **Språk** – bara svenska, eller även engelska?
5. **Finns grafisk profil** (logga, färger, typsnitt) eller ska den tas fram?
6. **Var ligger videorna idag?** (Vimeo, YouTube, Drive …)
7. **Kontakt** – räcker en e-postadress, eller vill ni ha ett formulär?
