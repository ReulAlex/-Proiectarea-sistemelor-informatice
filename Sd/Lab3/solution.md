# Laboratorul 3: Desenează limita Dashboard-ului

**Student:** Reul Alexei · **Grupa:** Infa-241 · **Data:** 13 octombrie 2026

## 0. Puncte de pornire și presupuneri generale

- Aceleași ținte de încărcare ca în Laboratorul 2: 103 / 1,030 / 10,296 RPS la deschiderea pieței pentru 300 / 3,000 / 30,000 de Utilizatori concurenți, cu p95 ≤ 2,000 ms (Stock price: p95 ≤ 500 ms).
- Aceleași reguli de prospețime (C1 din Laboratorul 2): un preț are vechimea = momentul răspunsului minus ora furnizorului; până la 17 min este acceptat, între 17 și 30 min este marcat „învechit”, peste 30 min sau lipsă este „indisponibil” (niciodată 0).
- Citirea unui Watchlist după o modificare confirmată trebuie să arate modificarea (C4 din Laboratorul 2); Watchlist-ul unui Utilizator nu poate fi citit sau modificat de altcineva.
- Utilizatorul este deja autentificat prin sesiune (token semnat, verificat de Dashboard Application). Înregistrarea și autentificarea inițială sunt în afara acestui laborator.
- Citirile Utilizatorului nu creează cereri către furnizor: Dashboard-ul sincronizează datele în propria stocare, iar citirile se fac din ea (la fel ca în secțiunea 4 din Laboratorul 2).
- Numele elementelor (Dashboard Application, Market Data Collector, Dashboard Database, Market Data Provider) rămân identice în toate cele șase diagrame. Diagramele au fost verificate sintactic cu parserul Mermaid.

## 1. Diagrama de context

```mermaid
C4Context
    title Diagramă de context: Dashboard
    Person(user, "Utilizator autentificat", "Răsfoiește și filtrează Stocks, citește prețuri și istoric, își gestionează Watchlist-ul privat")
    System(dashboard, "Dashboard", "Afișează date de piață întârziate și păstrează Watchlist-uri private")
    System_Ext(provider, "Market Data Provider", "Furnizează date de piață întârziate cu ~15 min. Poate returna date invalide, aplica limite de rată sau fi indisponibil")
    Rel(user, dashboard, "Răsfoiește, filtrează, citește prețuri și istoric, modifică Watchlist", "HTTPS")
    Rel(dashboard, provider, "Cere date de piață întârziate", "HTTPS, cheie API")
```

**Ce arată:** limita sistemului (Dashboard, tratat ca o singură cutie), un actor uman (Utilizatorul autentificat) și un sistem extern (Market Data Provider), cu relații etichetate. Nu apar părți interne. Utilizatorul nu comunică niciodată direct cu furnizorul.

## 2. Diagrama de containere

```mermaid
C4Container
    title Diagramă de containere: Dashboard
    Person(user, "Utilizator autentificat", "Folosește Dashboard-ul din browser")
    System_Boundary(dashboard, "Dashboard") {
        Container(app, "Dashboard Application", "Serviciu web", "Servește interfața și API-ul. Verifică identitatea și intrarea și răspunde la citiri și la modificări de Watchlist")
        Container(collector, "Market Data Collector", "Job programat", "Cere date de la furnizor, le validează și scrie doar datele acceptate")
        ContainerDb(db, "Dashboard Database", "Bază de date relațională", "Stochează Stocks, cele mai recente prețuri, istoricul, calendarul și Watchlist-urile")
    }
    System_Ext(provider, "Market Data Provider", "Furnizează date de piață întârziate")
    Rel(user, app, "Răsfoiește, filtrează, citește prețuri și istoric, modifică Watchlist", "HTTPS")
    Rel(app, db, "Citește date de piață, citește și scrie Watchlist-uri", "SQL, TLS")
    Rel(collector, db, "Scrie datele acceptate într-o tranzacție", "SQL, TLS")
    Rel(collector, provider, "Cere date întârziate", "HTTPS, cheie API")
```

| Container | Responsabilitate unică | Comunică direct cu |
| --- | --- | --- |
| Dashboard Application | Răspunde Utilizatorului: verifică identitatea și intrarea, apoi citește date sau modifică Watchlist-ul | Utilizator (HTTPS), Dashboard Database (SQL) |
| Market Data Collector | Aduce date de la furnizor și le face disponibile doar după validare | Market Data Provider (HTTPS), Dashboard Database (SQL) |
| Dashboard Database | Păstrează datele de piață acceptate și Watchlist-urile | Dashboard Application, Market Data Collector |

