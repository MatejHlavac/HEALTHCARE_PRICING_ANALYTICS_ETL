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

### 2. Load
V tejto fáze sme dáta zo staging tabuliek preniesli do finálneho multidimenzionálneho modelu. V rámci architektúry ELT sme surové dáta fyzicky usporiadali do štruktúry star schémy.


## Dimenzia DIM_PROVIDER  
**Vytvorenie tabuľky:**
```sql
CREATE OR REPLACE TABLE DIM_PROVIDER (
    PROVIDER_ID INT IDENTITY(1,1) PRIMARY KEY,
    NPI VARCHAR(64),
    PROVIDER_GROUP_ID VARCHAR(128),
    PROVIDER_NAME VARCHAR(256),
    ADDRESS VARCHAR(256),
    CITY VARCHAR(128),
    STATE VARCHAR(128),
    ZIP VARCHAR(128)
);
```


**Vloženie dát do tabuľky:**
```sql
INSERT INTO DIM_PROVIDER (NPI, PROVIDER_GROUP_ID, PROVIDER_NAME, ADDRESS, CITY, STATE, ZIP)
SELECT DISTINCT 
    NPI, 
    PROVIDER_GROUP_ID,
    PROVIDER_NAME, 
    ADDRESS, 
    CITY, 
    STATE, 
    ZIP
FROM STAGE_ALL_RATES
WHERE NPI IS NOT NULL;
```
Údaje sme čerpali priamo zo staging tabuľky `STAGE_ALL_RATES`, ktorá obsahovala kompletné informácie o názvoch aj lokalitách poskytovateľov.

---

## Dimenzia DIM_PAYER:  
**Vytvorenie tabuľky:**
```sql
CREATE OR REPLACE TABLE DIM_PAYER (
    PAYER_ID INT IDENTITY(1,1) PRIMARY KEY,
    PAYER VARCHAR(128),
    NEGOTIATION_ARRANGEMENT VARCHAR(64)
);
```


**Vloženie dát do tabuľky:**
```sql
INSERT INTO DIM_PAYER (PAYER, NEGOTIATION_ARRANGEMENT)
SELECT DISTINCT 
    PAYER, 
    NEGOTIATION_ARRANGEMENT
FROM STAGE_ALL_RATES
WHERE PAYER IS NOT NULL;
```
Údaje sme extrahovali z tabuľky `STAGE_ALL_RATES`.

---

## Dimenzia DIM_SERVICE:  
**Vytvorenie tabuľky:**
```sql
CREATE OR REPLACE TABLE DIM_SERVICE (
    SERVICE_ID INT IDENTITY(1,1) PRIMARY KEY,
    SERVICE_DEFINITION_ID INT,
    BILLING_CODE VARCHAR(64),
    BILLING_CODE_TYPE VARCHAR(64),
    NAME VARCHAR(128),
    DESCRIPTION VARCHAR(1024)
);
```


**Vloženie dát do tabuľky:**
```sql
INSERT INTO DIM_SERVICE (SERVICE_DEFINITION_ID, BILLING_CODE, BILLING_CODE_TYPE, NAME, DESCRIPTION)
SELECT DISTINCT
    SERVICE_DEFINITION_ID,
    BILLING_CODE,
    BILLING_CODE_TYPE,
    NAME,
    DESCRIPTION
FROM STAGE_SERVICE_DEFINITIONS
WHERE SERVICE_DEFINITION_ID IS NOT NULL;
```

---

## Dimenzia DIM_DATE:
**Vytvorenie tabuľky:**
```sql
CREATE OR REPLACE TABLE DIM_DATE (
    DATE_ID INT IDENTITY(1,1) PRIMARY KEY,
    DATE DATETIME,
    YEAR INT,
    QUARTER INT,
    MONTH VARCHAR(64),
    MONTH_NUM INT
);
```


**Vloženie dát do tabuľky:**
```sql
INSERT INTO DIM_DATE (DATE, YEAR, QUARTER, MONTH, MONTH_NUM)
SELECT DISTINCT
    DATE(EXPIRATION_DATE) AS DATE,
    YEAR(EXPIRATION_DATE) AS YEAR,
    CEIL(MONTH(EXPIRATION_DATE)/3.0) AS QUARTER,
    TO_CHAR(EXPIRATION_DATE, 'Month') AS MONTH,
    MONTH(EXPIRATION_DATE) AS MONTH_NUM
FROM STAGE_NEGOTIATED_RATES
WHERE EXPIRATION_DATE IS NOT NULL;
```
Táto tabuľka slúži na sledovanie dátumov expirácie cien, čo umožňuje presne určiť, ktoré zmluvné podmienky sú v danom momente aktuálne.


