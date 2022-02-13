# Access Monitor
Access Monitor è un sistema software il cui obiettivo è fornire un servizio di security e controllo degli
accessi per luoghi chiusi al pubblico (in cui si consente l'accesso solo a del personale autorizato).
Una videocamera IoT rileva volti umani durate la registrazione e li inoltra al sistema di autenticazione
che registra un accesso autorizzato o non autorizzato. Gli amministratori della security possono
avere accesso alla cronologia di tutti gli accessi autorizzati e non.


## Architettura del sistema
<img src="https://github.com/Access-Monitor/.github/blob/main/res/architecture.jpg" alt="architettura del sistema">

L'immagine di cui sopra ci mostra l'architettura del sistema Access Monitor, in cui vengono riassunte tutte le interazioni
tra le componenti software. 

La lettura del diagramma può iniziare dalla **IoT cam**: una videocamera smart che è in grado di rilevare volti umani durante la registrazione
e caricare l'immagine del rilevamento sull'Azure blob storage. Il software dell'IoT cam possiede una logica di _still detecting_, che unito ad
un meccanismo di _caching_ dei frames, evita il caricamento di un elevato numero di frames in un range di tempo breve.
---
Una risorsa **Blob Storage** è incaricata di immagazzinare tutte le immagini caricate dalla IoT cam contenenti dei volti da identificare.
Lo storage è stato pensato per poter organizzare file di detection provenienti da più IoT cam; i file infatti vengono salvati in subfolder, 
ciascuna rappresentante una diversa IoT cam. Questa strutturazione permette di aggiungere modularmente delle videocamere, e di poter 
distinguere le detection in base alla cam, e quindi al luogo, della rilevazione.
---
La **detection function** è un _trigger_ per il blob storage: viene invocata automaticamente ogni volta che un blob nello storage
viene aggiunto. I compiti di questa funzione sono:
* Identificare il volto contenuto nell'immagine ricevuta in input attraverso l'invocazione delle Face API dei Cognitive Services
* Registrare il risultato dell'identificazione all'interno di un database (di cui in seguito)
* Nel caso in cui il volto non viene identificato come autorizzato si trasmette una HTTP request alla function incaricata di 
inoltrare una notifica via mail agli amministratori di sistema

Questa function implementa una funzionalità di _flooding avoidance_, per cui non vengono persistite detections se queste contengono 
volti appartenenti ad una persona già registrata di recente. Quando una detection viene scartata, viene eliminato il relativo file nel
Blob Storage. Questa logica permette di ottimizzare l'uso delle risorse di storage (Blob Storage e Cosmos DB), che si traduce in un'ottimizzazione
dal punto di vista del _costo_, essendo quest'ultimo strettamente legato alla quantità di dati mantenuti nello storage.
---
La function **send mail** è predisposta per la ricezione di chiamate HTTP(s), ed è incaricata di inviare una notifica via mail agli amministratori
di sistema, indicando loro che è stato rilevato un volto non autorizzato. Per l'invio delle mail è stato utilizzato il servizio _SendGrid_ 
di Twilio, il quale espone degli endpoint per inviare email custom. La mail è composta da un oggetto standard, ed un corpo contenente alcune
informazioni sulla detection e l'immagine contenente il volto non identificato.
---
Una **web app** hostata sul servizio Azure App Service permette di accedere alle informazioni processate dal sottosistema di detection. 
La web app è pensata per consentire agli amministratori di sistema l'accesso alla cronologia delle detection, ed alla gestione degli 
utenti autorizzati all'accesso. In particolare le funzionalità dell'applicazione si possono riassumere come di seguito:
* accesso ad una dashboard riassuntiva dei dati statistici relativi detections composta da counters e grafici
* visualizzazone della cronologia degli accessi autorizzati
* visualizzazione delle cronologia degli accessi non autorizzati
* possibilità di registrare una persona autorizzata caricando l'immagine del suo volto (che verrà poi riconosciuto dal sottosistema di detection)

A seguire il manuale d'uso della Web App.

---
<img src="https://github.com/Access-Monitor/.github/blob/main/res/sequence_diagram.jpg" alt="sequence diagram del sistema">

Per maggiore chiarezza viene aggiunto il sequence diagram del sottosistema di detection che descrive lo scambio dei messaggi tra le
componenti durante la fase di detection & identification

## 