**De ce acest set minim:**

- **Market Data Collector separat de Dashboard Application.** Furnizorul poate fi lent, poate aplica limite de rată sau poate cădea (blocaj de disponibilitate din Laboratorul 2). Dacă apelul către furnizor ar fi în calea citirii, o problemă a furnizorului ar deveni o problemă a Dashboard-ului. Separarea păstrează și cheia API într-un singur loc, pe server, fără acces din browser sau din Dashboard Application.
- **O singură Dashboard Database.** Citirile Watchlist merg la stocarea primară, deci citirea după scriere (C4) este simplă de garantat. Volumul brut estimat este 2.41 GB inițial (Laboratorul 2), ceea ce nu cere împărțirea datelor în mai multe stocări.
- **Fără cache separat, coadă sau serviciu de autentificare.** Căile cu RPS mare (Stock price 6,000 RPS, Overview 1,600 RPS la deschidere pentru 30,000 de Utilizatori) sunt doar ipoteze de blocaj. Un cache se adaugă numai dacă testul de încărcare dovedește că este necesar. Sesiunile sunt tokenuri semnate, verificate în aplicație.
- **Fără container pentru interfață.** Dashboard Application servește și interfața web.

| Cerință din Laboratorul 2 | Cum este acoperită de design |
| --- | --- |
| Latență (L1–L3) | Toate citirile merg într-un singur salt către Dashboard Database, fără apel către furnizor |
| Disponibilitate | Căderea furnizorului afectează doar Market Data Collector; citirile continuă cu date existente, marcate „învechit” sau „indisponibil” |
| Consistență (C1, C3) | Collector scrie doar date validate; Dashboard Application calculează vechimea la fiecare citire |
| Consistență (C4, C5) | Watchlist se scrie și se citește din stocarea primară, cu identitatea luată din sesiune |
| Debit (T1, T2) | Dashboard Application poate fi rulat în mai multe instanțe în fața aceleiași stocări; limita reală se măsoară |

## 3. Diagrama de componente (Dashboard Application)

```mermaid
C4Component
    title Diagramă de componente: Dashboard Application
    Person(user, "Utilizator autentificat", "Folosește Dashboard-ul din browser")
    Container_Boundary(app, "Dashboard Application") {
        Component(router, "Request Router", "Modul", "Primește cererea HTTP, apelează verificările în ordine și întoarce răspunsul")
        Component(auth, "Authentication Guard", "Modul", "Verifică identitatea și stabilește Utilizatorul curent")
        Component(validator, "Input Validator", "Modul", "Verifică și normalizează parametrii cererii")
        Component(browse, "Market Browse Service", "Modul", "Construiește Overview, Filter și Search")
        Component(price, "Price and History Service", "Modul", "Întoarce prețul cu starea de prospețime și istoricul")
        Component(watch, "Watchlist Service", "Modul", "Verifică permisiunea și gestionează Watchlist-ul Utilizatorului")
        Component(dbaccess, "Database Access", "Modul", "Singurul punct de acces la stocare, cu limite de timp și erori uniforme")
    }
    ContainerDb(db, "Dashboard Database", "Bază de date relațională", "Date de piață și Watchlist-uri")
    Rel(user, router, "Trimite cereri și primește răspunsuri", "HTTPS")
    Rel(router, auth, "Cere verificarea identității")
    Rel(router, validator, "Cere validarea intrării")
    Rel(router, browse, "Deleagă Overview, Filter, Search")
    Rel(router, price, "Deleagă Stock price și History")
    Rel(router, watch, "Deleagă Watchlist")
    Rel(browse, dbaccess, "Cere Stocks și prețuri")
    Rel(price, dbaccess, "Cere preț, istoric, calendar")
    Rel(watch, dbaccess, "Cere citire și scriere Watchlist")
    Rel(dbaccess, db, "Interogări și tranzacții", "SQL, TLS")
```

| Componentă | Responsabilitate | Elemente care comunică direct cu ea |
| --- | --- | --- |
| Request Router | Punctul de intrare HTTP: ordinea verificărilor (identitate, intrare, serviciu) și maparea rezultatului în cod HTTP | Utilizator (HTTPS), Authentication Guard, Input Validator, cele trei servicii |
| Authentication Guard | Verifică sesiunea și furnizează user\_id; nu acceptă user\_id din cerere | Request Router |
| Input Validator | Respinge simboluri, filtre și parametri invalizi înainte de orice acces la stocare | Request Router |
| Market Browse Service | Overview, Filter și Search peste Stocks și cele mai recente prețuri | Request Router, Database Access |
| Price and History Service | Stock price și History; clasifică vechimea (până la 17 min, 17–30 min, peste 30 min sau lipsă, piață închisă) | Request Router, Database Access |
| Watchlist Service | Verifică dacă Watchlist-ul aparține Utilizatorului curent și aplică adăugarea, eliminarea și citirea | Request Router, Database Access |
| Database Access | Conexiuni, limite de timp (1 s) și transformarea erorilor de stocare într-un rezultat uniform | Cele trei servicii, Dashboard Database |

