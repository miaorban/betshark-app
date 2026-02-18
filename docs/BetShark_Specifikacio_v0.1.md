# BetShark — Alkalmazás Specifikáció v0.1

> **Státusz:** Piszkozat — Ügyfél visszajelzésére vár
> **Dátum:** 2026-02-18

---

## 1. Termékleírás

A **BetShark** egy sportfogadás-elemző mobilalkalmazás, amely egy AI-vezérelt pontozórendszer segítségével értékeli a fogadási piacok kimeneteleit, azonosítja a value fogadásokat, és közérthető elemzést nyújt a felhasználónak. A cél, hogy a felhasználó előre kiszámított pontszámok, odds-összehasonlítások és AI által generált kommentárok alapján megalapozottabb döntéseket hozzon — anélkül, hogy ismernie kellene a háttérben futó adatforrásokat.

---

## 2. Platformok és terjesztés

| Platform | Csatorna |
|---|---|
| iOS | Apple App Store |
| Android | Google Play Áruház |
| Web | Csak landing page (nincs webalkalmazás) |

---

## 3. Támogatott nyelvek

- **Magyar** (elsődleges)
- **Angol**

A nyelv az alkalmazás beállításaiban váltható. Alapértelmezetten az eszköz rendszernyelvét veszi fel.

---

## 4. Dizájnrendszer

| Elem | Érték |
|---|---|
| Háttér | Fekete |
| Elsődleges szín | Narancssárga / sárga-narancs |
| Ikonográfia | Cápa motívum (márkajelzés és value-indikátor) |
| UI minta | Csempés (kártya/tile) navigáció |

A csempék narancssárga/sárga alapon jelennek meg. Ha technikailag megoldható, halvány cápa ikon látható vízjelként a csempék hátterén.

---

## 5. Üzleti modell és hozzáférési szintek

### 5.1 Hozzáférési szintek

| Szint | Hozzáférés | Ár |
|---|---|---|
| Vendég (ingyenes) | Főképernyő (eseményböngészés) | Ingyenes — regisztráció nélkül |
| Prémium | Toplista | 2 990 Ft / hó |

- **Prototípus fázis (1. hónap):** A főképernyő teljesen ingyenes. A Toplista előfizetéshez kötött.
- Ha a prototípus beválik, az árazáson és a hozzáférési modellen módosítunk.

### 5.2 Ingyenes próbaidőszak

- A prémium szint **7 napos ingyenes próbát** kínál új előfizetőknek.
- A próbaidő felhasználónként egyszer vehető igénybe; az újbóli aktiválást technikailag meg kell akadályozni (ld. 9.2. fejezet).

### 5.3 Fizetési megoldás

- Apple In-App Purchase (iOS)
- Google Play Billing (Android)

> **[KÉRDÉS 1]** Szükséges-e harmadik feles fizetési megoldás (pl. Stripe) is, például webes előfizetéshez vagy promóciós kódokhoz? Vagy elegendő az App Store / Play Store IAP?

---

## 6. Pontozórendszer

*(A pontozórendszer algoritmusa már elkészült. Ez a fejezet dokumentációs célból rögzíti a szabályokat.)*

### 6.1 Bemeneti változók

| Változó | Leírás |
|---|---|
| `likelihood_nopred` | Likelihood pontszám, predikciós adat nélkül számítva |
| `value_nopred` | Value pontszám, predikciós adat nélkül számítva |
| `likelihood_pred` | Likelihood pontszám, predikciós adattal számítva |
| `value_pred` | Value pontszám, predikciós adattal számítva |

### 6.2 Overall pontszám kiszámítása

**1. lépés — Közbülső pontszámok:**
```
score_nopred = (likelihood_nopred × 9 + value_nopred × 1) / 10
score_pred   = (likelihood_pred   × 9 + value_pred   × 1) / 10
```

**2. lépés — Overall pontszám:**
```
overall = (score_nopred × 7 + score_pred × 3) / 10
```

**3. lépés — Alacsony odds büntetés:**
```
ha overall >= 7 ÉS odds < 1,5:
    overall -= 2
```

Az overall pontszám értékkészlete: 0–10. Megjelenítés: egy tizedesjegy.

### 6.3 Value besorolás

A kimenet likelihood pontszáma és a legjobb elérhető odds alapján:

| Besorolás | Feltétel |
|---|---|
| **Value** | `(100 / likelihood) < odds ≤ (100 / likelihood) + 0,2` |
| **Strong Value** | `odds > (100 / likelihood) + 0,2` |
| *(nincs jelölés)* | `odds ≤ (100 / likelihood)` |

---

## 7. Adatlekérés és backend pipeline

### 7.1 Meglévő algoritmus — jelenlegi állapot

A pontozó és adatlekérő algoritmus **Python nyelven készült**, és jelenleg a következőt csinálja:

- Véletlenszerűen kér le eseményeket (nincs prioritizálás vagy ütemezés)
- Lefuttatja a pontozó promptot a lekért eseményekre
- A kimenetetet **CSV fájlba menti** (nincs adatbázis)

### 7.2 Szükséges fejlesztési feladatok

Az algoritmus önmagában nem kiszolgálható a mobilalkalmazás számára. A következő fejlesztések szükségesek:

1. **Adatbázisba mentés:** Az algoritmus kimenetét (esemény, piac, kimenetel, pontszámok, AI elemzés) adatbázisba kell persistálni, amelyből a mobilapp REST API-n keresztül kiszolgálható.
2. **Esemény-ütemező (scheduler):** A véletlenszerű lekérés helyett a rendszernek folyamatosan figyelnie kell a közelgő meccseket, és minden meccset annak kezdési ideje előtt 1 órával kell feldolgozni.
3. **Liga-prioritizálás:** A véletlenszerű eseményválasztás helyett a napi 400 kimenet keretén belül a nagyobb ligák kapjanak előnyt (ld. 7.5. fejezet).
4. **REST API:** A backend egy REST API-t kell, hogy szolgáltasson a Flutter mobilalkalmazás számára (eseménylisták, piacok, kimenetel részletek, toplista).

### 7.3 Helyezkedés az architektúrában

Ez a komponens a **Python adatgyűjtő és DB-frissítő** (ld. 13.3. fejezet). Önállóan fut, nem fogad kéréseket a mobilalkalmazástól — csak ír az adatbázisba, amelyből a Node.js backend kiszolgálja az appot.

### 7.4 Napi lekérdezési mennyiség

- Naponta legfeljebb **400 kimenetel** kerül lekérdezésre és pontozásra.

### 7.5 Lekérdezés időzítése

- Az adatokat minden meccs **tervezett kezdési ideje előtt 1 órával** kell lekérdezni.
- Az ütemező folyamatosan figyeli a közelgő meccseket, és az 1 órás határidő elérésekor indítja a feldolgozást.

### 7.6 Liga / esemény prioritizálás

Ha a napi 400 kimenet nem elegendő az összes elérhető esemény lefedésére, a **nagyobb ligákat kell előnyben részesíteni**, hogy adatgazdag meccsek ne szoruljanak ki kisebb bajnokságok miatt.

> **[KÉRDÉS 2]** A ligaprioritás egy statikus, előre meghatározott lista (pl. Top 5 európai futball liga > egyéb európai ligák > világ többi része), vagy admin felületen legyen konfigurálható?
---

## 8. Képernyők és navigáció

### 8.1 Landing page (web)

Egyszerű marketing weboldal, külön a mobilalkalmazástól:
- Néhány mondat az alkalmazásról
- Átirányító link az App Store-ba és a Google Play Áruházba
- Nincs felhasználói fiókkezelés vagy alkalmazásfunkció

### 8.2 Onboarding / Jogi nyilatkozatok elfogadása

Az alkalmazás első indításakor, mielőtt bármilyen tartalom megjelenne, a felhasználónak el kell fogadnia:
- Adatvédelmi tájékoztatót
- Felhasználási feltételeket / egyéb jogi nyilatkozatokat

Az elfogadás eszközönként/fiókonként rögzítésre kerül.

### 8.3 Bejelentkezés és regisztrációs képernyő

| Lehetőség | Részletek |
|---|---|
| Email / jelszó | Hagyományos email regisztráció |
| Google (Gmail) | OAuth via Google |
| Apple bejelentkezés | Sign In with Apple |
| Folytatás vendégként | Regisztráció nélkül; csak ingyenes szint |

**Elfelejtett jelszó:**
- Email-es fiókhoz elérhető a bejelentkezési képernyőn
- A felhasználó megadja az email címét, a rendszer jelszó-visszaállító linket küld
- Google / Apple bejelentkezéssel létrehozott fióknál nem releváns (az adott platform kezeli)

