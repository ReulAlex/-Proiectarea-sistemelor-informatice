# Laboratorul 2: Cuantifică citirile Dashboard-ului

**Student:** Reul Alexei · **Grupa:** Infa-241 · **Data:** 13 octombrie 2026

## 0. Presupuneri generale

- Piața de referință este cea din SUA (NYSE, Nasdaq, NYSE American). Toate orele sunt în America/New\_York (ET); decalajul față de Chișinău este de 6–7 ore, în funcție de trecerea la ora de vară/iarnă.
- Fiecare acțiune enumerată a Utilizatorului creează o cerere către Dashboard (1 acțiune = 1 cerere). O cerere către Dashboard nu creează automat o cerere către Market Data Provider; traficul către furnizor nu este estimat în acest laborator.
- Procentele de participare se aplică independent fiecărui rând: același Utilizator poate apărea în mai multe rânduri.
- Unități: RPS = cereri pe secundă; 1 MB = 1,000,000 B; 1 GB = 1,000,000,000 B. Se folosește virgula pentru mii și punctul pentru zecimale, ca în enunț.
- Rezultatele nerotunjite sunt păstrate pe tot parcursul calculului; singura valoare rotunjită (în sus) este ținta finală de capacitate de la deschiderea pieței.

## 1. Cerințe de calitate

Fiecare cerință are forma: **măsură + țintă + condiție de operare**. Un răspuns la o citire se încadrează într-una dintre clase: (A) date corecte, (B) date corecte marcate vizibil ca învechite, (C) rezultat indisponibil marcat explicit, (D) eroare (5xx sau timeout). Clasele A, B și C sunt răspunsuri valide; D este eșec.

### 1.1 Latența citirilor

| ID | Măsură | Țintă | Condiție de operare |
| --- | --- | --- | --- |
| L1 | Latența Overview, Filter, History, Watchlist, Search: timpul de la primirea primului octet al cererii HTTP la edge-ul Dashboard-ului până la scrierea ultimului octet al răspunsului; percentila 95 (p95) calculată pe ferestre de 5 minute, separat pentru fiecare citire | p95 ≤ 2,000 ms | Ore de tranzacționare (definite în 1.2), la încărcare de până la ținta de la deschiderea pieței pentru nivelul ales: 103 / 1,030 / 10,296 RPS |
| L2 | Latența Stock price, măsurată ca la L1 | p95 ≤ 500 ms (presupunere: „par imediate” înseamnă o limită mai strictă decât 2 s) | La fel ca L1 |
| L3 | Latența de coadă, toate citirile, p99 | p99 ≤ 4,000 ms | La fel ca L1 |

**Ce se întâmplă la limita de 2 secunde:**

- O citire care se termină la exact 2,000 ms este în țintă (limita este ≤).
- O citire de peste 2,000 ms este „lentă”; este permisă pentru cel mult 5% din citiri (de aici p95), iar peste 4,000 ms pentru cel mult 1% (p99).
- O citire care nu se termină în 5,000 ms este timeout: se numără ca 5,000 ms în distribuția de latență, ca eșec la rata erorilor și ca eșec la disponibilitate. Astfel eșecurile nu pot îmbunătăți artificial percentilele.
- Latența se calculează pe toate răspunsurile valide (A, B, C). Un rezultat „indisponibil” returnat în 50 ms respectă L1/L2, dar încalcă C1–C3 din 1.3; de aceea cele două familii de cerințe se verifică separat.
- Timpul de rețea până la client nu este inclus; se validează ulterior cu măsurători din browser (RUM).

### 1.2 Disponibilitate bazată pe timpul de funcționare

**Dashboard utilizabil.** Un minut este „bun” dacă, în acel minut: (a) sondele sintetice pentru toate cele 6 citiri (rulate din 2 locații) reușesc, și (b) cel puțin 99% dintre cererile reale de citire primesc răspuns non-5xx în ≤ 5 s. Dashboard-ul este utilizabil în minutele bune. Un rezultat „indisponibil” cauzat de furnizor (clasa C, marcat explicit) nu face minutul rău, deoarece Dashboard-ul funcționează; același rezultat cauzat de o defecțiune internă face minutul rău.

