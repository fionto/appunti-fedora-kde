# Snapshot su btrfs: i concetti di base

Questo capitolo non contiene comandi da eseguire. Serve a mettere in ordine i concetti che useremo nei capitoli pratici: che cos'è un filesystem, che cosa rende btrfs diverso dagli altri, che cosa sono subvolumi e snapshot, e come lo strumento che ho scelto, **Snapper**, organizza tutto questo. Quando nel testo compare un output reale, è quello del mio sistema, così i concetti restano legati a una macchina concreta.

---

## 0. Prima di tutto: uno snapshot non è un backup

Il rischio di confondere le due cose è alto, quindi conviene chiarirlo subito. Uno snapshot vive sullo stesso disco, dentro lo stesso filesystem, e dipende dagli stessi blocchi fisici dei dati che protegge. Se il disco si rompe, se il portatile viene rubato o se la cifratura viene compromessa, gli snapshot vanno persi insieme al resto.

Quello che uno snapshot protegge bene è un altro tipo di incidente: un aggiornamento che rompe il sistema, un comando lanciato nella cartella sbagliata, un file cancellato per errore, una configurazione modificata e poi rimpianta. In questi casi lo snapshot è un punto di ritorno rapido e a costo quasi nullo. Il backup, invece, è una copia **indipendente** dal disco di origine. Le due cose si completano, e nella repo il backup vero verrà trattato in una sezione separata.

---

## 1. Che cos'è un filesystem, in breve

Un disco, a basso livello, è un insieme di blocchi numerati che contengono byte. Da solo non sa nulla di file, cartelle o permessi. Il **filesystem** è lo strato che organizza questi blocchi in una struttura comprensibile: decide dove si trova ogni file, quali blocchi occupa, come sono annidate le cartelle, quali sono i permessi, le date di modifica e tutti gli altri metadati.

Su Linux esistono diversi filesystem, e la differenza riflette filosofie diverse. **ext4** è il filesystem "classico" di molte distribuzioni: robusto, maturo, ben conosciuto, ma senza snapshot nativi. Per avere copie puntuali bisogna ricorrere a strumenti esterni o a meccanismi a livello di volume. **xfs** è pensato soprattutto per grandi volumi e carichi intensi, e anche lui non gestisce snapshot in modo integrato come btrfs. **btrfs** nasce con un'idea diversa: integrare nello stesso strato funzioni che altrove sono separate, come i sotto-alberi indipendenti, gli snapshot, la compressione trasparente e la verifica dell'integrità dei dati.

Nel mio sistema il filesystem btrfs non sta direttamente sulla partizione fisica. Come mostra `findmnt`, la sorgente è un volume `/dev/mapper/luks-...`, cioè un contenitore cifrato con LUKS che sta sopra la partizione reale. Tra il disco e btrfs c'è quindi almeno uno strato intermedio, e gli snapshot agiscono sopra quello strato, non direttamente sul disco fisico.

---

## 2. Btrfs: copy-on-write e alberi

Il concetto più importante per capire gli snapshot è il **copy-on-write**, spesso abbreviato in CoW. Quando un dato viene modificato, il filesystem non lo sovrascrive sul posto: scrive la nuova versione in un blocco libero diverso e poi aggiorna i riferimenti che puntano a quel dato.

Un'analogia che regge bene è l'indice di un libro. Le pagine sono i blocchi di dati, l'indice è la struttura che dice dove si trova ogni contenuto. Con il metodo tradizionale, se correggo un paragrafo, cancello il testo vecchio e scrivo quello nuovo nella stessa pagina. Con il copy-on-write stampo una pagina nuova con il paragrafo corretto e aggiorno l'indice perché punti lì. La pagina vecchia non viene toccata e resta dov'è, finché qualcuno nell'indice la referenzia ancora. È proprio questa proprietà che rende possibili gli snapshot: se conservo l'indice com'era prima, conservo anche le pagine a cui puntava.

Btrfs organizza le sue strutture interne come **alberi** (più precisamente, alberi B). Non serve conoscerne i dettagli per il resto del capitolo: basta sapere che file, cartelle e metadati sono registrati in questi alberi, e che ogni modifica produce una nuova versione del percorso dell'albero interessato.

