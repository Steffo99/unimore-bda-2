[ **Stefano Pigozzi** | Traccia #1 | Tema Big Data | Big Data Analytics | A.A. 2022/2023 | Unimore ]

# Redis per applicazioni scalabili, efficienti e veloci

> ### Approfondimento NoSQL
> 
> L’attività consiste nell’approfondire uno degli argomenti visti nelle lezioni relative ai modelli NOSQL, al CAP theorem e alle architetture per big data.
>
> Esempi di possibili approfondimenti sono:
> - **un sistema che implementa un modello NOSQL non sperimentato a lezione**
> - \[...\]
>
> Possibili fonti informative, oltre a Google e alla manualistica ufficiale, sono le digital library scientifiche come [Google Scholar](https://scholar.google.it/) e [Mendeley](https://www.mendeley.com).
>
> Al termine dell’attività di approfondimento, occorre produrre una relazione con le seguenti caratteristiche:
> - la relazione deve avere un titolo e contenere un abstract (riassunto del documento), una sezione di conclusioni e un elenco di riferimenti bibliografici;
> - è gradito includere una piccola sperimentazione pratica (es. un breve codice di prova per una piattaforma studiata, brevi frammenti di codice a confronto delle caratteristiche di diversi sistemi, ecc.), riassumendo nella parte finale della relazione cosa è stato provato e, se applicabile, i risultati ottenuti;
> - la relazione deve essere lunga almeno 2000 parole e non più di 3000 parole.

## Sinossi

In questa relazione si introduce il database key-value [Redis](https://redis.io/), ne si descrivono le funzionalità e i casi d'uso più comuni, e lo si utilizza per sviluppare [un'applicazione web scalabile e performante](https://github.com/Steffo99/distributed-arcade/), che sarà poi sottoposta a stress testing.

## Redis

**Redis** (**Re**mote **Di**ctionary **S**erver) è un database NoSQL creato e pubblicato originariamente da [Antirez](http://invece.org/), imprenditore italiano che necessitava di un database real-time e scalabile per sviluppare un [analizzatore di web log](https://github.com/antirez/lloogg), e in seguito sviluppato e mantenuto da [Redis Ltd](https://redis.com/).

È un database NoSQL con le seguenti caratteristiche:

* si basa sul paradigma **key-value**
    * i dati sono archiviati nel database associati a una determinata stringa, e possono essere richiamati attraverso di essa
    * convenzione vuole che le chiavi siano in `lower:colon:case`, come ad esempio `player:1:score`
* è interamente **in-memory**
    * tutti i dati sono salvati nella RAM per massimizzare la velocità di archiviazione e recupero
    * opzionalmente, è possibile configurare un salvataggio periodico su disco del database
* cerca di avere un **impatto minimo sulle risorse** dell'elaboratore
    * sia in termini di CPU-time, sia in termini di memoria utilizzata
    * l'overhead del database senza alcun dato dentro è di soli 3 MB
* utilizza brevi **comandi** per alterare i dati nel database
    * molto simili a comandi di una shell, facili da integrare in linguaggi di programmazione
    * tutti i comandi sono individualmente atomici, e possono essere opzionalmente combinati in transazioni atomiche
* è **loosely-typed**
    * sia chiavi sia valori sono sempre archiviati nel database come stringhe
    * sono i comandi utilizzati a determinare come i valori vengono interpretati, se come stringhe o come strutture dati
* è **facile da scalare orizzontalmente**
    * può utilizzare il paradigma master-replica, con meccanismi di sincronizzazione atomici, parziali o totali in base allo stato di connessione delle repliche
    * può essere eseguito come cluster, in cui ogni istanza gestisce un sottoinsieme delle chiavi del database complessivo

### Tipi di dato

Redis supporta vari tipi di dati, ciascuno adatto a diversi casi d'uso.

Come menzionato in precedenza, si disambigua tra essi attraverso i comandi utilizzati per accedervi: in particolare, tutti i comandi relativi a un determinato tipo di struttura dati hanno uno specifico prefisso, come ad esempio `Z` per i sorted set.

#### Stringhe

Il tipo *stringa* in Redis in realtà si riferisce a qualsiasi tipo di dato non strutturato, inclusi numeri o byte di file binari.

I comandi che si riferiscono alle stringhe non usano alcun prefisso, e sono `O(1)`:

* `GET {key}`
    * recupera il valore alla chiave specificata
* `SET {key} {value}`
    * imposta il valore della chiave specificata
* `SET {key} {value} NX`
    * imposta il valore della chiave specificata solo se essa non esiste già
* `SET {key} {value} EX {seconds}`
    * imposta il valore della chiave specificata per il numero di secondi specificato
* `INCR {key}`
    * se il valore alla chiave è un numero, lo incrementa di 1
    * se non esiste, il numero viene prima impostato a 0
* `INCRBY {key} {number}`
    * se il valore alla chiave è un numero, lo incrementa del numero specificato
    * se non esiste, il numero viene prima impostato a 0

Sono utili per archiviare e recuperare velocemente attributi di un'entità, implementare meccanismi di sincronizzazione (lock, semafori), realizzare controlli di accesso effimeri (rate limiting), o effettuare conteggi di particolari eventi.

#### List

Il tipo *list* in Redis rappresenta una serie ordinata di valori stringa.

I principali comandi delle liste usano i prefissi `L` o `R`, e sono `O(1)`:

* `LPUSH {key} {value}`
    * aggiunge un valore all'inizio della lista specificata
* `RPUSH {key} {value}`
    * aggiunge un valore alla fine della lista specificata
* `LPOP {key}`
    * recupera ed elimina il valore all'inizio della lista specificata
* `RPOP {key}`
    * recupera ed elimina il valore all'inizio della lista specificata

Particolari comandi relativi alle liste permettono al client Redis di bloccare la chiamata fino a che non si verifica una determinata condizione, come ad esempio:

* `BLPOP {key}`
    * se esiste, recupera ed elimina il valore all'inizio della lista specifiata
    * se non esiste, attende che venga inserito un valore nella lista prima di restituirlo

I comandi relativi alle liste sono tipicamente utilizzati per realizzare pile (stacks) e code (queues) di eventi, prodotti da alcuni processi e consumati da altri.

#### Set

Il tipo *set* in Redis rappresenta un insieme di valori stringa unici non ordinato.

I principali comandi dei set usano il prefisso `S`, e sono `O(1)`:

* `SADD {key} {value}`
    * aggiunge un valore all'insieme specificato
* `SISMEMBER {key} {value}`
    * verifica se un valore appartiene all'insieme specificato
* `SCARD {key}`
    * conta il numero di elementi nell'insieme specificato

Alcuni comandi permettono di effettuare operazioni tra due insiemi in tempo `O(N)`:

* `SINTER {key1} {key2}`
    * effettua l'intersezione tra due insiemi, e restituisce i valori dell'insieme risultante
* `SUNION {key1} {key2}`
    * effettua l'unione tra due insiemi, e restituisce i valori dell'insieme risultante

Sono usati nei casi in cui l'**unicità** degli elementi è un fattore chiave dell'operazione che si desidera effettuare, come ad esempio per raccogliere gli identificativi di tutti gli utenti che hanno letto un post su un forum.

#### Sorted sets

Il tipo *sorted set* in Redis rappresenta un insieme di valori stringa unici ordinato in base a un punteggio associato a ciascun valore.

I principali comandi dei sorted set usano il prefisso `Z`, e sono `O(log(N))`:

* `ZADD {key} {score} {value}`
    * aggiunge un valore all'insieme specificato con un dato punteggio
* `ZRANGE {key} {start} {stop}`
    * recupera i valori presenti nell'insieme specificato tra i dati indici (inclusi)
* `ZSCORE {key} {value}`
    * recupera il punteggio nell'insieme specificato di un dato valore
* `ZRANK {key} {value}`
    * restituisce la posizione del valore dato nell'insieme specificato

Sono usati in casi in cui si necessita di una struttura **sempre ordinata** di elementi unici, come nell'applicazione allegata a questa relazione, in cui sono stati usati per realizzare delle classifiche.

#### Altre strutture dati

Redis supporta altre strutture dati meno frequentemente usate:

* gli *hashes*, collezioni chiave-valore innestate in una singola chiave Redis, con prefisso `H`
* gli *stream*, registri in cui è solo possibile aggiungere valori, con prefisso `X`
* gli indici *geospatial*, in grado di processare coordinate cartesiane o geografiche, con prefisso `GEO`
* gli *hyperloglog*, strutture dati probabilistiche in grado di stimare velocemente la cardinalità di insiemi molto grandi, con prefisso `PF`
* le *bitmap*, array di bit che possono essere modificati o interrogati individualmente, con prefisso o suffisso `BIT`
* i *bitfields*, interi signed e unsigned di dimensione variabile, attraverso il comando `BITFIELD`

## Applicazione

### Rust

### Interfaccia con Redis

### Comandi eseguiti

## Testing e benchmarking

### Richieste HTTP di esempio

### Stress testing con [`siege`](https://www.joedog.org/siege-home/)

### Scalabilità orizzontale

## Conclusioni

## Bibliografia

- [**Documentazione** di Redis](https://redis.io/docs/)
- [**The SET command is a strange beast**, articolo dal blog di Redis](https://redis.com/blog/set-command-strange-beast/)
- [**Redis** su Wikipedia](https://en.wikipedia.org/w/index.php?title=Redis&oldid=1115152231)