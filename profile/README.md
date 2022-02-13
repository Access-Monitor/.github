# 🙈 Access Monitor
Access Monitor è un sistema software il cui obiettivo è fornire un servizio di security e controllo degli
accessi per luoghi chiusi al pubblico (in cui si consente l'accesso solo a del personale autorizzato).
Una o più videocamere IoT rilevano volti umani durante la registrazione e li inoltrano al sistema di autenticazione
che registra un accesso autorizzato o non autorizzato. Gli amministratori della security, oltre a poter indicare 
quali siano i volti autorizzati, possono avere accesso alla cronologia di tutti gli accessi autorizzati e non.

## ☁ Descrizione dei servizi utlizzati
* **App Service:** è un cloud service che consente di hostare web apps e RESTful APIs in un ambiente già confezionato. 
  Offre il vantaggio di non doversi preoccupare di amministrazione e impostazioni dell'ambiente, permettendo di scegliere il 
  linguaggio di programmazione che più si preferisce. Altro vantaggio, è quello di poter automatizzare 
  il deploy direttamente da GitHub. Oltre a semplificare molti compiti del programmatore, offre auto-scaling ed alta disponibilità. 
  
* **Blob Storage:** permette il salvataggio massiccio di dati non strutturati, in genere file binari, la cui interpretazione è a 
  carico di chi effettua la lettura. Alta disponibilità e consistenza forte sono i maggiori vantaggi che si possono ottenere utilizzando 
  tale cloud service. 
  
* **Azure Cosmos DB:** è un database NoSQL che garantisce un'elevata disponibilità ed una bassa latenza. Azure Cosmos DB semplifica l'amministrazione 
  del database con gestione, aggiornamenti e patch automatici.
  
* **Azure Function:** è una soluzione serverless che consente di scrivere meno codice, non preoccuparsi dell'infrastruttura e risparmiare sui costi. 
  Invece di preoccuparsi della distribuzione e della manutenzione dei server, l'infrastruttura cloud fornisce tutte le risorse aggiornate necessarie 
  per mantenere in esecuzione le tue applicazioni.

* **FaceAPI Cognitive Service:** fornisce una serie di algoritmi d'intelligenza artificiale che riconoscono e analizzano volti all'interno 
  di immagini. Il programmatore può usufruire di servizi di riconoscimento facciale senza doversi preoccupare di come essi siano implementati.


## 🏯 Architettura del sistema
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

## 🚀 Installazione e manuale d'uso

### Prerequisiti
* Microsoft Azure Subscription
* Python 3.x & [pip](https://pip.pypa.io/en/stable/installation/)
* Dispositivo con almeno un input per registrare video

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

Per l'interconnessione della functions con gli altri servizi descritti nella sezione [Architettura del sistema](#architettura-del-sistema) è necessario impostare
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

Per l'interconnessione della functions con gli altri servizi descritti nella sezione [Architettura del sistema](#architettura-del-sistema) è necessario impostare
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
Dal pannello di amministrazione recarsi al menù **Create a resource** > selezionare tra le opzioni possibili (o tramite la barra di ricerca)
**App Service** > **Create** > selezionare _Java 11_ come runtime stack e procedere con la creazione della risorsa.
Dalla pagina di gestione di App Service raggiungere il menù **Deployment Center**: da qui è possibile scegliere il metodo di deploy del codice,
tra cui metodi di Manual Deployment, e di Continuous Deployment (CI/CD).

Nel caso di deployment CI/CD, è necessario collegare il proprio account GitHub ad Azure, forkare la repository del progetto _AccessMonitorPortal_,
e connetterla ad Azure in CD selezionando **GitHub** come metodo di deployment, selezionando la repository ed il branch da cui attingere.
Verrà avviata una pipeline visibile dal proprio account GitHub nel tab **Actions** della propria repository.

Per l'interconnessione della functions con gli altri servizi descritti nella sezione [Architettura del sistema](#architettura-del-sistema) è necessario impostare
le chiavi di accesso sotto forma di ernvironment variable: recarsi nel menù **Settings** > **Configuration** e selezionare
**New Application Setting** per aggiungere una variabile d'ambiente. Le variabili necessarie sono:
```
> CosmosDBEndpoint = endpoint per l'accesso alla risorsa Cosmos DB
> CosmosDBKey = endpoint key per l'accesso alla risorsa Cosmos DB
> FaceAPIEndpoint = endpoint per l'accesso alla risorsa FaceAPI
> FaceAPISubscriptionKey = endpoint key per l'accesso alla risorsa FaceAPI
```

---

### Setup IoT Cam
Il progetto [VideoDetectionEngine](https://github.com/Access-Monitor/VideoDetectionEngine) contiene tutto il necessario per l'esecuzione 
della IoT camera. Il progetto deve essere eseguito su una macchina con almeno un dispositivo di input per la registrazione video (che sia la
webcam integrata di un laptop oppure una webcam con interfaccia USB). In caso di esecuzione su una macchina con multipli dispositivi
di registrazione video, verrà scelto il dispositivo di default di sistema. 

Clonare il progetto con git:
```
git clone https://github.com/Access-Monitor/VideoDetectionEngine
```

Entrare nella main folder del progetto ed installare le dependencies:
```
cd VideoDetectionEngine
pip3 install -r requirements.txt
```

Prima di avviare la detection dovrebbe essere impostata la variabile d'ambiente _CAMERA_ID_, un ID univoco della cam che distingue le sue 
detections dalle altre. Se la variabile d'ambiente non viene valorizzata assumerà il valore di _camera_01_.

Avviare la registrazione della camera col comando:
```
python .\detection.py
```

---

### Manuale d'uso della Web App

#### Login di un admin

Per accedere al portale web, è necessario effettuare il login come admin della sicurezza inserendo le proprie credenziali come
illustrato nella schermata in figura sottostante.

<img src="https://github.com/Access-Monitor/.github/blob/main/res/login_example.png" alt="esempio schermata di login">

#### Dashboard
 
Selezionando all'interno della barra laterale la voce "Dashboard" è possibile accedere all'area dedicata al monitoring generale.
La dashboard fornisce statistiche ed informazioni relative agli accessi. È possibile visionare:
* Il numero totale di accessi rilevati
* Il numero totale di accessi non autorizzati 
* Il numero totale di accessi autorizzati
* La percentuale di violazioni
* Un grafico che mostra e confronta il numero di accessi autorizzati e non autorizzati avvenuti nelle ultime 12 ore


<img src="https://github.com/Access-Monitor/.github/blob/main/res/dashboard_example.png" alt="esempio schermata di dashboard">

#### Accessi autorizzati

Selezionando all'interno della barra laterale la voce "Authorized" è possibile visualizzare tutti gli accessi autorizzati 
rilevati dalla IoT cam.
Apparirà per ogni accesso autorizzato:

* Un'immagine del volto rilevato
* Il nome del membro autorizzato rilevato
* La data e l'ora in cui è stato rilevato l'accesso

Le rilevazioni sono ordinate a partire dalla più recente.

<img src="https://github.com/Access-Monitor/.github/blob/main/res/authorized_example.png" alt="esempio schermata authorized">

#### Accessi non autorizzati

Selezionando all'interno della barra laterale la voce "Unauthorized" è possibile visualizzare tutti gli accessi non autorizzati
rilevati dalla IoT cam.
Apparirà per ogni accesso non autorizzato:

* Un'immagine del volto rilevato
* La data e l'ora in cui è stato rilevato l'accesso

Le rilevazioni sono ordinate a partire dalla più recente.

<img src="https://github.com/Access-Monitor/.github/blob/main/res/unauthorized_example.png" alt="esempio schermata unauthorized">

#### Membri autorizzati

Scegliendo all'interno della barra laterale la voce "Staff" è possibile visualizzare tutti i membri autorizzati.
Apparirà una tabella contente tutte le informazioni riguardo i membri.

<img src="https://github.com/Access-Monitor/.github/blob/main/res/member_example.png" alt="esempio schermata membri">

#### Aggiungere un membro autorizzato

Accedendo all'area "Staff" e premendo il pulsante verde locato in alto a destra si accederà alla pagina dedicata all'inserimento 
di un nuovo membro autorizzato.

All'interno dell'apposito form occorre inserire:

* Nome del membro (Solo lettere)
* Cognome del membro (Solo lettere)
* Email del membro (deve essere un email valida e non deve essere presente un altro membro con la stessa mail)
* Il ruolo del membro all'interno dell'organizzazione (Almeno due lettere)
* Il numero di telefono (dieci cifre)

Nel caso in cui uno dei campi sia errato verrà evidenziato in rosso, altrimenti verrà evidenziato in verde.
Quando tutti i campi saranno compilati correttamente si potrà premere su "SAVE" per salvare il nuovo membro. 
Il software effettua un controllo sull'immagine inserita, per cui se si tenta di inserire un immagine che non contiene 
nessun volto verrà notificato un errore.

<img src="https://github.com/Access-Monitor/.github/blob/main/res/addMember_example.png" alt="esempio schermata nuovo membro">

#### Rimuovere un membro autorizzato

Interagendo con la tabella che mostra tutti i membri autorizzati, è possibile rimuovere uno dei membri.
Per farlo occorre semplicemente premere il pulsante rosso "Remove" in corrispondenza del membro dello staff che si desidera 
eliminare. Tale rimozione non sarà retroattiva, pertanto tutti gli accessi rilevati in passato saranno notificati come non autorizzati.

<img src="https://github.com/Access-Monitor/.github/blob/main/res/remove_example.png" alt="esempio schermata rimozione membro">


#### Registrazione di un nuovo admin

Selezionando la voce "Register" dal menù laterale è possibile registrare un nuovo admin che potrà accedere alla piattaforma.
Per farlo occorrerà compilare il form di registrazione inserendo:

* Nome del nuovo admin (non può contenere numeri o caratteri speciali)
* Cogonme del nuovo admin (non può contenere numeri o caratteri speciali)
* Indirizzo email valido che non appartiene già ad un altro admin
* Password e ripeti Password (deve contenere almeno 8 caratteri, almeno una lettera ed un numero)

<img src="https://github.com/Access-Monitor/.github/blob/main/res/register_example.png" alt="esempio schermata registrazione admin">