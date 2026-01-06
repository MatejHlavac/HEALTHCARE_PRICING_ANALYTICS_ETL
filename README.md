# ELT proces - Healthcare Pricing Analytics

## 1. Úvod a popis zdrojových dát
### Prečo sme si vybrali tento dataset a čo je cieľom?
Pre náš projekt sme si vybrali dataset, ktorý zachytáva vyjednané zmluvné ceny za zdravotnícke úkony medzi poskytovateľmi starostlivosti a poisťovňami. Hlavným dôvodom bolo to, že nás zaujala téma cien v zdravotníctve. V USA je bežné, že za ten istý zákrok zaplatí každý inú cenu podľa toho, v ktorej je nemocnici a akú má poisťovňu. Naším cieľom je tieto dáta upratať a pripraviť analýzu, ktorá ukáže, aké veľké sú rozdiely v týchto "vyjednaných" cenách.

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

## 2. Návrh a popis dimenzionálneho modelu

V tejto časti navrhneme náš dimenzionálny model v štruktúre star schémy. Najprv si pripravíme dimenzie, teda tabuľky nemocníc, služieb a poisťovní. Keď budeme mať dimenzie hotové, vytvoríme hlavnú tabuľku faktov.

### DIM_PROVIDER  
Túto tabuľku sme vytvorili kombináciou dát zo zdrojových tabuliek `PROVIDER_REFERENCES` a `ALL_RATES` ->
- `PROVIDER_ID`(**INT**)(**PK**): Unikátne číslo každého záznamu.
- `PROVIDER_GROUP_ID`(**INT**): Pôvodné ID zo zdroja (náš spojovací kľúč).
- `PROVIDER_NAME`(**VARCHAR(128)**): Názov nemocnice (zo zdroja PROVIDER_REFERENCES).
- `NPI`(**VARCHAR(64)**): Daňové identifikačné číslo (zo zdroja PROVIDER_REFERENCES).
- `ADDRESS`(**VARCHAR(254)**): Adresa (vytiahnuté z ALL_RATES podľa `PROVIDER_GROUP_ID`).
- `CITY`(**VARCHAR(128)**: Mesto (vytiahnuté z ALL_RATES podľa `PROVIDER_GROUP_ID`).
- `STATE`(**VARCHAR(128)**: Štát (vytiahnuté z ALL_RATES podľa `PROVIDER_GROUP_ID`).
- `ZIP`(**VARCHAR(128)**: ZIP/PSČ (vytiahnuté z ALL_RATES podľa `PROVIDER_GROUP_ID`).
  
**SCD**: Typ 1 - Ak sa zmení adresa alebo názov nemocnice, zväčša nás ten predošlí údaj nezaujíma a je pre nás dôležitý iba ten aktuálny.


### DIM_SERVICE  
Túto tabuľku sme vytvorili zo zdrojovej tabuľky `SERVICE_DEFINITIONS` ->
- `SERVICE_ID`(**INT**) (**PK**): Unikátne číslo každého záznamu.
- `SERVICE_DEFINITION_ID`(**INT**): Pôvodné ID zo zdroja.
- `BILLING_CODE`(**VARCHAR(64)**): Kód zákroku.
- `BILLING_CODE_TYPE`(**VARCHAR(64)**): Typ kódu.
- `NAME`(**VARCHAR(128)**): Názov lekárskeho úkonu.
- `DESCRIPTION`(**VARCHAR(1024)**): Popis lekárskeho úkonu.

**SCD**: Typ 0 - Kódy a názvy zákrokov sú fixné medicínske štandardy.


### DIM_PAYER  
Túto tabuľku sme vytvorili kombináciou stĺpcov z `NEGOTIATED_RATES` a `METRICS` ->
- `PAYER_ID`(**INT**) (**PK**): Unikátne číslo každého záznamu.
- `PAYER`(**VARCHAR(128)**): Názov poisťovne.
- `SEGMENT`(**VARCHAR(128)**): Typ trhu (napr. Individual, Medicare - vytiahnuté z `METRICS`).
- `MARKET`(**VARCHAR(128)**): Región pôsobenia (vytiahnuté z `METRICS`).
- `NEGOTIATION_ARRANGEMENT`(**VARCHAR(64)**): Spôsob výpočtu ceny, napríklad či ide o fixnú sumu alebo o percentuálnu zľavu.

**SCD**: Typ 1 - Rovnaký prístup ako v prípade nemocnice.


### DIM_DATE  
Túto tabuľku sme vytvorili, aby sme mohli ceny a ich platnosť sledovať v čase -> 
- `DATE_ID`(**INT**) (**PK**): Unikátne číslo každého záznamu.
- `DATE`(**DATETIME**): Dátum.
- `YEAR`(**INT**): Rok.
- `QUARTER`(**INT**): Kvartál (Q1 – Q4).
- `MONTH`(**VARCHAR(64)**): Názov mesiaca.
- `MONTH_NUM`(**INT**): Číselné poradie mesiaca vrámci roka.

**SCD**: Typ 0 - Kalendár sa nemení, 1. január bude vždy 1. január.

---

### FACT_NEGOTIATED_RATES  
Táto tabuľka spája všetky dimenzie s konkrétnymi dohodnutými cenami. Každý riadok predstavuje jednu unikátnu cenu za konkrétnu službu v danej nemocnici.
- `FACT_ID`(**INT**) (**PK**): Unikátne číslo každého záznamu.
- `PROVIDER_ID`(**INT**): Odkaz na nemocnicu (prepojené na `DIM_PROVIDER`).
- `SERVICE_ID`(**INT**): Odkaz na lekársky úkon (prepojené na `DIM_SERVICE`).
- `PAYER_ID`(**INT**): Odkaz na poisťovňu (prepojené na `DIM_PAYER`).
- `EXPIRATION_DATE_ID`(**INT**): Odkaz na dátum expirácie ceny (prepojené na `DIM_DATE`).
- `CLAIM_COUNT`(**INT**): Celkový počet zrealizovaných medicínskych prípadov pre danú kombináciu poskytovateľa a služby (údaj vytiahnutý z tabuľky `VOLUME`).
- `NEGOTIATED_RATE`(**NUMBER(15,3)**): Samotná vyjednaná suma, ktorú poisťovňa platí nemocnici.
WINDOW FUNCTIONS ->
- `RATE_ORDER`(**INT**): Tento stĺpec hovorí o tom v akom poradí je konkrétna cena v porovnaní s ostatnými pre tú istú službu.

  ->`RANK() OVER (PARTITION BY SERVICE_ID ORDER BY NEGOTIATED_RATE ASC)`
  
- `MARKET_AVG`(**NUMBER(15,3)**): Tento stĺpec zobrazuje priemernú cenu na celom trhu pre danú službu.

  ->`AVG(NEGOTIATED_RATE) OVER (PARTITION BY SERVICE_ID)`


  ---

<p align="center">
  <img width="940" height="688" alt="ERD_dim_model" src="https://github.com/user-attachments/assets/e58f74b1-154b-4ad9-aa60-2a442e73b9b1" />
  <br>
  <strong>Obrázok 2</strong> ERD dimenzionálneho modelu
</p>


## 3. ELT proces v Snowflake
### 1. EXTRACT - Extrakcia dát
V tejto časti riešenia sme si prepojili náš Snowflake projekt so zdrojovým datasetom z marketplace. Cieľom je vytvoriť staging tabuľky, ktoré slúžia ako dočasné úložisko pre surové dáta.

Dáta sú importované z databázy `HEALTHCARE_PRICING_ANALYTICS_SAMPLE` a schémy `SAMPLES`.

**Príprava staging tabuliek:**
```sql  
CREATE OR REPLACE TABLE STAGE_ALL_RATES AS
SELECT * FROM HEALTHCARE_PRICING_ANALYTICS_SAMPLE.SAMPLES.ALL_RATES;

CREATE OR REPLACE TABLE STAGE_NEGOTIATED_RATES AS
SELECT * FROM HEALTHCARE_PRICING_ANALYTICS_SAMPLE.SAMPLES.NEGOTIATED_RATES;

CREATE OR REPLACE TABLE STAGE_PROVIDER_REFERENCES AS
SELECT * FROM HEALTHCARE_PRICING_ANALYTICS_SAMPLE.SAMPLES.PROVIDER_REFERENCES;

CREATE OR REPLACE TABLE STAGE_SERVICE_DEFINITIONS AS
SELECT * FROM HEALTHCARE_PRICING_ANALYTICS_SAMPLE.SAMPLES.SERVICE_DEFINITIONS;

CREATE OR REPLACE TABLE STAGE_METRICS AS
SELECT * FROM HEALTHCARE_PRICING_ANALYTICS_SAMPLE.SAMPLES.METRICS;

CREATE OR REPLACE TABLE STAGE_VOLUME AS
SELECT * FROM HEALTHCARE_PRICING_ANALYTICS_SAMPLE.SAMPLES.VOLUME;
```

Následne sme skontrolovali, či boli všetky tabuľky úspešne vytvorené a naplnené.

**Príkazy na konrolu štruktúry a čitateľnosti dát:**
```sql
SELECT * FROM STAGE_ALL_RATES LIMIT 10;
SELECT * FROM STAGE_NEGOTIATED_RATES LIMIT 10;
SELECT * FROM STAGE_PROVIDER_REFERENCES LIMIT 10;
SELECT * FROM STAGE_SERVICE_DEFINITIONS LIMIT 10;
SELECT * FROM STAGE_METRICS LIMIT 10;
SELECT * FROM STAGE_VOLUME LIMIT 10;
```

**Príkazy na kontrolu počtu záznamov v tabuľkách:**
```sql
SELECT COUNT(*) AS pocet_riadkov FROM STAGE_ALL_RATES;
SELECT COUNT(*) AS pocet_riadkov FROM STAGE_NEGOTIATED_RATES;
SELECT COUNT(*) AS pocet_riadkov FROM STAGE_PROVIDER_REFERENCES;
SELECT COUNT(*) AS pocet_riadkov FROM STAGE_SERVICE_DEFINITIONS;
SELECT COUNT(*) AS pocet_riadkov FROM STAGE_METRICS;
SELECT COUNT(*) AS pocet_riadkov FROM STAGE_VOLUME;
```
---