V každej tabuľke sme si vytvorili vlastné ID pomocou príkazu `IDENTITY`. Tento príkaz hovorí databáze, aby za nás automaticky generovala unikátne poradové čísla pre každý nový riadok, ktorý do tabuľky vložíme. Pre databázu je oveľa jednoduchšie a rýchlejšie spájať tabuľky cez jednoduché čísla než cez dlhé texty (ako sú mená poisťovní alebo NPI kódy).


## Faktová tabuľka: FACT_NEGOTIATIONS_RATE

Faktová tabuľka je najdôležitejšou časťou nášho dátového skladu. Zatiaľ čo v dimenziách máme len zoznamy, tu sa nachádzajú samotné čísla a výsledky, ktoré chceme analyzovať.

**Vytvorenie tabuľky:**
```sql
CREATE OR REPLACE TABLE FACT_NEGOTIATIONS_RATE (
    FACT_RATE_ID INT AUTOINCREMENT PRIMARY KEY,
    PROVIDER_ID INT NOT NULL,
    PAYER_ID INT NOT NULL,
    SERVICE_ID INT NOT NULL,
    EXPIRATION_DATE_ID INT NOT NULL,
    NEGOTIATED_RATE DECIMAL(15,3),
    RATE_ORDER INT,
    MARKET_AVG DECIMAL(15,3)
);
```

**Vloženie dát do tabuľky:**
```sql
INSERT INTO FACT_NEGOTIATIONS_RATE (
    PROVIDER_ID,
    PAYER_ID,
    SERVICE_ID,
    EXPIRATION_DATE_ID,
    NEGOTIATED_RATE,
    RATE_ORDER,
    MARKET_AVG
)
SELECT
    dp.PROVIDER_ID,
    dpa.PAYER_ID,
    ds.SERVICE_ID,
    dd.DATE_ID,
    nr.NEGOTIATED_RATE,
    RANK() OVER (PARTITION BY ds.SERVICE_ID ORDER BY nr.NEGOTIATED_RATE ASC),
    AVG(nr.NEGOTIATED_RATE) OVER (PARTITION BY ds.SERVICE_ID)
FROM STAGE_NEGOTIATED_RATES nr
JOIN DIM_PROVIDER dp 
    ON nr.PROVIDER_GROUP_ID = dp.PROVIDER_GROUP_ID
JOIN DIM_PAYER dpa 
    ON nr.PAYER = dpa.PAYER
JOIN DIM_SERVICE ds 
    ON nr.SERVICE_DEFINITION_ID = ds.SERVICE_DEFINITION_ID
JOIN DIM_DATE dd 
    ON CAST(nr.EXPIRATION_DATE AS DATE) = CAST(dd.DATE AS DATE);
```

Aby sme do faktov dostali správne ID čísla, prepojili sme staging tabuľku s našimi dimenziami. Napríklad sme povedali databáze: „Nájdi mi v dimenzii poskytovateľov ID pre nemocnicu, ktorá má tento konkrétny Group ID“.

### 3. Transform

**Čistenie dát a technické úpravy**  
**Pretypovanie (Casting):** Pri finančných hodnotách sme použili typ `DECIMAL(15,3)`, pre presnosť na tri desatinné miesta. Pri dátumoch sme využili funkciu `CAST(... AS DATE)`, aby sme zjednotili formáty a odstránili nepotrebnú časovú zložku.
**Filtrovanie NULL hodnôt:** Pomocou podmienky `WHERE ... IS NOT NULL` sme zo spracovania vylúčili neúplné záznamy, ktoré by inak znehodnotili výslednú analýzu.
**Deduplikácia:** Pôvodné dáta obsahovali množstvo opakujúcich sa záznamov. Použitím príkazu `SELECT DISTINCT` sme vyfiltrovali unikátne záznamy pre naše dimenzie, čím sme znížili objem dát a zvýšili prehľadnosť.


**SCD typy**  
Pre každú dimenziu sme zvolili vhodný prístup k správe zmien:

**SCD Typ 0 (Fixné):** Použili sme pre `DIM_SERVICE` a `DIM_DATE`. Tieto údaje (názvy lekárskych kódov alebo kalendárne dni) sme považovali za nemenné.  
**SCD Typ 1 (Aktuálne):** Použili sme pre `DIM_PROVIDER` a `DIM_PAYER`. Ak sa napríklad zmení adresa nemocnice alebo názov poisťovne, starý údaj sa jednoducho prepíše novým. Naším cieľom bolo mať v reporte vždy aktuálne informácie o subjektoch.


**Window Functions**  
Najdôležitejšou časťou transformácie vo faktovej tabuľke bolo použitie Window Functions. Tie nám umožnili obohatiť každý riadok o dôležitý kontext bez zložitého zoskupovania dát:

1.  **Poradie cien (RANK):**
    Pomocou `RANK() OVER (PARTITION BY ds.SERVICE_ID ORDER BY nr.NEGOTIATED_RATE ASC)` sme každému poskytovateľovi priradili poradie v rámci konkrétnej služby.
    **Význam:** Používateľ hneď uvidí, kto je pre daný zákrok najlacnejší (poradie 1).
2.  **Trhový priemer (AVG):**
    Pomocou `AVG(nr.NEGOTIATED_RATE) OVER (PARTITION BY ds.SERVICE_ID)` sme vypočítali priemernú cenu služby naprieč celým trhom.
    **Význam:** To nám umožňuje okamžite porovnať konkrétnu cenu s priemerom danej služby v jednom riadku.

Tieto transformácie sme vykonali priamo počas nahrávania dát.

## 4. Vizualizácia dát  
Dashboard obsahuje vizualizácie, ktoré poskytujú základný prehľad o vyjednaných cenách zdravotníckych služieb, poskytovateľoch a cenovej štruktúre trhu. Cieľom vizualizácií je odpovedať na dôležité analytické otázky, ako napríklad ktorí poskytovatelia ponúkajú najviac služieb alebo ako sa líšia ceny jednotlivých služieb v závislosti od ich poradia na trhu. 

### 1. Počet služieb podľa poskytovateľov

Ktorí poskytovatelia ponúkajú najväčší počet rôznych zdravotníckych služieb?
**Použitý SQL dotaz:**  
```sql
SELECT
    dp.PROVIDER_NAME,
    COUNT(DISTINCT f.SERVICE_ID) AS SERVICE_COUNT
FROM FACT_NEGOTIATIONS_RATE f
JOIN DIM_PROVIDER dp ON f.PROVIDER_ID = dp.PROVIDER_ID
GROUP BY dp.PROVIDER_NAME
ORDER BY SERVICE_COUNT DESC
LIMIT 20;
```

**Popis vizualizácie:**  
Táto vizualizácia zobrazuje 20 poskytovateľov zdravotnej starostlivosti, ktorí majú v databáze evidovaný najväčší počet rôznych zdravotníckych služieb. Z údajov je zrejmé, že niektorí poskytovatelia pokrývajú výrazne širšie spektrum služieb než ostatní.

### 2. Priemerná cena vs. poradie ceny pre jednotlivé služby  

Ako sa mení priemerná vyjednaná cena služby v závislosti od jej poradia ceny na trhu?
**Použitý SQL dotaz:**  
```sql
SELECT
    ds.NAME AS service_name,
    f.RATE_ORDER,
    ROUND(AVG(f.NEGOTIATED_RATE), 2) AS avg_rate
FROM FACT_NEGOTIATIONS_RATE f
JOIN DIM_SERVICE ds ON f.SERVICE_ID = ds.SERVICE_ID
GROUP BY ds.NAME, f.RATE_ORDER
ORDER BY ds.NAME, f.RATE_ORDER;
```

**Popis vizualizácie:**  
Táto vizualizácia zobrazuje vzťah medzi poradím ceny služby - `RATE_ORDER` a jej priemernou vyjednanou cenou. Poradie ceny určuje relatívnu pozíciu konkrétnej ceny medzi ostatnými cenami tej istej služby, kde hodnota 1 predstavuje najnižšiu cenu. Vizualizácia je prezentovaná formou heatmapy, kde každý riadok reprezentuje konkrétnu zdravotnícku službu a každý stĺpec predstavuje poradie ceny. Farebná intenzita buniek vyjadruje výšku priemernej ceny. 

