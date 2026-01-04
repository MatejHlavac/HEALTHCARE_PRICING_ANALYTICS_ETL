# ETL proces - Healthcare Pricing Analytics

## 1. Úvod a popis zdrojových dát
### Prečo sme si vybrali tento dataset a čo je cieľom?
Pre náš projekt sme si vybrali dataset, ktorý zachytáva vyjednané zmluvné ceny za zdravotnícke úkony medzi poskytovateľmi starostlivosti a poisťovňami. Hlavným dôvodom bolo to, že nás zaujala téma cien v zdravotníctve. V USA je bežné, že za ten istý zákrok zaplatí každý inú podľa toho, v ktorej je nemocnici a akú má poisťovňu. Naším cieľom je tieto dáta upratať a pripraviť analýzu, ktorá ukáže, aké veľké sú rozdiely v týchto "vyjednaných" cenách.

Dáta zachytávajú proces dohadovania cien medzi nemocnicami a poisťovňami. Obsahuje informácie o tom, kto službu poskytuje, kto ju platí a na akej konečnej sume sa dohodli.

### Čo všetko v dátach nájdeme?
V datasete sa nachádzajú hlavne textové informácie (názvy nemocníc, štáty, popisy lekárskych úkonov) a dôležité číselné údaje, ako sú samotné ceny.

### Popis tabuliek, s ktorými pracujeme:
**1. ALL_RATES:** Táto tabuľka obsahuje všetky dostupné údaje v jednej veľkej štruktúre. Nájdeme v nej informácie o cenách, poskytovateľoch aj službách. Pre náš ETL proces je to hlavný zdroj, z ktorého budeme vyberať dôležité stĺpce a čistiť ich.

**2. NEGOTIATED_RATES:** Ide o transakčnú tabuľku, kde sú uložené samotné vyjednané ceny a termíny ich platnosti. Je to kľúčový zdroj pre našu faktovú tabuľku.

**3. SERVICE_DEFINITIONS:** Táto tabuľka funguje ako číselník lekárskych úkonov. Obsahuje kódy a názvy zákrokov.

**4. PROVIDER_REFERENCES:** Tu sú uložené detaily o nemocniciach a lekároch. Obsahuje ich názvy a identifikačné čísla.

**5/6. METRICS a VOLUME:** Tieto tabuľky obsahujú už vopred vypočítané štatistiky a počty pacientov. My ich ale využívať nebudeme, pretože by v Star schéme pôsobili duplicitne. Všetky potrebné agregácie si dokážeme vypočítať sami z detailných údajov vo faktovej tabuľke.


## 1.1 ERD zdrojových dát

V našom diagrame zdrojových dát sme tabuľky rozdelili podľa toho, ako spolu v skutočnosti fungujú. Najviac nás zaujíma trojica tabuliek: ceny (**NEGOTIATED_RATES**), popisy služieb (**SERVICE_DEFINITIONS**) a zoznam nemocníc (**PROVIDER_REFERENCES**). Tie sú navzájom prepojené a tvoria kostru nášho projektu.

Keďže sme si všimli, že v zozname nemocníc chýbajú informácie o tom, v akom meste a štáte sa nachádzajú, využijeme aj tabuľku ALL_RATES. Tá nám slúži ako "záloha", z ktorej si tieto chýbajúce údaje (mesto a štát) vytiahneme a doplníme ich tam, kam patria. Zvyšné tabuľky so štatistikami (METRICS a VOLUME) v diagrame nikam nepripájame, pretože tie informácie v nich sú už vopred vypočítané a my si ich chceme v našom modeli vypočítať sami nanovo.