> **[KÉRDÉS 4]** Az előfizetés funkció hol érhető el?

### 8.4 Főképernyő (Ingyenes szint)

**Háromszintű navigációs hierarchia:**

---

#### 1. szint — Eseménylista

- Görgethető lista a közelgő sporteseményekről csempék formájában.
- Egy csempe tartalma: csapatok / eseménynév, sportág, versenysorozat, dátum/idő.
- Narancssárga/sárga hátterű csempék; opcionálisan halvány cápa vízjel.
- A reklámok a lista aljára görgetés után jelennek meg.

---

#### 2. szint — Piaclista (esemény csempéjéből nyílik)

- Az adott eseményhez tartozó fogadási piacok csempékként listázva.
- Minden piac csempéjén belül az összes kimenetel megjelenik.
- Kimenetel formátuma: `[Kimenetel neve] [Legjobb odds] ([Overall pontszám])`
- Példa: `Mindkét csapat szerez gólt: Igen  1,7 (7,2) | Nem  1,8 (5,8)`

---

#### 3. szint — Kimenetel részletei (piac csempéjéből nyílik)

- Nincsenek további alcsempék; a külső linkek böngészőben nyílnak meg.

**Pontozási panel (minden kimenetelhez):**

| Mező | Példaérték |
|---|---|
| Likelihood pontszám | 7 |
| Odds pontszám | 6 |
| Overall pontszám | 6,6 |
| Legjobb elérhető odds | 2,1 |
| Value besorolás | Value / Strong Value / — |

**Szöveges elemzés (~10 mondat):**
- A meglévő Python backend generálja a pontozással egyidőben
- Az alkalmazásban beállított nyelven (HU/EN) kerül tárolásra és megjelenítésre
- Szakmai szempontokra fókuszál, amelyek a kimenetelt befolyásolják
- Kiemeli a kulcstényezőket (forma, sérülések, egymás elleni mérleg stb.)
- **Tilos megemlíteni** a Sportmonks predikciós százalékait vagy bármely belső adatforrást
- Független szakértői véleményként olvasható

**Bukméker linkek** — partnerbukmékereknél elérhető legjobb oddsokra mutató hivatkozások

> **[KÉRDÉS 5]** Mely bukmékerekre mutassanak a linkek? Rögzített lista, vagy dinamikusan az adott kimenetelnél legjobb oddsot kínáló bukméker jelenik meg?

---

### 8.5 Toplista képernyő (Prémium — 2 990 Ft / hó)

A nap legjobban pontozó fogadási kimeneteleinek ranglistája.

**Megjelenési feltételek (mindkettőnek teljesülnie kell):**
- Overall pontszám ≥ 7,0
- Legjobb elérhető odds ≥ 1,5

**Rendezés:** Overall pontszám szerint csökkenő sorrendben.

**Vizuális jelzők:**

| Státusz | Megjelenés |
|---|---|
| Value | Cápa ikon + „Value" felirat, ezüst betűkkel |
| Strong Value | Cápa ikon + „Strong Value" felirat, arany betűkkel |
| Nincs jelölés | Nincs külön indikátor |

- Egy elemre koppintva megnyílik a Kimenetel részletei nézet (azonos a 3. szinttel).
- Reklámok a lista alján.

> **[KÉRDÉS 7]** A prototípus hónapban hogyan jelenjen meg a Toplista a nem előfizető felhasználóknak?
> - **(a)** Teljesen rejtve a menüből
> - **(b)** Látható, de zárolt / elmosódott képernyőként, előfizetési felhívással

---

### 8.6 Profil / Fiók képernyő

A felhasználó itt tudja:
- Megtekinteni és kezelni az előfizetését
- Váltani az alkalmazás nyelvét
- Megtekinteni az elfogadott jogi dokumentumokat
- Kijelentkezni / fiókot törölni

*(Részletes dizájn még kidolgozandó)*

---

## 9. Felhasználói azonosítás és visszaélés-megelőzés

### 9.1 Regisztrációs módszerek

- Email / jelszó
- Google OAuth
- Apple Sign-In

### 9.2 Ingyenes próbaidőszak visszaélés-megelőzése