**Ce nu este arătat:** doar Dashboard Application este deschis; Dashboard Database apare ca element extern. Market Data Collector și Market Data Provider nu comunică direct cu nicio componentă: singura legătură dintre ele și Dashboard Application sunt datele din Dashboard Database. Nu sunt arătate clase, funcții sau aplicații desfășurate separat.

## 4. Diagrame de secvență

### A. Sincronizarea datelor de piață

```mermaid
sequenceDiagram
    autonumber
    participant Collector as Market Data Collector
    participant Provider as Market Data Provider
    participant DB as Dashboard Database
    Note over Collector: Pornire: planificator intern, la 60 s în orele de tranzacționare
    Collector->>Provider: Cere prețurile întârziate (HTTPS, cheia API de pe server)
    alt Timeout sau eroare 5xx
        Provider--xCollector: fără răspuns valid la timp
        Collector->>Collector: Reîncearcă cu backoff (max 2 încercări), apoi renunță la ciclu
        Note over Collector,DB: Nu se scrie nimic. Datele acceptate anterior rămân neschimbate
    else Limită de rată (429)
        Provider-->>Collector: 429 cu Retry-After
        Collector->>Collector: Amână până la Retry-After și sare peste ciclul curent
        Note over Collector,DB: Nu se scrie nimic. Datele acceptate anterior rămân neschimbate
    else Răspuns 200 cu lot de date
        Provider-->>Collector: Lot de prețuri cu provider_ts
        Collector->>Collector: Validează fiecare înregistrare (schemă, simbol cunoscut, preț numeric mai mare ca 0, provider_ts valid și mai nou decât cel stocat)
        alt Mai mult de 10% dintre înregistrări sunt invalide
            Collector->>Collector: Respinge tot lotul și înregistrează eroarea
            Note over Collector,DB: Nu se scrie nimic. Datele acceptate anterior rămân neschimbate
        else Cel mult 10% invalide
            Collector->>Collector: Elimină înregistrările invalide și le înregistrează în jurnal
            Collector->>DB: Scrie înregistrările acceptate într-o singură tranzacție (upsert)
            alt Tranzacția reușește
                DB-->>Collector: COMMIT confirmat
                Note over Collector,DB: De la COMMIT, citirile ulterioare văd prețurile noi
            else Stocarea eșuează
                DB--xCollector: eroare, ROLLBACK
                Note over Collector,DB: Datele acceptate anterior rămân. Reîncercare la următorul ciclu
            end
        end
    end
```

**Presupuneri de pornire:**

- Sincronizarea este pornită de planificatorul intern al lui Market Data Collector, la fiecare 60 s între 09:00 și 16:30 ET (prețuri recente); datele de referință și istoricul rulează după programul din Laboratorul 2. Nu o pornește nicio cerere a Utilizatorului.
- Cheia API a furnizorului este în configurația de pe server, accesibilă doar lui Market Data Collector; nu ajunge în browser, în răspunsuri sau în jurnale.
- Pragul de 10% (lot suspect) și cele 2 reîncercări sunt presupuneri de proiectare, de ajustat după măsurători.

**Ce se întâmplă cu datele acceptate anterior când sincronizarea eșuează:** rămân neschimbate (nu se șterg și nu se suprascriu). Datele devin vizibile pentru citiri doar după COMMIT, deci nu apar niciodată date parțiale. Cât timp sincronizarea eșuează, vechimea prețurilor crește și Diagrama B le marchează „învechit” după 17 min și „indisponibil” după 30 min.

### B. Citirea unui Stock price