**Ore de tranzacționare:** 09:00–16:30 ET, de luni până vineri, în zilele de tranzacționare NYSE (fără weekenduri și sărbători). Fereastra acoperă sesiunea 09:30–16:00, încă 30 min înainte (pregătire și deschidere) și 30 min după (datele furnizorului sunt întârziate cu \~15 min, deci ultimele prețuri ale sesiunii apar până la \~16:15). Durată: 7.5 h/zi. **Restul zilei:** toate celelalte minute (nopți, weekenduri, sărbători).

**Ținte alese:**

- Ore de tranzacționare: **99.9% (trei nouari)**.
- Restul zilei: **99% (doi nouari)**.

**Bugetul de indisponibilitate pe 30 de zile** (presupunere: 21 de zile de tranzacționare în fereastra de 30 de zile; cele două intervale se măsoară cu contoare separate):

| Interval | Calcul al minutelor în 30 de zile | Țintă | Buget de indisponibilitate |
| --- | --- | --- | --- |
| Ore de tranzacționare | 21 zile × 7.5 h × 60 = 9,450 min | 99.9% | 9,450 × 0.001 = 9.45 min (9 min 27 s) |
| Restul zilei | 30 × 1,440 − 9,450 = 43,200 − 9,450 = 33,750 min | 99% | 33,750 × 0.01 = 337.5 min (5 h 37 min 30 s) |

**De ce ținte diferite.** În orele de tranzacționare, Utilizatorii iau decizii pe baza prețurilor și le reîmprospătează chiar o dată pe secundă; o pauză costă direct încrederea și utilizarea. În afara lor, piața este închisă: se citesc mai ales History și Watchlist, nu apar prețuri noi, iar echipa poate face implementări, migrări și reîncărcări de date. Dacă am folosi 99% peste tot, în orele de tranzacționare ar fi permise 9,450 × 0.01 = 94.5 min de indisponibilitate, prea mult. Dacă am folosi 99.9% peste tot, am cheltui efort pentru 33,750 × 0.001 = 33.75 min în intervalul cel mai puțin critic. Patru nouari (9,450 × 0.0001 = 0.945 min ≈ 57 s) sunt nerealiști pentru o primă versiune care depinde de un furnizor extern și are implementări.

### 1.3 Consistență

| ID | Măsură | Țintă | Condiție de operare |
| --- | --- | --- | --- |
| C1 | Vechimea unui Stock price = momentul trimiterii răspunsului − ora furnizorului a prețului | Cel puțin 99% dintre citirile Stock price au vechime ≤ 17 min (15 min întârziere furnizor + 60 s interval de sincronizare + 60 s toleranță). 100% afișează ora furnizorului și eticheta de întârziere. Între 17 și 30 min prețul este marcat vizibil „învechit”. Peste 30 min rezultatul devine „indisponibil” | Ore de tranzacționare, cu fluxul furnizorului sănătos |
| C2 | Ponderea rezultatelor „indisponibil” (clasa C) din citirile Stock price | ≤ 1% | Ore de tranzacționare, cu fluxul furnizorului sănătos |
| C3 | Preț lipsă | 0 cazuri în care un preț lipsă este afișat ca 0 sau în care un preț vechi este prezentat drept curent; prețul lipsă este afișat ca indisponibil | Oricând |
| C4 | Citire după scriere pentru Watchlist: după ce o adăugare sau o eliminare primește răspuns 2xx, următoarea citire a Watchlist-ului de către același Utilizator trebuie să reflecte modificarea | 100% dintre citirile care încep după răspunsul de succes (din orice sesiune a aceluiași Utilizator) arată modificarea; dacă garanția nu poate fi oferită, se returnează eroare, nu lista veche | Operare normală, până la ținta de capacitate |
| C5 | Izolarea Watchlist-ului | 0 citiri sau modificări ale Watchlist-ului unui alt Utilizator | Oricând |

