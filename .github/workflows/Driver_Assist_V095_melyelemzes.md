# Driver Assist V095 – forrás-, logika- és adatbázis-audit

## Vizsgált bemenet

- Programforrás: kizárólag a `build_V085.yml` aktuális projektje.
- Adatforrás: `driver_assist_backup_2026-08-06_064826.json`.
- A V064 vagy más korábbi programverzió tartalma nem került be a módosításba.

## Ellenőrzési terjedelem

- 28 Java-forrásfájl.
- 3 egységtesztfájl.
- 24 562 Java- és tesztforrássor.
- Android manifest, erőforrás-XML-ek, Gradle-konfiguráció, GitHub Actions buildfolyamat.
- SQLite-sémák, indexek, verziók, rekordkulcsok, kategóriák és JSON/SQLite egyezés.
- Beállításmigráció, alkalmazásindítás, adatbázis-szétválasztás, import/export, Google Drive-szinkron, POI-keresés és útvonalpont-duplikáció teljes vezérlési lánca.

## A feltöltött adatállomány tényleges összetétele

Összes rekord: **28 171**.

- PARKOLO: **20 252**
- TELEPHELY, azaz Cég: **7 226**
- BOLT: **521**
- SZOLGALTATAS: **169**
- EGYEB: **3**

A `SZOLGALTATAS` rekordok a V095 fő adatbázisában `TELEPHELY`, vagyis **Cég** kategóriára normalizálódnak. Így a fő adatbázis végleges összetétele:

- Cég: **7 395**
- Bolt: **521**
- Egyéb: **3**
- új Parkoló: kezdetben **0**

A külön régi parkoló-adatbázis: **20 252 Parkoló**.

A mentés nem tartalmaz „régi/új parkoló” eredetjelzőt. Emiatt nem lehet hitelesen kiválasztani egy feltételezett 13 000 darabos régi részhalmazt. Az egyetlen veszteségmentes és reprodukálható felosztás az, hogy a mentés összes jelenlegi `PARKOLO` rekordja régi parkoló lesz, a V095 után felvitt parkolók pedig a fő adatbázisban maradnak.

Adatminőség:

- ismétlődő `record_key`: **0**;
- fő és régi adatbázis közötti kulcsátfedés: **0**;
- 20 méteren belüli koordinátaduplikátum-pár: **0**;
- hibás vagy féloldalas koordinátapár: **0**;
- mindkét koordináta nélküli rekord: **42** (18 fő, 24 régi). Ezek szöveges kereséssel megmaradnak, térbeli keresésben természetesen nem jelenhetnek meg.

## A V085-ben talált és a V095-ben javított hibák

### 1. A régi parkolók valójában nem váltak le a telepített fő adatbázisról

A V085 tartalmazott külön beágyazott parkolófájlt, de a már telepített `driver_assist_locations.db` parkolórekordjait nem mozgatta ki. Emiatt kikapcsolt régi parkolókapcsoló mellett is megmaradhattak a fő adatbázis parkolói, bekapcsolva pedig ugyanazok két forrásból is érkezhettek.

**Javítás:** a V095 első indulásakor:

1. létrejön/megnyílik a V095 régi parkoló-adatbázisa;
2. SQL-szinten átmásolódik a fő adatbázis összes addigi Parkoló rekordja;
3. a program ellenőrzi az átmásolt darabszámot;
4. csak sikeres ellenőrzés után törli ezeket a fő adatbázisból;
5. ellenőrzi a törölt darabszámot;
6. csak ezután rögzíti a migráció elkészültét.

Ha a másolás hibázik, a fő adatbázisból semmi nem törlődik. Ha az alkalmazás a két lépés között áll le, az ismételt futás idempotens: a rekordkulcs alapján újraírja a célrekordot, majd befejezi a törlést.

### 2. A V085→V095 egyszerű verzióátírás beállításvesztést okozott volna

A fő beállításfájl és a POI-kereső kapcsolója verziózott néven szerepel. Puszta átnevezéssel a V095 nem látta volna a V085 útvonalát és a „Régi parkolók” állapotát.

**Javítás:**

- a fő beállításmigráció első forrása V085;
- a régi parkolókapcsoló külön V085→V095 átvételt kapott;
- a korábbi V085 régi parkoló-adatbázis esetleges kézi importjai SQL-szinten átkerülnek a V095 adatbázisába.

### 3. A Google Drive vissza tudta volna tölteni a leválasztott parkolókat

A felhőben még a korábbi, egyesített adatállomány lehet. A V085 logikával ez a következő szinkronkor visszaszúrhatta volna a régi parkolókat a fő adatbázisba.

**Javítás:** a V095 minden letöltött és feltöltendő fő adatbázissorozatból rekordkulcs alapján kiszűri a külön régi adatbázisban már meglévő rekordokat. Ha régi parkolót távolít el a felhőpéldányból, a megtisztított fő adatbázist vissza is tölti.

Fontos: mindkét készüléket V095-re kell frissíteni. Egy tovább futó V085 készülék továbbra is képes lenne a régi, egyesített felhőfájlt újra feltölteni.

### 4. A duplikált útvonalpont megváltoztathatta a sorrendet