Ogni volta che il filesystem chiude una transazione di scrittura, incrementa un contatore interno chiamato **generation**, o `gen`. È un numero che cresce nel tempo e serve a dire "questo stato è più recente di quello". Lo ritroveremo nel capitolo sul comando `btrfs subvolume list`.

Btrfs aggiunge altre due funzioni che influenzano il comportamento degli snapshot. La prima è il **checksum**: dati e metadati sono accompagnati da una somma di controllo, così il filesystem può accorgersi di una corruzione silenziosa, cioè di un blocco alterato senza che nessuno se ne accorgesse. La seconda è la **compressione trasparente**. Nel mio sistema compare l'opzione `compress=zstd:1`, che significa: i dati vengono compressi al momento della scrittura con l'algoritmo zstd, a un livello basso. Lo spazio occupato reale può essere minore di quello che i file sembrano occupare, e questo si riflette anche sugli spazi degli snapshot.

---

## 3. Il subvolume

Un **subvolume** è il concetto che più spesso confonde chi arriva da ext4. La cosa da fissare subito è che un subvolume non è una partizione.

Una partizione ha una dimensione fissa, decisa al partizionamento, e il sistema la vede come un disco a sé. Un subvolume è un sottoalbero all'interno del filesystem btrfs. Può essere montato in un punto qualsiasi della gerarchia del sistema, come una cartella con un'identità propria, ma non ha una dimensione riservata: tutti i subvolumi dello stesso filesystem attingono allo stesso spazio libero. Creandone uno nuovo non tolgo niente agli altri in anticipo; lo spazio viene usato man mano che i file vengono scritti.

Questa flessibilità ha una conseguenza pratica: se in un subvolume si accumulano molti dati, lo spazio libero diminuisce anche per gli altri, perché sono tutti nello stesso contenitore.

Ogni subvolume ha un **ID numerico**. Il filesystem stesso è organizzato come un subvolume speciale che fa da radice di tutto: ha ID 5 ed è chiamato **top level**. È la radice dell'albero btrfs. Di norma non viene montato per l'uso quotidiano, ma contiene al suo interno gli altri subvolumi come figli.

Il mio sistema lo mostra chiaramente. L'output di `btrfs subvolume list` riporta:

```text
ID 256 gen 7616 top level 5 path root
ID 257 gen 7616 top level 5 path home
```

`ID 256` è il subvolume `root`, `gen 7616` è la generazione in cui è stato modificato l'ultima volta, `top level 5` dice che il suo genitore è la radice del filesystem, `path root` è il suo nome dentro quella radice. Per `home` vale lo stesso ragionamento con ID 257. I due sono **fratelli**, figli diretti della radice, e non uno dentro l'altro.

Gli ID sono numeri e non nomi per una ragione precisa: il sistema, quando monta un subvolume, lo riconosce tramite l'ID, mentre il nome è un'etichetta comoda per noi. Gli ID compaiono anche nelle opzioni di montaggio, come vedremo nella sezione sul ripristino.

Anche `findmnt` conferma questa struttura. Nella riga di `/` troviamo `[/root]` e `subvol=/root`: il filesystem è montato sulla radice del sistema prendendo il contenuto del subvolume `root`. Nella riga di `/home` troviamo `[/home]` e `subvol=/home`: la cartella `/home` è il contenuto del subvolume `home`. La riga di `/home` è indentata sotto quella di `/` perché `/home` è un punto di montaggio dentro `/`, ma punta a un subvolume diverso.

---

## 4. Lo snapshot

Uno **snapshot**, in btrfs, è un subvolume creato come copia puntuale di un altro. La parola "copia" va presa con cautela, perché non c'è duplicazione dei dati. Al momento della creazione, lo snapshot e l'originale condividono **gli stessi blocchi**: lo snapshot è un nuovo indice che punta alle stesse pagine dell'indice originale.