În afara orelor de tranzacționare, C1 nu se aplică: prețul afișat este ultimul preț oficial, cu eticheta „piață închisă” și ora furnizorului.

### 1.4 Debit

| ID | Măsură | Țintă | Condiție de operare |
| --- | --- | --- | --- |
| T1 | RPS de citire cu răspuns non-5xx, susținut 30 min | 103 / 1,030 / 10,296 RPS (nivelurile 300 / 3,000 / 30,000), cu rata 5xx ≤ 0.1% și L1–L3 respectate | Mixul de la deschiderea pieței (secțiunea 3) |
| T2 | La fel, trafic stabil | 79.2 / 792 / 7,920 RPS, cu rata 5xx ≤ 0.1% și L1–L3 respectate | Mixul stabil (secțiunea 2) |

## 2. Estimări RPS în regim stabil

Formula: RPS = Utilizatori concurenți × cota participantă × acțiuni per Utilizator / secunde. Fiecare celulă arată calculul și rezultatul nerotunjit, în RPS.

| Citire (cotă × acțiuni / secunde) | 300 Utilizatori | 3,000 Utilizatori | 30,000 Utilizatori |
| --- | --- | --- | --- |
| Overview (0.70 × 1 / 30 s) | 300 × 0.70 × 1 / 30 = 210 / 30 = 7.00 | 3,000 × 0.70 × 1 / 30 = 2,100 / 30 = 70.00 | 30,000 × 0.70 × 1 / 30 = 21,000 / 30 = 700.00 |
| Filter (0.50 × 3 / 60 s) | 300 × 0.50 × 3 / 60 = 450 / 60 = 7.50 | 3,000 × 0.50 × 3 / 60 = 4,500 / 60 = 75.00 | 30,000 × 0.50 × 3 / 60 = 45,000 / 60 = 750.00 |
| Stock price (0.20 × 1 / 1 s) | 300 × 0.20 × 1 / 1 = 60.00 | 3,000 × 0.20 × 1 / 1 = 600.00 | 30,000 × 0.20 × 1 / 1 = 6,000.00 |
| History (0.20 × 1 / 300 s) | 300 × 0.20 × 1 / 300 = 60 / 300 = 0.20 | 3,000 × 0.20 × 1 / 300 = 600 / 300 = 2.00 | 30,000 × 0.20 × 1 / 300 = 6,000 / 300 = 20.00 |
| Watchlist (0.60 × 1 / 60 s) | 300 × 0.60 × 1 / 60 = 180 / 60 = 3.00 | 3,000 × 0.60 × 1 / 60 = 1,800 / 60 = 30.00 | 30,000 × 0.60 × 1 / 60 = 18,000 / 60 = 300.00 |
| Search (0.10 × 3 / 60 s) | 300 × 0.10 × 3 / 60 = 90 / 60 = 1.50 | 3,000 × 0.10 × 3 / 60 = 900 / 60 = 15.00 | 30,000 × 0.10 × 3 / 60 = 9,000 / 60 = 150.00 |
| **Total în regim stabil** | 7.00 + 7.50 + 60.00 + 0.20 + 3.00 + 1.50 = **79.20 RPS** | 70.00 + 75.00 + 600.00 + 2.00 + 30.00 + 15.00 = **792.00 RPS** | 700.00 + 750.00 + 6,000.00 + 20.00 + 300.00 + 150.00 = **7,920.00 RPS** |

Observație: Stock price reprezintă 60 / 79.2 = 6,000 / 7,920 ≈ 75.76% din traficul stabil la fiecare nivel, deoarece toate valorile cresc liniar cu numărul de Utilizatori.

## 3. Estimări RPS la deschiderea pieței