### 3. Cenové rozpätie služieb podľa poskytovateľov
Ktorí poskytovatelia majú najväčší počet služieb a aké sú minimálne a maximálne ceny ich služieb?

**Použitý SQL dotaz:**  
```sql
SELECT
    dp.PROVIDER_NAME,
    COUNT(DISTINCT f.SERVICE_ID) AS service_count,
    MIN(f.NEGOTIATED_RATE) AS min_rate,
    MAX(f.NEGOTIATED_RATE) AS max_rate
FROM FACT_NEGOTIATIONS_RATE f
JOIN DIM_PROVIDER dp ON f.PROVIDER_ID = dp.PROVIDER_ID
GROUP BY dp.PROVIDER_NAME
ORDER BY service_count DESC
LIMIT 5;
```

**Popis vizualizácie:**  
Táto vizualizácia zobrazuje päť poskytovateľov s najväčším počtom rôznych zdravotníckych služieb a zároveň ukazuje minimálnu a maximálnu vyjednanú cenu pre tieto služby. 

---

<p align="center">
  <img width="976" height="436" alt="grafy_123" src="https://github.com/user-attachments/assets/28770c4f-3edd-4400-9fd1-4b4eca7fc48d" />
  <br>
  <strong>Obrázok 3</strong> Obrázky grafov 1, 2 a 3
</p>

---

### 4. Top 10 miest s najvyššími priemernými cenami  
V ktorých mestách v rámci štátu sú vyjednané ceny za zdravotnícke služby v priemere najvyššie?

**Použitý SQL dotaz:**  
```sql
SELECT 
    p.CITY AS Mesto,
    ROUND(AVG(f.NEGOTIATED_RATE), 2) AS Priemerna_Cena
FROM FACT_NEGOTIATIONS_RATE f
JOIN DIM_PROVIDER p ON f.PROVIDER_ID = p.PROVIDER_ID
GROUP BY p.CITY
ORDER BY Priemerna_Cena DESC
LIMIT 10;
```

**Popis vizualizácie:**  
Táto vizualizácia identifikuje desať miest, v ktorých náš poisťovateľ vyjednal v priemere najvyššie ceny za poskytované služby. Pomáha nám pochopiť geografické rozdiely v nákladoch v rámci jedného štátu a určiť lokality, ktoré sú z hľadiska zdravotnej starostlivosti najnákladnejšie.

<p align="center">
  <img width="1088" height="429" alt="graf_4" src="https://github.com/user-attachments/assets/16887252-6229-43ac-a77a-a5b9e55e2214" />
  <br>
  <strong>Obrázok 3</strong> Obrázok grafu 4
</p>

---

### 5. Analýza cenových rozdielov v rámci rovnakých služieb  
Aké veľké sú rozdiely medzi najlacnejšou a najdrahšou vyjednanou cenou pre jednotlivé zdravotnícke úkony?

```sql
SELECT 
    ds.NAME AS Sluzba,
    MIN(f.NEGOTIATED_RATE) AS Najnizsia_Cena,
    MAX(f.NEGOTIATED_RATE) AS Najvyssia_Cena,
    (MAX(f.NEGOTIATED_RATE) - MIN(f.NEGOTIATED_RATE)) AS Rozdiel_v_Eurach
FROM FACT_NEGOTIATIONS_RATE f
JOIN DIM_SERVICE ds ON f.SERVICE_ID = ds.SERVICE_ID
GROUP BY ds.NAME
ORDER BY Rozdiel_v_Eurach DESC;
```

**Popis vizualizácie:** 
Táto vizualizácia poukazuje na extrémnu variabilitu cien za identické lekárske úkony v rámci jedného štátu. Pre poisťovateľa je to kľúčový podklad pre optimalizáciu nákladov – identifikuje služby, pri ktorých by mal vyjednať lepšie podmienky s drahšími poskytovateľmi tak, aby sa priblížili k minimálnej cene na trhu.

<p align="center">
  <img width="1101" height="421" alt="graf_5" src="https://github.com/user-attachments/assets/68de25c6-01e3-4d21-81d9-8e9064533f73" />
  <br>
  <strong>Obrázok 3</strong> Obrázok grafu 5
</p>





