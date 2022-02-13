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

## Installazione e manuale d'uso

### Prerequisiti
* Microsoft Azure Subscription

### Tipi di risorse utilizzate
* Azure Blob Storage
* Azure Serverless Functions
* Azure Face API Cognitive Services
* Azure Cosmos DB
* Azure App Service
* Twilio SendGrid

### Deployment risorse in cloud

#### Setup di Base
La creazione di qualsiasi tipo di risorsa nel cloud Azure necessita di una subscription. Accedere al portale Azure 
tramite account Microsoft, selezionare la voce **Subscriptions** > **Add** e seguire i passaggi indicati.
Una volta fatto verrà mostrata la pagina della subscription appena creata; da qui recarsi al menù **Resource Groups** e creare 
un nuovo gruppo di risorse. 
Ora si è pronti alla creazione delle risorse necessarie al progetto.

---

#### Azure Blob Storage Setup
Dal pannello di amministrazione recarsi al menù **Create a resource** > selezionare tra le opzioni possibili (o tramite la barra di ricerca)
**Storage Account** > **Create**. Associare la risorsa alla subscription ed al resource group precedentemente creati. Le opzioni di 
performance, redundancy e region possono essere impostate in base alle necessità.
Una volta creato lo storage account, recarsi al menù **containers** e creare 2 containers di blob:
* _accessmonitorblob_: servirà per la memorizzazione delle immagini caricate dalla IoT cam
* _membersauthorized_: servirà per la memorizzazione delle immagini delle persone autorizzate registrate dall'amministratore tramite la web app

Entrambi i container vengono creati con accesso _privato_, per questo motivo è necessario recuperare la _connection string_ che verrà utilizzata
per autenticare l'accesso allo storage. La _connection string_ si trova nel menù **Access Keys** > **Connection String**.

---

#### Azure Face API Cognitive Services
Dal pannello di amministrazione recarsi al menù **Create a resource** > selezionare tra le opzioni possibili (o tramite la barra di ricerca)
**Cognitive Services** > **Create**. La creazione di questo tipo di risorsa mette automaticamente a disposizione tutti gli endpoint della 
suite Face API. E' sufficiente recarsi nel menù **Keys and Endpoints** e memorizzare il campo **Endpoint** ed uno dei campi **KEY**;
entrambe queste informazioni permetteranno di autenticare le richieste HTTP.

---

#### Azure Cosmos DB
Dal pannello di amministrazione recarsi al menù **Create a resource** > selezionare tra le opzioni possibili (o tramite la barra di ricerca)
**Azure Cosmos DB** > **Create** > selezionare l'API di interfacciamento **Core (SQL)** e seguire le indicazioni per la creazione.
Una volta creata la risorsa, recarsi nel manù **Keys** e recuperare la proprietà **URI** e **PRIMARY KEY** per consentire l'accesso al database.

---

#### Detection Serverless Functions
Dal pannello di amministrazione recarsi al menù **Create a resource** > selezionare tra le opzioni possibili (o tramite la barra di ricerca)
**Function App** > **Create** > selezionare _Java_ alla proprietà **Runtime Stack** in verisone _11_ e procedere alla creazione.
Dalla pagina di gestione della function raggiungere il menù **Deployment Center**: da qui è possibile scegliere il metodo di deploy del codice,
tra cui metodi di Manual Deployment, e di Continuous Deployment (CI/CD).

Nel caso di deployment CI/CD, è necessario collegare il proprio account GitHub ad Azure, forkare la repository del progetto _Blob-Changes-Trigger_,
e connetterla ad Azure in CD selezionando **GitHub** come metodo di deployment, selezionando la repository ed il branch da cui attingere.
Verrà avviata una pipeline visibile dal proprio account GitHub nel tab **Actions** della propria repository.

Per l'interconnessione della functions con gli altri servizi descritti nella sezione _Architettura del sistema_ è necessario impostare
le chiavi di accesso sotto forma di ernvironment variable: recarsi nel menù **Settings** > **Configuration** e selezionare 
**New Application Setting** per aggiungere una variabile d'ambiente. Le variabili necessarie sono:
```
> AzureWebJobsStorage = connection string per l'accesso alla risorsa Blob Storage (utilizzata solo per l'implementazione del trigger)
> CosmosDBEndpoint = endpoint per l'accesso alla risorsa Cosmos DB
> CosmosDBKey = endpoint key per l'accesso alla risorsa Cosmos DB
> FaceAPIEndpoint = endpoint per l'accesso alla risorsa FaceAPI
> FaceAPISubscriptionKey = endpoint key per l'accesso alla risorsa FaceAPI
> UnauthorizedManagerEndpoint = endpoint per l'accesso alla risorsa Send Mail Function
> UnauthorizedManagerAccessKey = endpoint key per l'accesso alla risorsa Send Mail Function
> BlobStorageConnectionString = connection string per l'accesso alla risorsa Blob Storage
```

---

#### Send Mail Serverless Functions
Dal pannello di amministrazione recarsi al menù **Create a resource** > selezionare tra le opzioni possibili (o tramite la barra di ricerca)
**Function App** > **Create** > selezionare _Java_ alla proprietà **Runtime Stack** in verisone _11_ e procedere alla creazione.
Dalla pagina di gestione della function raggiungere il menù **Deployment Center**: da qui è possibile scegliere il metodo di deploy del codice,
tra cui metodi di Manual Deployment, e di Continuous Deployment (CI/CD).

Nel caso di deployment CI/CD, è necessario collegare il proprio account GitHub ad Azure, forkare la repository del progetto _unauthorizedmanager_,
e connetterla ad Azure in CD selezionando **GitHub** come metodo di deployment, selezionando la repository ed il branch da cui attingere.
Verrà avviata una pipeline visibile dal proprio account GitHub nel tab **Actions** della propria repository.

Per l'interconnessione della functions con gli altri servizi descritti nella sezione _Architettura del sistema_ è necessario impostare
le chiavi di accesso sotto forma di ernvironment variable: recarsi nel menù **Settings** > **Configuration** e selezionare
**New Application Setting** per aggiungere una variabile d'ambiente. Le variabili necessarie sono:
```
> CosmosDBEndpoint = endpoint per l'accesso alla risorsa Cosmos DB
> CosmosDBKey = endpoint key per l'accesso alla risorsa Cosmos DB
> FaceAPIEndpoint = endpoint per l'accesso alla risorsa FaceAPI
> FaceAPISubscriptionKey = endpoint key per l'accesso alla risorsa FaceAPI
> BlobStorageConnectionString = connection string per l'accesso alla risorsa Blob Storage
> FromMailAddress = indirizzo mail mittente
> SENDGRID_API_KEY = endpoint key per l'accesso alla risorsa SendGrid per l'invio delle mail
```
---

#### Twilio SendGrid
Dal pannello di amministrazione recarsi al menù **Marketplace** > selezionare tra le opzioni possibili (o tramite la barra di ricerca)
**Twilio SendGrid** > selezionare un piano di billing, poi **Set up + create**. Verrà creata una risorsa connessa al servizio SaaS di Twilio, 
da qui è possibile raggiungere il pannello di controllo di Twilio in cui è necessario creare un API Key in **settings** > **API Keys** > **Create API Key**

---

#### Azure App Service

---

### Setup IoT Cam