Fluxurile suplimentare sunt calculate separat și adăugate la traficul stabil din secțiunea 2, care nu este numărat a doua oară:

- Overview suplimentar: 30% dintre Utilizatori reîmprospătează o dată în 10 s: Utilizatori × 0.30 × 1 / 10.
- Watchlist suplimentar: 60% din acel grup de 30%, adică 0.30 × 0.60 = 0.18 din Utilizatori, reîmprospătează Watchlist o dată în 10 s: Utilizatori × 0.18 × 1 / 10.

| Calcul la deschiderea pieței (RPS) | 300 Utilizatori | 3,000 Utilizatori | 30,000 Utilizatori |
| --- | --- | --- | --- |
| Trafic stabil de citire | 79.20 | 792.00 | 7,920.00 |
| Flux suplimentar de reîmprospătare Overview | 300 × 0.30 × 1 / 10 = 90 / 10 = 9.00 | 3,000 × 0.30 × 1 / 10 = 900 / 10 = 90.00 | 30,000 × 0.30 × 1 / 10 = 9,000 / 10 = 900.00 |
| Flux suplimentar de reîmprospătare Watchlist | 300 × 0.30 × 0.60 × 1 / 10 = 54 / 10 = 5.40 | 3,000 × 0.30 × 0.60 × 1 / 10 = 540 / 10 = 54.00 | 30,000 × 0.30 × 0.60 × 1 / 10 = 5,400 / 10 = 540.00 |
| **Subtotal la deschiderea pieței** | 79.20 + 9.00 + 5.40 = **93.60** | 792.00 + 90.00 + 54.00 = **936.00** | 7,920.00 + 900.00 + 540.00 = **9,360.00** |
| Marjă de capacitate de 10% | 93.60 × 0.10 = 9.36 | 936.00 × 0.10 = 93.60 | 9,360.00 × 0.10 = 936.00 |
| Subtotal + marjă (nerotunjit) | 93.60 + 9.36 = 102.96 | 936.00 + 93.60 = 1,029.60 | 9,360.00 + 936.00 = 10,296.00 |
| **Țintă rotunjită în sus la deschiderea pieței** | **103 RPS** | **1,030 RPS** | **10,296 RPS** |

La deschidere, traficul crește cu 9,360 / 7,920 − 1 ≈ 18.2% față de regimul stabil (înainte de marjă). Pentru nivelul de 30,000: Overview = 700 + 900 = 1,600 RPS, Watchlist = 300 + 540 = 840 RPS.

## 4. Estimări de stocare

### 4.1 Cercetare și definiția unui Stock

**Ce poate însemna „Stock”.** Conform SEC (Investor.gov), acțiunile (stocks, numite și equities) sunt de două tipuri principale: common stock (drept de vot și dividende) și preferred stock (de regulă fără drept de vot, dar cu dividende plătite înaintea celor ale acționarilor comuni). Aceeași sursă tratează stock fund-urile ca o altă modalitate de a deține acțiuni, nu ca pe o acțiune în sine. Deci „Stock” poate însemna: common stock, preferred stock, certificate ADR ale emitenților străini, iar în limbajul curent al aplicațiilor de piață și ETF-uri sau fonduri.

**Definiția pentru acest Dashboard.** Un Stock este un instrument de capital propriu de tip common stock (acțiune ordinară), inclusiv ADR-uri ale emitenților străini, listat pe o bursă acceptată și tranzacționat în USD. Fiecare clasă de acțiuni a aceluiași emitent (de exemplu clasa A și clasa B) este un Stock separat, deoarece are simbol și preț propriu.

