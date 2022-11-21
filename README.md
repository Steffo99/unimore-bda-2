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

Per via delle sue caratteristiche, Redis è frequentemente utilizzato come database "secondario": per comunicazione inter-processo o inter-servizio, per caching di dati costosi da computare, o per carichi di lavoro che necessitano di numerosissimi accessi veloci a risorse.

### Tipi di dato

Redis supporta vari tipi di dati, ciascuno adatto a diversi casi d'uso.

Come menzionato in precedenza, si disambigua tra essi attraverso i comandi utilizzati per accedervi: in particolare, tutti i comandi relativi a un determinato tipo di struttura dati hanno uno specifico prefisso, come ad esempio `Z` per i sorted set.

#### Strings

Il tipo [*string*](https://redis.io/docs/data-types/strings/) in Redis in realtà si riferisce a qualsiasi tipo di dato non strutturato, inclusi numeri o byte di file binari.

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

#### Lists

Il tipo [*list*](https://redis.io/docs/data-types/lists/) in Redis rappresenta una serie ordinata di valori stringa.

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

#### Sets

Il tipo [*set*](https://redis.io/docs/data-types/sets/) in Redis rappresenta un insieme di valori stringa unici non ordinato.

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

Il tipo [*sorted set*](https://redis.io/docs/data-types/sorted-sets/) in Redis rappresenta un insieme di valori stringa unici ordinato in base a un punteggio associato a ciascun valore.

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

* gli [*hashes*](https://redis.io/docs/data-types/hashes/), collezioni chiave-valore innestate in una singola chiave Redis, con prefisso `H`
* gli [*stream*](https://redis.io/docs/data-types/streams/), registri in cui è solo possibile aggiungere valori, con prefisso `X`
* gli indici [*geospatial*](https://redis.io/docs/data-types/geospatial/), in grado di processare coordinate cartesiane o geografiche, con prefisso `GEO`
* gli [*hyperloglog*](https://redis.io/docs/data-types/hyperloglogs/), strutture dati probabilistiche in grado di stimare velocemente la cardinalità di insiemi molto grandi, con prefisso `PF`
* le [*bitmap*](https://redis.io/docs/data-types/bitmaps/), array di bit che possono essere modificati o interrogati individualmente, con prefisso o suffisso `BIT`
* i [*bitfields*](https://redis.io/docs/data-types/bitfields/), interi signed e unsigned di dimensione variabile, attraverso il comando `BITFIELD`

## Disponibilità e scalabilità

Redis offre numerose funzionalità per fare in modo che il database rimanga sempre disponibile e possa scalare orizzontalmente per supportare numeri di accessi sempre crescenti.

### Replicazione

È possibile creare illimitate [*repliche*](https://redis.io/docs/management/replication/) in sola lettura di qualsiasi database Redis.

Le repliche si connettono al database originale, richiedono ad esso una **sincronizzazione totale**, e poi mantengono la connessione aperta per ricevere dall'originale il **flusso di comandi** necessario a mantenere i dati sincronizzati.

Nel caso una replica perdesse la connessione all'originale, essa continuerebbe a funzionare indipendentemente, e al ripristino della connessione essa può richiedere una **sincronizzazione parziale**, ricevendo i cambiamenti avvenuti al database dal momento della disconnessione.

Questo permette a Redis grandi benefici di scalabilità, rendendo possibile ad esempio:

* distribuire il carico di lavoro geograficamente, minimizzando la latenza tra utente e servizio
* delegare query di letture computazionalmente costose a una replica, conservando potenza computazionale dell'originale per gestire più scritture
* eseguire una replica mentre il software del database originale viene aggiornato, mantenendo così il database accessibile tutto il tempo

La sincronizzazione tra le repliche è effettuata asincronamente: ciò significa che la **consistenza non è garantita**, ma che, con il passare del tempo, il database convergerà alla consistenza.

#### Sentinel

A questo sistema può essere aggiunto il servizio [*Redis Sentinel*](https://redis.io/docs/management/sentinel/), un daemon che:

* monitora il database originale e le sue repliche
* notifica l'amministratore automaticamente nel caso di downtime di un'istanza di Redis
* promuove una delle repliche a "principale" nel caso il database originale smettesse di essere accessibile
* comunica autorevolmente alle altre repliche l'indirizzo del nuovo database principale nel caso di un cambio, prevenendo il [problema dei due generali](https://en.wikipedia.org/wiki/Two_Generals'_Problem)

### Clustering

È possibile configurare più istanze di Redis perchè funzionino come un unico database utilizzando [*Redis Cluster*](https://redis.io/docs/management/scaling/).

Attraverso di esso, le istanze si spartiscono l'autorità su determinate chiavi, effettuando l'hash su di esse per determinare quale istanza è la "originale" per la data chiave.

In particolare, l'hash viene effettuato sulla parte di chiave contenuta tra parentesi graffe, permettendo a una data istanza di avere l'autorità su tutte le chiavi relative a una certa entità.

Ad esempio, tutte queste chiavi saranno assegnate alla stessa istanza:

```
board:{first}:scores
board:{first}:name
board:{first}:creation_date
user:{first}:name
```

Queste chiavi potrebbero invece venire assegnate a un'altra istanza:

```
board:{second}:scores
user:{third}:name
second
```

Come per le repliche, essendo la sincronizzazione tra i cluster effettuata asincronamente, la **consistenza non è garantita**.

## Applicazione

Per testare le caratteristiche di Redis, ho sviluppato una piccola applicazione che lo utilizza.

L'applicazione, [**Distributed Arcade**](https://github.com/Steffo99/distributed-arcade), è un servizio di gestione classifiche in grado di processare l'immissione di numerosissimi punteggi senza avere grossi costi di performance sulle macchine su cui è ospitato.

![Diagramma di funzionamento di Distributed Arcade](media/diagram-app.png)

L'applicazione è intesa per essere utilizzata da videogiochi disponibili su svariate piattaforme: browser web, desktop computer, smartphone e tablet...  
A tale scopo, si è scelto di realizzarla come una web API attraverso la quale essi possano interfacciarsi in modo controllato con il database Redis.

Nell'applicazione, amministratori autorizzati possono creare ***board*** ("tabelloni", da *leaderboard*, "classifica"), ricevendo un ***token*** (una stringa di testo url-safe generata in modo crittograficamente sicuro) per l'immissione di punteggi in quello specifico board.
Il token può poi essere inserito all'interno delle applicazioni che si desidera autorizzare a inserire ***score*** ("punteggi") nel board, in modo che esse possano periodicamente inviare i risultati dei giocatori.

In qualsiasi momento, le applicazioni possono interrogare l'applicazione per ricevere lo stato aggiornato di un board, o la posizione in classifica di un player specifico, in modo da avere dati da visualizzare all'utente.

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