A V085 előbb beszúrta az új pontot a kért helyre, és csak utána vonta össze a duplikátumokat. Ha az új példány korábbi pozícióra került, az összevonás a meglévő pontot is erre az új helyre húzhatta.

**Javítás:** a program beszúrás előtt végigkeresi a meglévő útvonalat. Egyezéskor:

- nem szúr be új sort;
- az eredeti listahely változatlan marad;
- a két rekordot mezőnként egyesíti;
- a részletesebb név/cím marad;
- a megjegyzés és forrásinformáció kiegészülhet;
- nincs megszakító párbeszédablak;
- a munkafolyamat folytatódik.

### 5. A teljes rekordcsere hasznos mezőket veszíthetett

A korábbi logika egy összpontszám alapján teljes rekordot cserélt. Egy jobb nevű új rekord felülírhatta a régi rekord külön megjegyzését vagy forrásnevét.

**Javítás:** mezőnkénti összevonás került be.

### 6. Nem volt külön régi parkoló-export

Importálni lehetett a külön adatbázisba, de visszaadni/menteni nem.

**Javítás:** új menüpont:

`RÉGI PARKOLÓK EXPORTÁLÁSA – JSON`

### 7. A két JSON-fájlt fel lehetett volna cserélni

A korábbi JSON nem jelezte, hogy normál vagy régi parkoló-adatbázis.

**Javítás:** a V095 exportok `database_role` mezőt kapnak:

- `main`
- `legacy_parking`

A program a rossz importmenüben egyértelmű hibaüzenettel visszautasítja a másik fájlt. A régi, jelöletlen mentések kompatibilitási okból továbbra is beolvashatók.

### 8. A kézzel mentett névtelen pont Egyéb kategóriára állhatott

Ez ellentétes volt azzal a szabállyal, hogy a kézzel mentett pont alapértelmezése Cég legyen.

**Javítás:** minden kézi/térképi új mentés alapból Cég. Bolt, Parkoló vagy Egyéb csak tudatos kategóriaválasztással lesz.

### 9. Elavult egységteszt

A teszt még azt várta, hogy ismeretlen külső kategóriából új kategória keletkezzen. Ez ellentmondott a négy stabil kategóriának, és a GitHub build tesztlépését megbuktathatta.

**Javítás:** a teszt most azt ellenőrzi, hogy az ismeretlen külső kategória Egyéb lesz.

### 10. Nem minden belső verzióazonosító volt 95

A gyorsítótár promptverziója még 85 maradt volna.

**Javítás:** fájlnév, workflow, projekt, alkalmazáscímke, `versionCode`, `versionName`, adatbázisverzió, JSON-séma, felhőséma, gyorsítótárséma és promptverzió 95.

## V095 működés a gyakorlatban

### Meglévő V085 telepítés frissítése

A `Driver_Assist_V095.apk` telepítése után az első indulás automatikusan elvégzi a szétválasztást. Nem kell kézzel SQLite-fájlt másolni vagy szerkeszteni.

A végeredmény:

- fő adatbázis: Cég, Bolt, új Parkoló, Egyéb;
- külön régi adatbázis: az összes addigi Parkoló;
- „Régi parkolók” kikapcsolva: csak a fő adatbázis kereshető;
- „Régi parkolók” bekapcsolva: a fő adatbázis és a külön régi parkolóforrás együtt kereshető;
- a V095 után kézzel vagy normál importtal felvitt parkoló a fő adatbázisban marad.

### Tiszta telepítés vagy kézi visszaállítás

1. Normál `IMPORTÁLÁS` menü: `driver_assist_main_v095.json`.
2. `RÉGI PARKOLÓK IMPORTÁLÁSA` menü: `driver_assist_legacy_parking_v095.json`.

A V095 APK eleve tartalmazza a 20 252 régi parkolót, ezért tiszta V095 telepítésnél a második import csak akkor szükséges, ha később módosított vagy külön exportált régi parkolófájlt akarsz visszatölteni.

A `.db` fájlok ellenőrzési/archiválási SQLite-adatbázisok. Az alkalmazás menüjében a JSON-fájlokat kell importálni; a nyers `.db` fájl kézi bemásolását nem javasolt használni.

## Ellenőrzések eredménye

Sikeres:

- YAML-struktúra betöltése;
- beágyazott projekt Base64-visszafejtése;
- projektarchívum SHA-256 ellenőrzése;
- az archívum visszabontása és forrásauditja;
- XML-parszolás;
- Java szintaktikai regressziókeresés;
- tiszta Java útvonalsorrend-, KML- és GPX-próba;
- mindkét JSON-fájl rekordszám-, kategória-, kulcs- és szerepellenőrzése;
- mindkét SQLite-fájl `integrity_check` ellenőrzése;
- mindkét SQLite `user_version = 95`;
- JSON és SQLite rekordkulcsainak teljes egyezése;
- szétválasztási SQL szimulációja: 20 252 átmásolt és törölt Parkoló, 7 919 megmaradó fő rekord.

Helyben teljes Android APK/AAB fordítás nem történt, mert ebben a futtatókörnyezetben nincs Android SDK/Gradle függőségtár. A `build_V095.yml` GitHub Actions folyamata a forrásaudit után futtatja a `testReleaseUnitTest`, `assembleRelease` és `bundleRelease` lépéseket; tényleges fordítási hibát ez fog véglegesen kizárni.