| Decizie | Alegere | Justificare |
| --- | --- | --- |
| Țări / burse | SUA: NYSE, Nasdaq (toate segmentele), NYSE American. Fără OTC | O singură țară, o singură monedă (USD) și un singur calendar de tranzacționare simplifică Search, Filter și orele de tranzacționare din 1.2; sursa de numărare există pentru aceste burse |
| Instrumente incluse | Common stock și ADR-uri; clase multiple tratate separat | Sunt „Stock” în sensul de bază al sursei S1 și au același model de date (preț, istoric) |
| Instrumente excluse | Preferred stock, ETF-uri, fonduri (inclusiv closed-end), warrants, drepturi, unități | Preferred are comportament hibrid (dividend fix, prioritate la lichidare), iar fondurile sunt produse diferite după S1; ar necesita câmpuri și filtre proprii. Pot fi adăugate într-o versiune ulterioară |
| Instrumente inactive / delistate | Înregistrarea de referință și istoricul rămân stocate cu status INACTIVE; sunt ascunse din Search și Filter, dar rămân vizibile în Watchlist-urile existente, cu preț „indisponibil” | Un Utilizator nu trebuie să-și piardă Watchlist-ul sau istoricul dintr-un titlu delistat; costul de stocare este mic (vezi 4.3) |
| Număr de Stocks | **5,500** (cifră de planificare) | Sursa S2 dă 5,172 de companii (3,657 interne + 1,515 străine) la sfârșitul lui 2025. Adaug \~6.3% (5,500 − 5,172 = 328): clase multiple de acțiuni, REIT-uri (excluse din S2) și instrumente inactive păstrate. Este o presupunere, de verificat cu lista oficială a burselor |

**Surse:**

- **S1 (definiția instrumentului):** U.S. Securities and Exchange Commission, Investor.gov, „Stocks – FAQs”, https://www.investor.gov/investing-basics/investment-products/stocks. Observat la 13 octombrie 2026.
- **S2 (numărul de Stocks):** Jay R. Ritter, University of Florida, „The Number of Operating Companies Listed on Nasdaq and the NYSE (including NYSE MKT), 1980–2025”, https://site.warrington.ufl.edu/ritter/files/number-of-listed-firms-on-US-exchanges.pdf. Cifra se referă la sfârșitul lui 2025 (3,657 companii interne și 1,515 străine, excluzând REIT-uri, closed-end funds și ETF-uri); observată la 13 octombrie 2026. Numără companii, nu instrumente, de aceea apare marja de 6.3%.

### 4.2 Datele sincronizate

Frecvențele sunt presupuneri; nu se estimează aici traficul către furnizor.

| Set de date | Ce se stochează | Nevoia de produs | Sincronizare (presupunere) |
| --- | --- | --- | --- |
| Date de referință Stock | instrument\_id (intern), symbol, name, exchange, currency, type, sector, industry, market\_cap\_bucket, status, updated\_at | symbol și name pentru Search; exchange, sector, industry și market\_cap\_bucket pentru Filter; currency pentru afișarea prețului; type și status pentru domeniul de aplicare | O dată pe zi, \~02:00 ET (în afara orelor de tranzacționare) |
| Cele mai recente prețuri | instrument\_id, price, prev\_close, volume, provider\_ts, synced\_at | price și provider\_ts pentru Stock price și ora furnizorului; prev\_close pentru variația din Overview și Filter; volume pentru Overview; synced\_at pentru calculul vechimii (C1). O înregistrare per Stock, suprascrisă | La 60 s, 09:00–16:30 ET |
| Price history, intraday | Bare de 5 minute: instrument\_id, ts, open, high, low, close, volume. 78 de bare/zi (6.5 h × 12). Păstrare: 30 de zile de tranzacționare | Graficele 1D, 1W, 1M din History. 5 minute dau un grafic util fără volumul bărilor de 1 minut | La fiecare 5 min, pe măsură ce barele devin disponibile (după întârzierea de 15 min) |
| Price history, zilnic | Bare zilnice: instrument\_id, date, open, high, low, close, volume. Backfill inițial 5 ani (5 × 252 = 1,260 de zile de tranzacționare); păstrare maximă 10 ani | Graficele 1Y și 5Y din History | O dată pe zi, după 16:30 ET |
| Alte date: acțiuni corporative | instrument\_id, effective, kind (split / schimbare de simbol), ratio | Ajustarea istoricului după split-uri; fără ea graficul arată salturi false | O dată pe zi |
| Alte date: calendar de tranzacționare | date, open, close, status | Definește zilele de tranzacționare, orele de tranzacționare și eticheta „piață închisă” | O dată pe an, cu verificare lunară |