Per questo uno snapshot appena creato occupa quasi nulla. Lo spazio cresce solo quando i due subvolumi divergono. Se modifico un file nell'originale, per la regola del copy-on-write la nuova versione finisce in un blocco nuovo: l'originale punta al blocco nuovo, lo snapshot continua a puntare al vecchio. Il blocco vecchio resta occupato finché lo snapshot esiste, e il blocco nuovo occupa spazio in più. Lo spazio aggiuntivo corrisponde quindi alle differenze accumulate, non alla dimensione totale dei dati.

Uno snapshot può essere **in sola lettura** o **in lettura e scrittura**. Quelli in sola lettura non si modificano, e sono quelli che ci interessano per il ripristino: uno snapshot è un punto di riferimento immutabile, non un'area di lavoro. Quelli in lettura e scrittura sono più flessibili ma meno adatti a questo scopo, perché una modifica li allontana dallo stato che volevamo conservare.

Il ciclo di vita di uno snapshot ha tre fasi: creazione, divergenza e cancellazione. Eliminare uno snapshot non libera automaticamente tutto lo spazio che sembrava occupare. Libera solo i blocchi che nessun altro subvolume referenzia più. Se un blocco è condiviso con l'originale o con un altro snapshot, resta occupato. Per questo a volte si cancella un vecchio snapshot e il disco non cambia: quei blocchi erano ancora in uso altrove.

C'è infine una regola di struttura che guida le scelte pratiche. Uno snapshot cattura il contenuto del subvolume **senza** includere i subvolumi annidati dentro di esso. Se `root` contenesse un subvolume chiamato `var/lib/libvirt`, lo snapshot di `root` lo lascerebbe fuori.

---

## 5. Che cosa finisce davvero dentro uno snapshot

Dire "lo snapshot del sistema" è comodo, ma bisogna sapere che cosa contiene. Il mio sistema ha due subvolumi principali: `root`, montato su `/`, e `home`, montato su `/home`. È la configurazione più semplice che Fedora propone durante l'installazione con btrfs, ed è quella che ho.

Con questo layout tutto ciò che sta sotto `/` e non è un subvolume separato finisce in `root`. Quindi anche `/var`, con i log, le cache dei pacchetti, i dati delle applicazioni e gli eventuali container, fa parte di `root`, e quando creo uno snapshot di `root` questi dati entrano nello snapshot.

È una conseguenza con due facce. Da un lato è comodo: lo snapshot riflette lo stato completo del sistema, compresi configurazioni e pacchetti. Dall'altro può essere un problema: log e cache cambiano continuamente, quindi lo snapshot di `root` diverge più rapidamente del necessario e fa crescere lo spazio occupato. Inoltre, ripristinando lo stato del sistema, anche i log tornano indietro, il che può essere sgradito se servono per capire che cosa è andato storto.

Esiste un'alternativa, usata in configurazioni più avanzate: separare in subvolumi distinti cartelle come `/var/log`, `/var/cache` o i dati dei container, in modo che restino fuori dagli snapshot di `root`. La guida di Luca Giulianini, che è una delle fonti di questa repo, propone proprio questo schema. Non è l'unico modo corretto: dipende da quanto spazio si ha, da quanto si vuole conservare e da come si usa la macchina. Nel mio caso la configurazione è quella semplice, e la questione resta aperta: deciderò dopo aver visto quanto cresce lo spazio occupato dagli snapshot.

Lo stesso ragionamento vale per `/home`. Essendo un subvolume separato, il suo snapshot non include `root`, e viceversa. Questo permette di gestire dati personali e sistema con regole diverse.

---

## 6. Lo spazio: che cosa significa davvero "occupa poco"

Quando si guarda il disco con un comando come `df`, il valore mostrato riflette lo spazio del filesystem nel suo complesso, non lo spazio appartenente a un singolo snapshot.

Per ragionare in modo corretto servono due nozioni. La prima è lo **spazio referenziato**: quanti dati vede un subvolume o uno snapshot, cioè i blocchi a cui punta, indipendentemente dal fatto che siano condivisi. La seconda è lo **spazio esclusivo**: quanti blocchi appartengono solo a quel subvolume o snapshot, cioè quelli che verrebbero liberati se venisse cancellato. È lo spazio esclusivo che conta quando si decide cosa eliminare. Uno snapshot può avere uno spazio referenziato grande e uno spazio esclusivo quasi nullo, perché quasi tutti i suoi dati sono condivisi con l'originale.