A 7 napos ingyenes próba csak **egyszer vehető igénybe valódi felhasználónként** — nem csak fiókonként. Az ügyfél specifikációja szerint pusztán új fiók létrehozásával nem szabad újra aktiválhatónak lennie.

**Javasolt megközelítés:**
- Előfizetésnél számlaszámot ad az ügyfél, amit a fiók törlésekor megtartunk. Így validáljuk, hogy később nem akar új profilból új próbaidőt igénybe venni.


---

## 10. Reklámhirdetések

**A prototípusban nem kerül implementálásra.** A reklámrendszer önálló, részletes tervezést igényel (hirdetők köre, hirdetési formátumok, hálózat kiválasztása stb.), ezért ez egy külön, későbbi fázis feladata.

---

## 11. Push értesítések

*(Az eredeti specifikációban nem szerepel)*

> **[KÉRDÉS 10]** Szükségesek-e push értesítések? Például: „A mai top tippek megérkeztek", vagy értesítés, ha magas pontozású value fogadás jelenik meg?

---

## 12. Admin / Backend felület

*(Az eredeti specifikációban nem szerepel)*

> **[KÉRDÉS 11]** Szükséges-e admin felület az üzemeltető számára? Például: felhasználók és előfizetések monitorozása, ligaprioritás-lista szerkesztése, AI által generált tartalom átnézése közzététel előtt?

---

## 13. Rendszerarchitektúra és technikai döntések

A rendszer **3 önálló komponensből** áll:

---

### 13.1 Mobil alkalmazás

| Elem | Döntés |
|---|---|
| Keretrendszer | **Flutter** ✓ |
| Platformok | iOS + Android |
| Kommunikáció | REST API a Node.js backend felé |

---

### 13.2 Mobil backend (Node.js)

Feladata: a mobilalkalmazás kiszolgálása, felhasználókezelés, előfizetés-kezelés.

| Elem | Döntés |
|---|---|
| Nyelv / runtime | **Node.js** ✓ |
| Felelősségi körök | Autentikáció, felhasználói adatok, előfizetés (IAP validáció), REST API a Flutter app felé |
| Adatbázis | Nyitott |
| Hosting | Nyitott |

A Node.js backend **olvassa** a Python poller által feltöltött adatokat (közös adatbázison keresztül), és ezeket szolgálja ki a mobilalkalmazásnak.

---

### 13.3 Python adatgyűjtő és DB-frissítő

Feladata: sportesemények lekérése, pontozás futtatása, eredmény adatbázisba írása.

| Elem | Döntés |
|---|---|
| Nyelv | **Python** ✓ |
| Jelenlegi állapot | Véletlenszerű lekérés → scoring prompt → CSV kimenet |
| Fejlesztendő | Scheduler, liga-prioritizálás, DB-írás (ld. 7. fejezet) |
| AI szöveggenerálás | A pontozó prompttal együtt fut, külön szolgáltatás nem szükséges ✓ |
| Hosting | Nyitott (önálló process / cron job) |

---

### 13.4 Közös infrastruktúra

| Elem | Döntés |
|---|---|
| Adatbázis | Nyitott — a Python poller írja, a Node.js backend olvassa |
| Hosting platform | Nyitott |

---

## 14. Logó

A BetShark logó cápa motívumot alkalmaz narancssárga/fekete színvilágban. A teljes felbontású asset-ek külön átadásra kerülnek.

> **[KÉRDÉS 12]** Ennek megtervezéséhez designer kell, kérjek tőle árajánlatot?
---

## 15. Nyitott kérdések összefoglalója az ügyfélnek

| # | Kérdés |
|---|---|
| 1 | Szükséges-e webes fizetési lehetőség (pl. Stripe), vagy elegendő az App Store / Play Store IAP? |
| 2 | A ligaprioritás statikus lista, vagy admin felületen konfigurálható? |
| 3 | Milyen sportágakat fed le az alkalmazás? Csak labdarúgás, vagy más sportágak is? |
| 4 | Az előfizetés funkció hol érhető el az alkalmazáson belül? |
| 5 | Mely bukmékerekre mutassanak a linkek? Rögzített lista vagy dinamikus? |
| 6 | Hogyan jelenjen meg a Toplista nem előfizető felhasználóknak a prototípus hónapban? |
| 7 | Szükségesek-e push értesítések? |
| 8 | Szükséges-e admin / backend kezelőfelület? |

---

*A specifikáció vége — v0.1*
