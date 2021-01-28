# Immuni: visione d'insieme

Immuni è la soluzione di notifiche di esposizione del Governo Italiano, realizzata dal Commissario straordinario per l'emergenza COVID-19 (Presidenza del Consiglio dei Ministri), in collaborazione con il Ministero della Salute e il Ministero per l'innovazione tecnologica e la digitalizzazione. Il sistema usa esclusivamente infrastrutture pubbliche situate all'interno dei confini nazionali. È gestito nella sua interezza dall'azienda pubblica Sogei S.p.A. Il codice sorgente antecedente al 13 ottobre 2020 è stato sviluppato per la Presidenza del Consiglio dei Ministri da parte di Bending Spoons S.p.A. ed è reso disponibile sotto licenza GNU Affero General Public License versione 3.

## Sommario

- [Executive summary](#executive-summary)
- [Introduzione](#introduzione)
- [Visione e obiettivi](#visione-e-obiettivi)
- [Principi](#principi)
- [Il prodotto](#il-prodotto)
  - [Come funziona](#come-funziona)
- [Dati statistici](#dati-statistici)
  - [Informazioni epidemiologiche](#informazioni-epidemiologiche)
  - [Informazioni operative](#informazioni-operative)
- [Privacy](#privacy)

## Executive summary

- Immuni è una soluzione tecnologica con al centro **un'app mobile per iOS e Android**. Ci aiuta a combattere l'epidemia di COVID-19 avvertendo utenti a rischio di aver contratto il virus il prima possibile, **anche prima di averne sviluppato i sintomi**. Questi utenti possono di conseguenza autoisolarsi per evitare di contagiare altri e consultare un medico.
- La progettazione e lo sviluppo di Immuni sono basati su sei principi: **utilità**, **accessibilità**, **precisione**, **privacy**, **scalabilità** e **trasparenza**.
- Immuni include un sistema di **notifiche di esposizione** che sfrutta la tecnologia **Bluetooth Low Energy**:
	- Quando due utenti entrano in sufficiente prossimità fisica per abbastanza tempo, i loro dispositivi si scambiano e registrano nella memoria locale i rispettivi _rolling proximity identifiers_ (codici casuali). I codici casuali sono generati a partire da _temporary exposure keys_ (chiavi temporanee di esposizione) e cambiano diverse volte nell'arco di ogni ora. Le chiavi temporanee di esposizione sono generate casualmente una volta al giorno.
	- Quando un utente risulta positivo al SARS-CoV-2, il virus che causa il COVID-19, gli viene data la possibilità di caricare su un server le più recenti chiavi temporanee di esposizione. Questa operazione richiede l'assistenza di un **operatore sanitario**.
	- L'app scarica periodicamente nuovi chiavi temporanee di esposizione e le usa per ricavare i codici casuali trasmessi dai dispositivi di utenti infetti. Questi codici vengono confrontati con quelli presenti nella memoria del dispositivo e, nel caso di in cui sia avvenuta un'esposizione a rischio, l'utente **viene avvertito mediante una notifica**.
	- Il sistema **non fa uso di dati di geolocalizzazione** di alcun tipo, inclusi i dati del GPS. Per questo motivo l'app non può sapere dove sia avvenuto il contatto a rischio, né può sapere chi fossero gli utenti coinvolti.
 - Per implementare la funzionalità delle notifiche di esposizione, Immuni sfrutta **il framework per le notifiche di esposizione di Apple, Google e HMS Core**. Questo consente a Immuni di essere più affidabile di quanto non consentirebbe un qualsiasi altro approccio.
 - Oltre alle chiavi temporanee di esposizione, l'app Immuni invia al server alcuni dati statistici. Questi includono **informazioni epidemiologiche e operative** e sono inviati al fine di aiutare il Servizio Sanitario Nazionale a curare i propri assistiti in modo efficace.
 - Immuni è stato sviluppato prestando grande attenzione alla privacy degli utenti e diverse misure sono state adottate per proteggerla al meglio. Ad esempio l'app **non raccoglie alcun dato personale che permetta l'identificazione dell'utente**, come il nome, l'età, l'indirizzo, l'email o il numero di telefono.

## Introduzione
Il mondo intero è unito dalla volontà di fermare la diffusione del COVID-19, la malattia causata dal virus SARS-CoV-2. La pandemia sta mettendo a rischio la salute delle persone e sta danneggiando seriamente le economie su scala globale.

Molti esperti concordano nel ritenere probabile che il futuro riservi nuove pandemie. Alcune di queste potrebbero rivelarsi ancora più pericolose per l'umanità di quella che stiamo combattendo ora.

L'innovazione tecnologica può dare un contributo determinante a questa sfida. Immuni è uno degli strumenti e delle iniziative messe in campo dal Governo italiano per aiutare a rallentare la diffusione della malattia e ad accelerare il ritorno alla vita normale.

Questo documento fornisce una visione d'insieme di Immuni e rappresenta quindi il punto di partenza migliore. Informazioni più dettagliate si trovano nei documenti seguenti (in inglese):

- [Product](/Product.md)
- [Technology](/Technology.md)
- [Application Security](/Application%20Security.md)
- [Privacy-Preserving Analytics](/Privacy-Preserving%20Analytics.md)
- [Traffic Analysis Mitigation](/Traffic%20Analysis%20Mitigation.md)

Il codice di Immuni è pubblicamente disponibile sotto licenza [GNU Affero General Public License versione 3](https://www.gnu.org/licenses/agpl-3.0.en.html).

## Visione e obiettivi

Immuni è una soluzione tecnologica con al centro un'app mobile.

Ci aiuta a combattere epidemie, a partire dal COVID-19.

- L'app si propone di allertare con una notifica gli utenti a rischio di aver contratto il virus il prima possibile, anche quando sono asintomatici.
- Questi utenti possono isolarsi per evitare di contagiare altri, minimizzando la diffusione del virus e accelerando il ritorno alla vita normale per la maggior parte delle persone.
- Venendo preventivamente allertati, possono inoltre consultare un medico e ridurre così il rischio di incorrere in conseguenze gravi per la propria salute.

Immuni è stato pensato per rispondere alla crisi in corso, ma gli strumenti che sono stati sviluppati ci possono rendere più preparati ad affrontare situazioni simili nel caso in cui si ripresentino in futuro.

## Principi

I principi che guidano il progetto Immuni sono i seguenti:

- **Utilità.** L'app dev'essere utile nel raggiungere la visione e gli obiettivi del progetto definiti sopra. È essenziale avvertire una percentuale quanto più alta possibile delle persone che sono considerevolmente a rischio nonché farlo il prima possibile. Questo è il principio più importante.
- **Accessibilità.** Per motivi di equità e per supportare la più ampia diffusione, Immuni dev'essere accessibile al maggior numero di persone possibile. Questo principio impatta decisioni prese in ogni ambito, inclusi esperienza utente, design, localizzazione e tecnologia.
- **Precisione.** Immuni mira ad avvertire solo gli utenti che sono sostanzialmente a rischio di aver contratto il virus. Questo è fondamentale per ridurre al minimo le conseguenze psicologiche dell'essere avvertiti di un potenziale rischio e perché una percentuale troppo alta di falsi positivi porterebbe a una perdita di fiducia nell'app. In aggiunta, quanto più è precisa l'app, tanto più efficientemente il Servizio Sanitario Nazionale può prendersi cura dei suoi assistiti, prioritizzando quelli a più alto rischio.
- **Privacy.** Immuni deve proteggere la privacy degli utenti mantenendo al contempo la sua efficacia. Riuscire a guadagnare e a mantenere la fiducia degli utenti è cruciale; un fallimento in questo senso ne ridurrebbe la diffusione.
- **Scalabilità.** È importante che Immuni sia ampiamente diffusa in tutto il Paese. Affinché questo avvenga la tecnologia su cui si basa il sistema dev’essere scalabile e il Servizio Sanitario Nazionale dev’essere in grado di gestire il carico aggiuntivo causato dall’app.
- **Trasparenza.** Tutti devono avere accesso alla documentazione relativa a tutte le componenti di Immuni e alle motivazioni che sottendono le più importanti scelte progettuali. Inoltre, il codice dell'app è aperto. Questo permette agli utenti di verificare che l'app funziona come da documentazione e consente alla comunità di esperti di migliorarla.

## Il prodotto

Immuni è una soluzione tecnologica basata su un'app per smartphone disponibile per iOS e Android.

La soluzione include un sistema di notifiche di esposizione che aiuta ad avvertire per tempo utenti potenzialmente positivi al virus SARS-CoV-2. Il sistema tiene traccia dei contatti tra utenti di Immuni, anche quando non si conoscono. Quando un utente risulta positivo al tampone, l'app utilizza questo sistema per avvertire altri utenti a rischio. Il sistema funziona grazie al Bluetooth Low Energy e non utilizza in alcun modo dati di geolocalizzazione, inclusi i dati del GPS. L'app rileva l'avvenuto contatto con utente positivo, ne conosce la durata e può stimare la distanza che separava i due utenti, ma non può in alcun modo sapere dove il contatto è avvenuto o le identità degli individui coinvolti.

L'app suggerisce agli utenti a rischio il da farsi. Le indicazioni consigliano l'autoisolamento (che aiuta a minimizzare la diffusione della malattia) e suggeriscono di contattare il Medico di Medicina Generale per ricevere le cure più appropriate e ridurre il rischio di gravi complicanze.

Come già detto, il sistema di notifiche di esposizione di Immuni sfrutta il Bluetooth Low Energy. Questo presenta alcuni vantaggi rispetto a una soluzione basata sul tracciamento della posizione:

- **Il sistema è più preciso.** La geolocalizzazione ha in molti contesti una precisione nell'ordine della decina di metri. Al contrario, i segnali Bluetooth Low Energy permettono di catturare contatti avvenuti in un raggio di pochi metri dall'utente. Questo tipo di contatti sono quelli rilevanti per la trasmissione del SARS-CoV-2.
- **L'uso della batteria è più efficiente.** Il Bluetooth Low Energy è eccellente dal punto di vista dell'efficienza energetica. Questo è importante perché ci si può aspettare che un consumo di batteria più alto causi un maggior numero di disinstallazioni.
- **Non è necessario alcun dato di geolocalizzazione.** Grazie al Bluetooth Low Energy, i contatti sono catturati senza tracciare la posizione degli utenti. Questo può migliorare il gradimento pubblico dell'app e facilitarne la diffusione, aumentandone l'utilità.

Per l'implementazione della funzionalità delle notifiche di esposizione, Immuni sfrutta il framework di notifiche di esposizione di Apple, Google e HMS Core (qui [la documentazione di Apple](https://www.apple.com/covid19/contacttracing) e [la documentazione di Google](https://www.google.com/covid19/exposurenotifications/)). Questo permette a Immuni di superare alcune limitazioni tecniche e di essere più affidabile.

### Come funziona

Segue una descrizione semplificata e ad alto livello. Per maggiori dettagli, rimandiamo al resto della documentazione di Immuni.

Dopo essere stata installata e configurata su un dispositivo (_dispositivo_A_), l'app genera una _chiave temporanea di esposizione_. Questa chiave è generata in modo casuale e cambia quotidianamente. L'app inizia anche a trasmettere un segnale Bluetooth Low Energy. Il segnale contiene un _codice casuale_ (_ID_A1_, che per semplicità assumiamo essere fisso in questo esempio). Il codice casuale è generato a partire dalla chiave temporanea di esposizione corrente. Quando un altro dispositivo (_device_B_) con l'app installata riceve questo segnale, registra _ID_A1_ nella sua memoria locale. Contemporaneamente, _dispositivo_A_ registra il codice casuale del _dispositivo_B_ (_ID_B1_, che assumiamo ugualmente essere fisso in questo esempio).

Se l'utente del _dispositivo_A_ risulta in seguito positivo al test per il SARS-CoV-2, coerentemente con il protocollo definito dal Servizio Sanitario Nazionale gli viene data la possibilità di caricare sul server di Immuni le chiavi temporanee di esposizione da cui l'app Immuni può derivare i codici casuali trasmessi negli ultimi giorni dal _dispositivo_A_ (_ID_A1_ tra questi). Il _dispositivo_B_ confronta periodicamente le chiavi caricate sul server con la lista locale di codici casuali. _ID_A1_ determina una corrispondenza. L'app avverte quindi l'utente del _dispositivo_B_ che potrebbero essere a rischio e fornisce suggerimenti sul da farsi (ad esempio, autoisolarsi e chiamare il Medico di Medicina Generale).

Nella pratica, il fatto che gli utenti del _dispositivo_A_ e del _dispositivo_B_ siano stati vicini non è sufficiente a determinare se l'utente del _dispositivo_B_ sia a rischio. Immuni determina il rischio sulla base della durata dell'esposizione e della distanza tra i due dispositivi. La distanza viene stimata sulla base dell'attenuazione del segnale Bluetooth Low Energy ricevuto dal _dispositivo_B_. Più lunga è l'esposizione e più ravvicinato il contatto, più alto è il rischio che un contagio sia avvenuto. Un contatto che sia durato solo un paio di minuti e che sia avvenuto a diversi metri di distanza sarà generalmente considerato a basso rischio. Il modello di rischio potrebbe essere perfezionato nel tempo con l'aumentare della informazioni disponibili sul SARS-CoV-2.

Si noti che la stima della distanza è soggetta a errori. L'attenuazione del segnale Bluetooth Low Energy dipende infatti da fattori quali l'orientazione relativa dei due dispositivi e gli ostacoli (incluso il corpo umano) che si trovano tra di essi. Posto che sfruttare questa informazione è utile per aumentare la precisione della valutazione del rischio di contagio elaborata da Immuni, si possono avere casi d'errore in questa valutazione.

Il codice casuale che viene diffuso dall'app è generato a partire dai codici temporanei di esposizione che sono a loro volta casuali e non contengono alcuna informazione sul dispositivo e tanto meno sull'utente. In aggiunta, questi codici _ruotano_, in altre parole cambiano più volte nell'arco di un'ora, per proteggere ancora meglio la privacy degli utenti di Immuni.

Per assicurarsi che solo gli utenti che sono risultati effettivamente positivi al test per il SARS-CoV-2 carichino le loro chiavi sul server, la procedura di caricamento può essere effettuata soltanto con la collaborazione di un operatore sanitario autenticato. L'operatore chiede all'utente di fornire un codice generato dall'app e lo inserisce in un'interfaccia web dedicata. Il caricamento può avere successo soltanto se il codice usato dall'app per autenticare i dati corrisponde a quello inserito nel sistema dall'operatore sanitario.

## Dati statistici

In aggiunta alle chiavi temporanee di esposizione, alcune informazioni aggiuntive sono inviate al server e analizzate per accertarsi del corretto funzionamento del sistema:

- Informazioni epidemiologiche
- Informazioni operative

Questi dati sono importanti per la gestione efficace del sistema da parte del Servizio Sanitario Nazionale, ovvero per massimizzare l'efficacia del sistema delle notifiche di esposizione e per fornire un'assistenza sanitaria ottimale agli utenti.

L'ottimizzazione resa possibile dalle informazioni epidemiologiche e operative è più efficace se effettuata a livello locale. Le regioni e province autonome italiane differiscono per politiche sanitarie e risorse disponibili. Inoltre l'epidemia può essere più o meno diffusa in zone diverse. Per questo motivo, quando questi dati vengono inviati al server, l'app allega la provincia di domicilio dell'utente che viene fornita durante la prima apertura dell'app.

### Informazioni epidemiologiche

Le uniche informazioni epidemiologiche che Immuni raccoglie sono relative all'esposizione a utenti infetti. Questi dati includono:

- Il giorno in cui l'esposizione è avvenuta
- La durata dell'esposizione
- L'attenuazione del segnale, usata nella stima della distanza tra i dispositivi dei due utenti

L'app può inviare informazioni epidemiologiche al server solo al momento del caricamento delle chiavi temporanee di esposizione. Quando un operatore sanitario comunica all'utente la positività al test per il SARS-CoV-2, le informazioni epidemiologiche relative agli ultimi 14 giorni eventualmente disponibili verranno caricate assieme alle chiavi. Affinché questo avvenga, il caricamento dei dati deve essere disposto dall'utente e approvato dall'operatore sanitario.

Per proteggere la privacy dell'utente, i dati relativi alla loro esposizione a utenti potenzialmente contagiosi sono circoscritti. Per esempio, la durata dell'esposizione viene misurata a incrementi di cinque minuti e limitata a 30 minuti. Immuni non ha modo di determinare se esposizioni avvenute in giorni diversi hanno coinvolto lo stesso utente infetto. L'app effettua periodicamente dei caricamenti fittizi per mitigare il rischio che qualcuno possa ottenere informazioni sensibili sull'utente analizzando il traffico di rete.

La raccolta di questi dati aiuta il Servizio Sanitario Nazionale a ottimizzare il modello di rischio dell'app. Osservando come le informazioni epidemiologiche (ad esempio la durata dell'esposizione) correlano con un avvenuto contagio, si ha la possibilità di migliorare il modello di rischio dell'app e, di conseguenza, di migliorare la sua precisione. Questo può aumentare a sua volta l'efficacia del sistema delle notifiche di esposizione. Si noti che la valutazione del rischio avviene sempre sul dispositivo dell'utente, mentre l'app acquisisce l'ultima versione del modello di rischio dal server.

## Informazioni operative

In aggiunta alle informazioni di cui sopra, possono essere caricati alcuni dati sull'attività del dispositivo e sulle notifiche di esposizione. Questi dati includono:

- La piattaforma del dispositivo (iOS/Android)
- La concessione del permesso per l'utilizzo del framework delle notifiche di esposizione di Apple, Google e HMS Core (vero/falso)
- Lo stato di attivazione del Bluetooth (acceso/spento)
- La concessione del permesso per l'invio di notifiche locali (vero/falso)
- Il fatto che l'utente sia stato avvertito di un'esposizione a rischio durante l'ultima rilevazione di esposizione (ovvero dopo che l'app ha scaricato le nuove chiavi temporanee di esposizione dal server e ha rilevato se l'utente è stato esposto a utenti positivi al SARS-CoV-2) (vero/falso)
- La data in cui è avvenuta l'ultima esposizione a rischio

Il caricamento può avvenire dopo la rilevazione di esposizione. Le informazioni operative vengono caricate automaticamente.

Per proteggere la privacy degli utenti, i dati sono caricati senza che l'utente venga autenticato in alcun modo, ad esempio tramite email o numero di telefono. Inoltre, l'analisi del traffico viene ostacolata grazie a dei caricamenti fittizi.

Grazie a questi dati è possibile stimare il livello di diffusione dell'app nel paese, non solo sulla base dei download (una misura poco significativa) ma sulla base dei dispositivi su cui l'app funziona correttamente. Queste informazioni sono di grande aiuto, in quanto l'utilità di Immuni dipende fortemente dall'utilizzo da parte della popolazione. Con il supporto di questi dati, il Servizio Sanitario Nazionale può prendere decisioni migliori in una serie di aree determinanti per l'efficacia del sistema delle notifiche di esposizione e per la fornitura di assistenza sanitaria ottimale. Queste aree includono la progettazione del prodotto, la tecnologia e la comunicazione.

Questi dati possono aiutare a ottimizzare l'allocazione di risorse. Avvertendo gli utenti a rischio, l'app comporterà un aumento del volume delle interazioni con il Servizio Sanitario Nazionale. Per questo motivo, riuscire a stimare il numero di utenti che verranno avvertiti dal sistema può aiutare il Servizio Sanitario Nazionale ad allocare le risorse in modo efficiente. La conoscenza della data in cui è avvenuta l'ultima esposizione a rischio aiuta a stimare quando i sintomi potrebbero manifestarsi. Questi fattori permettono al Servizio Sanitario Nazionale di adattare gli interventi con ancora più precisione.

## Privacy

Immuni è stato e continua ad essere progettato e sviluppato prestando molta attenzione alla privacy degli utenti. Si tratta di un diritto fondamentale ed è giusto fare tutto il possibile per difenderlo. Riteniamo anche che una protezione eccezionale della privacy sia essenziale per far apprezzare l'app da quante più persone possibili, aumentandone al massimo l'utilità.

Forniamo qui una lista di alcune delle misure adottate da Immuni per proteggere la privacy degli utenti:

- L'app non raccoglie dati personali di alcun tipo che rivelino l'identità dell'utente. Ad esempio non raccoglie il nome, la data di nascita, l'indirizzo, l'email o il numero di telefono dell'utente.
- L'app non raccoglie dati di geolocalizzazione, inclusi i dati del GPS. I movimenti degli utenti non sono tracciati in alcun modo.
- I codici casuali trasmessi dall'app sono generati a partire da chiavi temporanee di esposizione casuali che non contengono alcuna informazione riguardo al dispositivo e tanto meno all'utente. Inoltre questi codici cambiano più volte nel corso di un'ora.
- Le informazioni epidemiologiche relative all'esposizione a utenti potenzialmente contagiosi che vengono caricate sul server sono circoscritte. Ad esempio, la durata dell'esposizione è misurata a incrementi di cinque minuti e limitata a 30 minuti. Inoltre Immuni non ha modo di determinare se esposizioni avvenute in giorni diversi abbiano coinvolto lo stesso utente infetto.
- Le informazioni operative sono caricate senza richiedere che l'utente si autentichi in alcun modo (ad esempio verificando un numero di telefono o un'email).
- L'app effettua periodicamente caricamenti fittizi per mitigare il rischio che qualcuno ottenga informazioni sensibili sulla base dell'analisi del traffico di rete.
- I dati memorizzati sul dispositivo sono crittografati.
- Tutte le connessioni tra dispositivo e server sono crittografate.
- Tutti i dati, sia che siano memorizzati sul dispositivo o sul server, sono eliminati quando non più necessari e in ogni caso non oltre il 31 dicembre 2020.
- Il Ministero della Salute sarà il titolare del trattamento. I dati saranno utilizzati solo allo scopo di contenere l'epidemia di COVID-19 o per la ricerca scientifica.
- I dati saranno archiviati su server situati in Italia e gestiti da entità a controllo pubblico.