**Nu se stochează** (nu sunt necesare fluxurilor din Laboratorul 1): fundamentale financiare, știri, carnetul de ordine, tranzacții individuale, bare de 1 minut, dividende. Datele Watchlist ale Utilizatorilor sunt date ale produsului, nu date de piață, și nu sunt incluse în această estimare.

### 4.3 Calculul stocării

Mărimea medie per înregistrare a fost măsurată pe înregistrări-eșantion, serializate ca JSON compact (UTF-8), valorile rotunjite în sus: Stock de referință \~270 B (eșantion de 270 B, nume de \~31 de caractere), preț recent \~160 B (eșantion 158 B), bară de 5 min 125 B, bară zilnică 115 B, acțiune corporativă \~100 B (eșantion 102 B), zi de calendar 65 B. Exemple de eșantioane:

- Referință: instrument\_id 1048213, symbol AAPL, name „Apple Inc. Common Stock Class A”, exchange XNAS, currency USD, type COMMON, sector Technology, industry Consumer Electronics, market\_cap\_bucket MEGA, status ACTIVE, updated\_at 2026-10-13T14:30:00Z.
- Preț recent: price 231.4500, prev\_close 229.8800, volume 48213900, provider\_ts 2026-10-13T14:15:00Z, synced\_at 2026-10-13T14:31:02Z.
- Bară de 5 min: ts 2026-10-13T14:15:00Z, o 231.1000, h 231.6000, l 230.9000, c 231.4500, v 1250300.

Formule: stocare brută = număr de înregistrări × octeți medii per înregistrare; număr de înregistrări de istoric = Stocks acceptate × puncte de istoric per Stock per zi × zile păstrate. Numărul de Stocks acceptate: **5,500**.

| Set de date | Decizia despre produs și perioada de păstrare | Calculul numărului de înregistrări | Octeți per înregistrare | Stocare brută |
| --- | --- | --- | --- | --- |
| Date de referință Stock | 5,500 Stocks, inclusiv inactive; păstrate permanent | 5,500 × 1 = 5,500 | 270 B | 5,500 × 270 = 1,485,000 B (1.485 MB) |
| Cele mai recente prețuri | O înregistrare per Stock, suprascrisă la fiecare sincronizare | 5,500 × 1 = 5,500 | 160 B | 5,500 × 160 = 880,000 B (0.88 MB) |
| Price history, intraday (5 min) | 78 de bare/zi, 30 de zile de tranzacționare, fereastră glisantă | 5,500 × 78 × 30 = 12,870,000 | 125 B | 12,870,000 × 125 = 1,608,750,000 B (1.609 GB) |
| Price history, zilnic | 1 bară/zi; backfill 5 ani, păstrare maximă 10 ani | 5,500 × 1 × 1,260 = 6,930,000 | 115 B | 6,930,000 × 115 = 796,950,000 B (0.797 GB) |
| Alte date: acțiuni corporative | Presupunere: 5% dintre Stocks pe an (275 evenimente/an), păstrate 10 ani | 275 × 10 = 2,750 | 100 B | 2,750 × 100 = 275,000 B (0.275 MB) |
| Alte date: calendar | 1 rând/zi, 10 ani | 365 × 10 = 3,650 | 65 B | 3,650 × 65 = 237,250 B (0.237 MB) |
| **Total (inițial)** |  |  |  | 1,485,000 + 880,000 + 1,608,750,000 + 796,950,000 + 275,000 + 237,250 = **2,408,577,250 B ≈ 2.41 GB** |

**Stocare inițială, zilnică și pe un an:**