Ne deriva una conseguenza pratica: se si accumulano molti snapshot con poche differenze tra uno e l'altro, lo spazio cresce lentamente ma in modo costante. Se invece il sistema cambia molto tra uno snapshot e il successivo, per esempio durante grandi aggiornamenti o installazioni pesanti, ogni snapshot trattiene molti blocchi vecchi e lo spazio cresce più in fretta. Per questo serve una **politica di conservazione**: una regola che stabilisce quanti snapshot tenere e per quanto tempo, perché il disco non si riempia senza che me ne accorga.

Btrfs offre anche un meccanismo per limitare lo spazio in modo più fine, chiamato **gruppo di quota** (quota group, o qgroup). È potente ma ha un costo in prestazioni, e Snapper lo usa in modo opzionale. Lo menziono per completezza: nel capitolo pratico vedremo se e quando conviene attivarlo.

---

## 7. Snapper: lo strumento che userò

Gestire gli snapshot a mano con i comandi nativi di btrfs è possibile e utile per capire il meccanismo, ma diventa presto scomodo: bisogna ricordarsi di crearli, nominarli, cancellarli e confrontarli. Per questo ho scelto **Snapper**, uno strumento che automatizza queste operazioni.

La scelta nasce dalla guida di Luca Giulianini, *Fedora 44: Guida Completa di Setup e Post-Installazione*, pubblicata ad aprile 2026, che è una delle fonti da cui parte questa repository. Nella sua guida Snapper è il cuore della gestione degli snapshot, affiancato da Btrfs Assistant come interfaccia grafica. Dal punto di vista teorico, Snapper si basa su alcuni concetti da presentare uno alla volta.

Il primo è la **configurazione**. Snapper ragiona per configurazioni, e ogni configurazione è associata a un subvolume. Nella guida ci sono due configurazioni, una per `root` (montato su `/`) e una per `home` (montato su `/home`). Ciascun subvolume ha così una politica propria: quanti snapshot conservare, con quale frequenza crearli, quando ripulirli. È il modo in cui Snapper applica in pratica la distinzione tra sistema e dati personali.

Il secondo è la **tipologia** di snapshot. Snapper distingue tre tipi principali. Gli snapshot **single** sono singoli, creati una volta, per esempio a mano. Gli snapshot **pre** e **post** vengono sempre creati in coppia attorno a un'operazione: il primo prima che inizi, il secondo dopo che è finita, e sono collegati tra loro. Gli snapshot **timeline** sono creati periodicamente, secondo una pianificazione, e servono a costruire una storia del sistema nel tempo.

Il terzo è la **pulizia**. Snapper applica algoritmi di sfoltimento che eliminano alcuni snapshot secondo regole precise. Un algoritmo elimina le coppie pre/post in cui l'operazione non ha cambiato nulla; un altro sfoltisce gli snapshot timeline tenendo più quelli recenti e meno quelli vecchi; un terzo limita il numero di snapshot per tipo. Sono questi algoritmi a tenere sotto controllo lo spazio di cui abbiamo parlato.

Il legame con i pacchetti è un altro punto della guida. Il gestore DNF5 può eseguire azioni prima e dopo una transazione, e questo permette di creare automaticamente una coppia pre/post attorno a ogni installazione, rimozione o aggiornamento. Concettualmente è semplice: prima della transazione si crea lo snapshot "pre", dopo lo "post", e il sistema registra il legame. Se un aggiornamento rompe qualcosa, si ha un punto preciso a cui tornare, identificato dall'operazione che lo ha causato. I dettagli dell'implementazione fanno parte del capitolo pratico.