```mermaid
sequenceDiagram
    autonumber
    actor User as Utilizator
    participant App as Dashboard Application
    participant DB as Dashboard Database
    User->>App: GET preț pentru un Stock (cu sesiune)
    App->>App: Verifică identitatea
    alt Neautentificat
        App-->>User: 401 Neautentificat
    else Autentificat
        App->>App: Validează simbolul (1 până la 10 caractere permise)
        alt Intrare invalidă
            App-->>User: 400 Simbol invalid
        else Intrare validă
            App->>DB: Citește Stock, cel mai recent preț și starea pieței
            alt Stocarea eșuează sau depășește 1 s
                DB--xApp: eroare sau timeout
                App-->>User: 503 Datele nu pot fi citite acum (nu se afișează 0)
            else Stock necunoscut
                DB-->>App: nicio înregistrare
                App-->>User: 404 Stock inexistent
            else Stock găsit
                DB-->>App: price, provider_ts, starea pieței
                App->>App: Calculează vechimea = acum minus provider_ts
                alt Preț lipsă
                    App-->>User: 200 stare INDISPONIBIL, fără valoare numerică
                else Piață închisă
                    App-->>User: 200 ultimul preț oficial, provider_ts, eticheta piață închisă
                else Vechime peste 30 min
                    App-->>User: 200 stare INDISPONIBIL, cu ultima oră a furnizorului
                else Vechime între 17 și 30 min
                    App-->>User: 200 preț, provider_ts, eticheta ÎNVECHIT
                else Vechime până la 17 min
                    App-->>User: 200 preț, provider_ts, eticheta întârziat cu ~15 min
                end
            end
        end
    end
```

**Presupuneri de pornire:**

- **O citire a Utilizatorului nu cere nicio dată nouă furnizorului.** Prețul vine din Dashboard Database, scris anterior de Market Data Collector (Diagrama A). Market Data Provider nu apare în această diagramă.
- Limitele de vechime sunt cele din Laboratorul 2: 17 min acceptat, 30 min indisponibil; în afara orelor de tranzacționare se aplică ultimul preț oficial cu eticheta „piață închisă”.
- Limita de timp la stocare este de 1 s (presupunere); un 503 este rezultat de eroare și se numără la rata erorilor și la disponibilitate. Un rezultat „indisponibil” (200, marcat) este alt tip de rezultat și se măsoară separat (C2).

### C. Modificarea unui Watchlist privat

```mermaid
sequenceDiagram
    autonumber
    actor User as Utilizator
    participant App as Dashboard Application
    participant DB as Dashboard Database
    User->>App: POST adaugă un Stock în Watchlist (cu sesiune)
    App->>App: Verifică identitatea și stabilește user_id din sesiune
    alt Neautentificat
        App-->>User: 401 Neautentificat
    else Autentificat
        App->>App: Validează intrarea (simbol, limita de elemente)
        alt Intrare invalidă
            App-->>User: 400 Cerere invalidă
        else Intrare validă
            App->>App: Verifică permisiunea: Watchlist-ul țintit aparține user_id
            alt Watchlist-ul aparține altui Utilizator
                App-->>User: 403 Interzis (nu se citește și nu se scrie nimic)
            else Permis
                App->>DB: Scrie elementul în Watchlist-ul Utilizatorului (tranzacție)
                alt Stocarea eșuează sau rezultatul este nesigur
                    DB--xApp: eroare sau timeout
                    App-->>User: 503 Modificarea nu a fost confirmată (se poate reîncerca)
                else Stock inexistent
                    DB-->>App: încălcare cheie externă
                    App-->>User: 422 Stock inexistent
                else COMMIT reușit
                    DB-->>App: COMMIT confirmat
                    Note over App,DB: Punctul de succes: doar după COMMIT confirmat
                    App-->>User: 200 Modificarea a fost aplicată
                    User->>App: GET Watchlist (următoarea citire, același Utilizator)
                    App->>DB: Citește Watchlist-ul din stocarea primară, fără cache sau replică
                    DB-->>App: lista cu modificarea
                    App-->>User: 200 lista include Stock-ul adăugat
                end
            end
        end
    end
```

**Presupuneri de pornire:**

- user\_id vine din sesiune, nu din corpul sau adresa cererii; Watchlist-ul țintit este cel al lui user\_id. Dacă cererea indică Watchlist-ul altcuiva, rezultatul este 403 și nu se atinge stocarea (se poate returna și 404 pentru a nu dezvălui existența, am ales 403 pentru claritate).
- Adăugarea este idempotentă (același Stock adăugat de două ori produce aceeași listă), deci o reîncercare după un 503 este sigură. Limita de elemente (de exemplu 100) este o presupunere.
- **Succesul se raportează doar după COMMIT confirmat.** Dacă rezultatul scrierii este nesigur (timeout), Utilizatorul primește 503, nu succes.
- **Următoarea citire a Watchlist-ului de către același Utilizator trebuie să arate modificarea (100%, C4)**: citirea se face din stocarea primară, fără cache sau replică întârziată, din orice sesiune a aceluiași Utilizator.