| Moment | Calcul | Stocare brută |
| --- | --- | --- |
| Inițial (după backfill) | Total din tabelul de mai sus | 2,408,577,250 B ≈ **2.41 GB** |
| Zilnic, scris (zi de tranzacționare) | Intraday: 5,500 × 78 × 125 = 53,625,000 B; zilnic: 5,500 × 115 = 632,500 B; alte date ≈ 140 B/zi (51,225 B/an ÷ 365) | ≈ 54,257,640 B ≈ **54.3 MB/zi** (98.8% din bare intraday) |
| Zilnic, creștere netă | Barele intraday mai vechi de 30 de zile se șterg, deci creșterea netă vine din barele zilnice și din alte date | ≈ 632,500 B + \~140 B ≈ **0.63 MB/zi** |
| La un an | Barele zilnice noi: 252 × 632,500 = 159,390,000 B; alte date: 27,500 + 23,725 = 51,225 B; intraday net 0 | 2,408,577,250 + 159,441,225 = 2,568,018,475 B ≈ **2.57 GB** (creștere ≈ 159.4 MB) |

Note: (1) Scrierea intraday de 54.3 MB/zi × 252 zile = 13.5 GB/an trece prin sistem, dar nu rămâne stocată. (2) Valorile sunt date brute; indexii, replicarea și suprasarcina sistemului de stocare pot multiplica volumul de \~2–3 ori (≈ 5–8 GB, presupunere). (3) JSON-ul este o estimare conservatoare; un format binar tipizat ar fi mai mic.

## 5. Posibile blocaje

O valoare RPS mare este un motiv de investigare, nu o dovadă. Fiecare rând este o ipoteză de verificat prin măsurare.

| Calitate | Posibil blocaj | Dovezi din acest laborator | Efect posibil | Ce trebuie măsurat în continuare |
| --- | --- | --- | --- | --- |
| Latență | Calea de citire Stock price (interogare repetată pe stocarea comună) | 60 / 600 / 6,000 RPS în regim stabil, ≈ 75.76% din trafic; ținta L2 (p95 ≤ 500 ms) este cea mai strictă | Cozi la stocare și p95 peste 500 ms (L2) | Latența pe etape (edge, aplicație, stocare), așteptarea în pool-ul de conexiuni, rata de acerturi în cache, la 60 / 600 / 6,000 RPS |
| Consistență | Calea Watchlist: citire după scriere dacă citirile merg la cache sau la o replică | Watchlist: 300 RPS stabil și 840 RPS la deschidere (30,000 Utilizatori); C4 cere 100% | Citire a unei liste vechi după o modificare confirmată (C4) | Perechi scriere-apoi-citire (de exemplu 10,000) la mixul de vârf; întârzierea replicii p99; TTL și invalidare cache |
| Debit | Calcul Overview și Filter pe stocarea comună | Overview 700 RPS stabil, 1,600 RPS la deschidere; Filter 750 RPS; împreună 2,350 / 9,360 ≈ 25.1% din subtotalul la deschidere | CPU saturat, coadă, p95 > 2 s și 5xx > 0.1% (T1 nerespectat) | Milisecunde CPU și rânduri citite per cerere; RPS maxim înainte ca p95 să treacă de 2 s sau 5xx de 0.1%; acerturi în cache dacă Overview se precalculează |
| Disponibilitate | Dependența de Market Data Provider și fluxul de sincronizare | Furnizorul este sistemul extern; întârziere de \~15 min; C1 limitează vechimea la 17 min și face prețul indisponibil la 30 min; bugetul în orele de tranzacționare este de 9.45 min/30 zile | Prețuri învechite sau indisponibile chiar dacă Dashboard-ul răspunde; minute rele dacă defectul este intern | Rata de succes și latența furnizorului; distribuția vechimii prețurilor; timpul de recuperare; test în care furnizorul este blocat 20 min |

