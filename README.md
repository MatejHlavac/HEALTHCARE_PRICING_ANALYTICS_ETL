# ETL proces - Healthcare Pricing Analytics

## 1. Úvod a popis zdrojových dát
### Prečo sme si vybrali tento dataset a čo je cieľom?
Pre náš projekt sme si vybrali dataset, ktorý zachytáva vyjednané zmluvné ceny za zdravotnícke úkony medzi poskytovateľmi starostlivosti a poisťovňami. Hlavným dôvodom bolo to, že nás zaujala téma cien v zdravotníctve. V USA je bežné, že za ten istý zákrok zaplatí každý inú podľa toho, v ktorej je nemocnici a akú má poisťovňu. Naším cieľom je tieto dáta upratať a pripraviť analýzu, ktorá ukáže, aké veľké sú rozdiely v týchto "vyjednaných" cenách.

Dáta zachytávajú proces dohadovania cien medzi nemocnicami a poisťovňami. Obsahuje informácie o tom, kto službu poskytuje, kto ju platí a na akej konečnej sume sa dohodli.

### Čo všetko v dátach nájdeme?
V datasete sa nachádzajú hlavne textové informácie (názvy nemocníc, štáty, popisy lekárskych úkonov) a dôležité číselné údaje, ako sú samotné ceny.

### Popis tabuliek, s ktorými pracujeme:
**1. ALL_RATES:** Hoci táto tabuľka obsahuje všetky údaje v jednej veľkej štruktúre, pre náš projekt slúži primárne ako podporný zdroj. Využívame ju na vytiahnutie doplňujúcich atribútov (napr. mesto a štát), ktoré v základnom zozname poskytovateľov chýbali.

**2. NEGOTIATED_RATES:** Ide o hlavnú transakčnú tabuľku, kde sú uložené samotné vyjednané ceny a termíny ich platnosti. Spolu s tabuľkami služieb a poskytovateľov tvorí základ pre našu faktovú tabuľku.

**3. SERVICE_DEFINITIONS:** Táto tabuľka funguje ako číselník lekárskych úkonov. Obsahuje oficiálne kódy a názvy zákrokov, čo nám umožňuje presne identifikovať, za čo sa platí.

**4. PROVIDER_REFERENCES:** Tu sú uložené základné detaily o nemocniciach a lekároch, ako sú ich mená a identifikačné čísla (NPI).

**5/6. METRICS a VOLUME:** Tieto tabuľky obsahujú vopred vypočítané štatistiky a počty pacientov. V našom modeli ich nevyužívame, aby sme sa vyhli duplicite dát. Všetky potrebné výpočty a agregácie si budeme realizovať sami priamo z detailných údajov vo faktovej tabuľke.


## 1.1 ERD zdrojových dát

V našom diagrame zdrojových dát sme tabuľky rozdelili do logických celkov podľa toho, ako spolu v skutočnosti súvisia. Hlavnú kostru projektu tvorí trojica tabuliek: **NEGOTIATED_RATES** (ceny), **SERVICE_DEFINITIONS** (popisy služieb) a **PROVIDER_REFERENCES** (zoznam nemocníc). Tieto tabuľky sú navzájom prepojené cez identifikačné kódy, čo nám umožňuje spárovať konkrétnu cenu s konkrétnym úkonom a nemocnicou.

Ostatné tabuľky v diagrame stoja samostatne, pretože v tejto surovej podobe nemajú na hlavnú trojicu vytvorené priame väzby. Tabuľka **ALL_RATES** slúži ako denormalizovaný zdroj, z ktorého budeme neskôr čerpať chýbajúce informácie o lokalite nemocníc (CITY a STATE). Tabuľky **METRICS** a **VOLUME** obsahujú predbežné štatistické údaje, ktoré v našom finálnom modeli nebudeme priamo využívať, keďže si vlastné metriky vypočítame sami z detailných dát.

<p align="center">
  <img width="1158" height="890" alt="ERD_source_data" src="https://github.com/user-attachments/assets/b2a4b2ae-f80a-4ab1-8f59-fea037a5e160" />
  <br>
  <em><strong>Obrázok 1</strong> ERD zdrojových dát</em>
</p>


