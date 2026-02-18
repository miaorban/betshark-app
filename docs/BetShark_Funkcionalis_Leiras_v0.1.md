# BetShark — Funkcionális leírás v0.1

> **Dátum:** 2026-02-18

---

## Az alkalmazásról

A **BetShark** egy labdarúgás-fogadás elemző mobilalkalmazás, amely naponta feldolgozza a közelgő focimeccsek fogadási piacait, és minden kimenetelre pontszámot, odds-összehasonlítást, valamint szöveges elemzést készít. A cél, hogy a felhasználó egy helyen, érthetően lássa, mely fogadások kínálnak valódi értéket — anélkül, hogy ezt saját maga kelljen kikalkulálnia.

Az alkalmazás iOS és Android platformon érhető el, magyar és angol nyelven.

---

## Megjelenés

Az alkalmazás fekete hátterű, narancssárga-sárga kiemelőszínekkel épül fel. A navigáció csempékre alapul: minden esemény, piac és kimenetel egy-egy kártyaként jelenik meg. A márkajelzés cápa motívumot alkalmaz, amely az értékes fogadásoknál is visszaköszön jelzőként.

---

## Hozzáférési szintek

Az alkalmazás két szinten érhető el:

| Szint | Tartalom | Ár |
|---|---|---|
| **Ingyenes** | Főképernyő — események böngészése, piacok és elemzések megtekintése | Ingyenes, regisztráció sem szükséges |
| **Prémium** | Toplista — a nap legjobb fogadási lehetőségeinek napi ranglistája | 2 990 Ft / hó |

- Az első **7 nap ingyenesen** kipróbálható a prémium szint.
- A próbaidőszak felhasználónként egyszer vehető igénybe.
- Az előfizetés az App Store-on és a Google Play Áruházon keresztül kezelhető.

---

## Első indítás

Az alkalmazás első megnyitásakor a felhasználónak el kell fogadnia az adatvédelmi tájékoztatót és a felhasználási feltételeket. Ez csak egyszer szükséges.

---

## Bejelentkezés és regisztráció

A főképernyő regisztráció nélkül, vendégként is elérhető. Fiók létrehozása csak az előfizetéshez szükséges.

Regisztrációs lehetőségek:
- **Email-cím és jelszó**
- **Google-fiók** (Gmail)
- **Apple-fiók** (Sign in with Apple)

**Elfelejtett jelszó:** Email-es fiókhoz a bejelentkezési képernyőn elérhető a jelszó-visszaállítás. A felhasználó megadja az email-címét, és visszaállító linket kap. Google vagy Apple fiókkal belépők esetén ezt a megfelelő platform kezeli.

---

## Főképernyő — Eseményböngészés (ingyenes)

A főképernyő a közelgő sportesemények listáját mutatja, csempékbe rendezve. Minden csempén látható a két csapat neve, a versenysorozat és a meccs időpontja.

### Piacok megtekintése

Egy eseményre koppintva megnyílik az ahhoz tartozó fogadási piacok listája (pl. végeredmény, mindkét csapat szerez gólt, félidei eredmény stb.). Minden piac csempéjén az összes kimenetel megjelenik a hozzá tartozó legjobb elérhető odds-szal és overall pontszámmal együtt.

**Példa:**
> Mindkét csapat szerez gólt — Igen: 1,7 **(7,2)** | Nem: 1,8 **(5,8)**

### Kimenetel részletes nézete

Egy piacra koppintva megnyílik a részletes nézet, amely minden kimenetelhez a következőket tartalmazza:

**Pontszámok:**

| Mező | Leírás |
|---|---|
| Likelihood pontszám | Mennyire valószínű a kimenetel a rendelkezésre álló adatok alapján |
| Odds pontszám | Mennyire kedvező az elérhető odds ehhez a valószínűséghez képest |
| Overall pontszám | A két szempont súlyozott összesítése (0–10 skálán) |
| Legjobb elérhető odds | A partnerbukmékereknél aktuálisan elérhető legjobb odds |
| Value besorolás | Ha az odds érdemben meghaladja a kalkulált valószínűséget: Value vagy Strong Value jelölés |

**Szöveges elemzés (~10 mondat):**
Az alkalmazás minden kimenetelhez szöveges indoklást is megjelenít. Ez szakmai szempontokat emel ki: a csapatok aktuális formáját, sérüléseket, egymás elleni mérleget és az eredményt befolyásoló egyéb tényezőket. Az elemzés független szakértői véleményként olvasható.

---

## Toplista — Prémium funkció (2 990 Ft / hó)

A Toplista a nap legjobb fogadási lehetőségeinek összesített napi ranglistája.

**Megjelenési feltételek** (mindkettőnek teljesülnie kell):
- Overall pontszám legalább 7,0
- Legjobb elérhető odds legalább 1,5

**Rendezés:** overall pontszám szerint, csökkenő sorrendben.

**Value jelölések:**

| Jelölés | Megjelenés | Jelentés |
|---|---|---|
| **Value** | Cápa ikon + ezüst „Value" felirat | Az odds érdemben meghaladja a kalkulált valószínűséget |
| **Strong Value** | Cápa ikon + arany „Strong Value" felirat | Az odds kiemelkedően meghaladja a kalkulált valószínűséget |

Egy elemre koppintva megnyílik a kimenetel teljes részletes nézete (pontszámok, szöveges elemzés) — ugyanúgy, mint a főképernyőn.

---

## Profil és fiókkezelés

A profilképernyőn a felhasználó:
- Megtekintheti és kezelheti az előfizetését
- Válthat az alkalmazás nyelve (magyar / angol) között
- Megtekintheti az elfogadott jogi dokumentumokat
- Kijelentkezhet, illetve törölheti a fiókját

---

## Landing oldal (web)

Az alkalmazáshoz egy egyszerű webes bemutatkozó oldal is tartozik, amelyen néhány mondatban szerepel az alkalmazás leírása, valamint közvetlen linkek az App Store-ba és a Google Play Áruházba.

---

## Nyitott kérdések — döntést igényelnek

Az alábbi pontokon az ügyfél visszajelzése szükséges a véglegesítéshez:

| # | Kérdés |
|---|---|
| 1 | Szükséges-e webes előfizetési lehetőség is, vagy elegendő az App Store / Play Store? |
| 2 | A megjelenő bajnokságok sorrendje egy rögzített lista legyen, vagy ezt a jövőben rugalmasan lehessen módosítani? |
| 3 | Az előfizetési képernyő hol legyen elérhető az alkalmazáson belül? (pl. Toplista zárt képernyőjéről, profil menüből, stb.) |
| 4 | A prototípus hónapban a Toplista hogyan jelenjen meg a nem előfizető felhasználóknak? **(a)** Teljesen rejtve, vagy **(b)** látható, de zárolt képernyőként, előfizetési felhívással? |
| 5 | Szükségesek-e push értesítések? (pl. „A mai tippek megérkeztek", vagy értesítés kiemelkedő value fogadásról) |
| 6 | Szükséges-e egy kezelőfelület az üzemeltető számára? (pl. felhasználók és előfizetések áttekintése) |

---

*A leírás vége — v0.1*