**Latență.** (1) Calea de citire Stock price: fiecare Utilizator participant reîmprospătează o dată pe secundă. (2) Dacă fiecare cerere interoghează stocarea comună, cozile cresc odată cu încărcarea și p95 poate depăși 500 ms (L2), chiar dacă celelalte citiri respectă 2 s. (3) Dovezi: 6,000 din 7,920 RPS (≈ 75.76%) în regim stabil la 30,000 de Utilizatori, plus țintă mai strictă decât restul. (4) Se confirmă dacă p95 al Stock price depășește 500 ms în testul de încărcare înainte ca totalul să atingă ținta; se respinge dacă p95 ≤ 500 ms la mixul complet de 10,296 RPS.

**Consistență.** (1) Calea stării: Watchlist-ul unui Utilizator, scris și apoi citit imediat. (2) Dacă citirile sunt servite din cache sau dintr-o replică întârziată, o citire imediat după modificare poate arăta lista veche, încălcând C4. (3) Dovezi: Watchlist are 300 RPS stabil și 540 RPS suplimentari la deschidere, deci multe citiri se petrec exact în perioada de vârf; C4 cere 100%, fără toleranță. Sincronizarea prețurilor (vechime ≤ 17 min, C1) este o a doua zonă de risc. (4) Se confirmă dacă testul scriere-apoi-citire găsește măcar o încălcare sau dacă întârzierea replicii p99 depășește timpul dintre o scriere și următoarea citire; se respinge dacă sunt 0 încălcări la încărcarea de vârf.

**Debit.** (1) Calea de citire Overview și Filter: calcule peste \~5,500 de instrumente (clasamente, filtre). (2) Overview este identic pentru toți Utilizatorii, deci recalcularea la fiecare cerere risipește capacitate; Filter are parametri variabili. Saturarea CPU duce la cozi, p95 > 2 s și 5xx > 0.1%, deci T1 nu este respectat. (3) Dovezi: Overview ajunge la 1,600 RPS la deschidere (30,000 Utilizatori), iar Overview + Filter la 2,350 RPS, ≈ 25.1% din subtotal. (4) Se confirmă dacă costul per cerere înmulțit cu RPS depășește capacitatea la 10,296 RPS; se respinge dacă sistemul susține 30 min fără să încalce T1.

**Disponibilitate.** (1) Dependența externă: Market Data Provider și fluxul de sincronizare. (2) Dacă fluxul se oprește, vechimea prețurilor crește: după \~1–2 min depășește 17 min (C1), după \~14–15 min depășește 30 min și prețurile devin indisponibile. Dacă citirile ar apela furnizorul direct, o defecțiune a furnizorului ar deveni o defecțiune a Dashboard-ului. (3) Dovezi: întârzierea de \~15 min a furnizorului, pragurile de 17 și 30 min din C1 și bugetul de 9.45 min din 1.2. (4) Se confirmă dacă în testul cu furnizorul blocat 20 min citirile încep să returneze 5xx sau etichetele nu trec la „învechit” (la 17 min) și „indisponibil” (la 30 min); se respinge dacă citirile rămân 2xx, etichetele sunt corecte, iar minutele sunt clasificate corect.

## Listă de verificare

- ✔ Am scris cerințe măsurabile pentru toate calitățile de bază (latență, disponibilitate, consistență, debit).
- ✔ Am ales și am justificat ținte ale timpului de funcționare pentru ambele părți ale zilei (99.9% și 99%).
- ✔ Am arătat calculele RPS în regim stabil și la deschiderea pieței pentru toate cele trei niveluri.
- ✔ Am cercetat și am definit domeniul de aplicare pentru Stocks acceptate (5,500 Stocks, SUA).
- ✔ Am estimat stocarea brută inițială (2.41 GB), zilnică (54.3 MB scrise, 0.63 MB creștere netă) și pentru un an (2.57 GB).
- ✔ Am precizat presupunerile, unitățile, intervalele și rotunjirea finală.
- ✔ Am analizat un posibil blocaj pentru fiecare calitate de bază.