Infine, la guida propone una **voce nel menu di avvio** per gli snapshot, attraverso grub-btrfs: il bootloader può mostrare gli snapshot come opzioni, così da partire da uno stato precedente anche se il sistema attuale non si avvia. La guida sottolinea che l'avvio da uno snapshot dal menu di GRUB è in sola lettura, e che il ripristino vero va fatto in un secondo momento dall'interfaccia di gestione. Nel mio sistema c'è un dettaglio da verificare: la riga del kernel contiene già `rootflags=subvol=root`, quindi una voce che punta a uno snapshot deve sostituire quel parametro. Non do per scontato che lo strumento lo faccia nel modo atteso, e lo verificheremo nel capitolo pratico.

Un'osservazione di metodo. La guida di Giulianini è scritta per un'altra macchina (una workstation con GPU NVIDIA e Intel) e per un ambiente GNOME, mentre io uso KDE Plasma. Le sue scelte sono un punto di partenza, non da applicare alla lettera: quello che descrive va verificato sul mio sistema, e le parti legate a GNOME vanno adattate. Lo stesso vale per i termini con cui la guida presenta alcune cose come "atomiche" o "enterprise": il comportamento reale va misurato, non dedotto dal nome.

---

## 8. Il ripristino: i concetti, non le procedure

Il ripristino è la ragione per cui si fanno gli snapshot, ma è anche la parte in cui si può fare più danno, quindi merita ordine.

Ci sono due situazioni diverse. Nella prima recupero un singolo file o una cartella: ho cancellato un documento o modificato un file di configurazione e voglio la versione precedente. Lo snapshot è una fonte di dati, da cui copio ciò che serve nell'originale. È un'operazione locale, che non tocca il resto del sistema e si annulla facilmente.

Nella seconda voglio riportare un intero subvolume di sistema allo stato di uno snapshot precedente. È un'operazione molto più incisiva, perché cambia insieme pacchetti, configurazioni e librerie. Qui serve capire che cosa decide quale ramo dell'albero viene avviato.

Btrfs ha un **subvolume predefinito**, quello che viene montato quando nessuno dichiara altro. Nel mio sistema il predefinito è la radice del filesystem, cioè ID 5, non `root`. Ma il sistema non si affida a questo default. Il bootloader passa al kernel il parametro `rootflags=subvol=root`, e `/etc/fstab` monta esplicitamente `/` con `subvol=root` e `/home` con `subvol=home`. Un montaggio con `subvol=` ha la precedenza sul default. Ne segue che cambiare il subvolume predefinito non cambierebbe nulla all'avvio: il sistema continuerebbe a montare `root`.

Quindi il ripristino completo di `root` non può basarsi sul solo default. Bisogna intervenire su ciò che decide il montaggio: la riga del kernel, il file fstab, oppure la struttura dei subvolumi, in modo che il ramo montato sia quello dello snapshot. Le strategie concrete e i rischi saranno nel capitolo pratico. Qui basta fissare il principio: viene avviato il ramo dichiarato più esplicitamente, e il default conta solo quando nessuno lo dichiara.

Il concetto da portarsi via è la distinzione tra le due situazioni. Recuperare un file è un'operazione sul contenuto. Ripristinare il sistema è un'operazione su quale stato viene montato. Confonderle porta a ripristinare più di quanto si volesse.

---

## 9. Che cosa non copre questo capitolo

Questo capitolo si limita ai concetti. Non contiene comandi per creare, elencare, cancellare o ripristinare snapshot, né la configurazione di Snapper, di Btrfs Assistant o di grub-btrfs. Quelle operazioni saranno nei capitoli pratici, che presupporranno i concetti qui esposti.

Restano fuori anche altri temi della manutenzione di btrfs, come lo **scrub** (la verifica dell'integrità tramite i checksum) e il **balance** (la redistribuzione dei blocchi), che hanno un ruolo importante ma sono un argomento a sé. Il backup vero, su disco esterno o in remoto, sarà trattato nella sezione dedicata, perché è un problema diverso da quello degli snapshot, come abbiamo visto all'inizio.

I concetti da avere chiari prima di passare alla parte pratica sono quattro. Il copy-on-write come base di tutto. Il subvolume come sottoalbero condiviso, non come partizione. Lo snapshot come subvolume che condivide i blocchi con l'originale. E la differenza tra recuperare un file e ripristinare il sistema, con la consapevolezza che il sistema viene avviato dal ramo che la configurazione dichiara.
