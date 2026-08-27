# Guida alla pre-installazione: da Windows 11 a Fedora 44 KDE Plasma

## Parte 1 — Dentro Windows 11: preparare il terreno

Pensa a questa fase come al "misura due volte, taglia una volta" dei falegnami. Quasi tutti i racconti di disastri che si leggono nei forum su installazioni Linux andate storte nascono da uno di questi passaggi saltato.

### 1. Backup

Trattalo come faresti con qualsiasi backup serio: sapere *dove* stanno i dati non basta, devi anche sapere che puoi *ripristinarli*, non solo che li hai "salvati da qualche parte". Se stai facendo un dual boot, il backup è la tua assicurazione contro un imprevisto; se stai cancellando Windows del tutto, è un passaggio obbligatorio, non facoltativo. Cosa mettere in salvo, in concreto:

- File personali (Documenti, Immagini, cartelle di progetto)
- Segnalibri e password del browser — oppure verifica che siano già sincronizzati su un account
- Dati di applicazioni che non si sincronizzano da soli sul cloud: salvataggi di videogiochi, chiavi SSH, il file `.ssh/config`, configurazioni VPN
- Licenze dei software a pagamento che pensi di reinstallare

### 2. Controlla lo stato di BitLocker e gestisci Microsoft Pluton

Molti Zenbook recenti con Ryzen AI 300 hanno la crittografia del disco attiva di default, legata a un modulo di sicurezza firmware (fTPM) integrato nel processore. Tecnicamente, il silicio dei Ryzen AI 300 include anche una licenza Microsoft chiamata Pluton, ma la sua attivazione effettiva dipende dal produttore: non è garantito che sia acceso sul tuo esemplare specifico, a differenza delle linee professionali AMD dove è più sistematico. In pratica, per te cambia poco: che sia Pluton "vero" o il classico fTPM AMD, il comportamento verso BitLocker è identico, e non serve capire quale dei due hai.

Apri PowerShell come Amministratore ed esegui:

```powershell
manage-bde -status
```

Se BitLocker risulta attivo, hai due strade:

- **Disattivalo del tutto** (la scelta più semplice se non ti serve la crittografia):
  ```powershell
  manage-bde -off C:
  ```
  La decrittazione richiede qualche minuto (anche venti o trenta su dischi molto pieni) e puoi continuare a usare il PC nel frattempo. Verifica che sia completata rilanciando lo stesso comando con `-status` prima di procedere oltre.
- **Mantieni la crittografia** (per esempio se fai dual boot e vuoi tenere dati sensibili protetti su Windows): in questo caso salva subito la chiave di ripristino a 48 cifre, scaricandola da `account.microsoft.com/devices/recoverykey` oppure annotandola su un supporto esterno **non collegato allo stesso PC**. Ti serve perché ridimensionare la partizione Windows mentre è cifrata, o modificare le voci di avvio, può far scattare una richiesta della chiave al riavvio successivo.

Se disattivare BitLocker ti sembra un dettaglio secondario, considera questo: è anche il modo più semplice per evitare che, più avanti, un cambio nello stato del Secure Boot ti chieda a sorpresa di reinserire la chiave di ripristino.

### 3. Controlla e sana il filesystem NTFS

Windows tiene un registro delle transazioni sul disco chiamato `$LogFile`. Se il sistema si spegne in modo brusco, entra in ibernazione ibrida, oppure viene interrotto durante una scrittura, il volume viene marcato come **"dirty"** (letteralmente "sporco"). Uno strumento Linux come `ntfsresize` (quello che useranno più avanti l'installer di Fedora) si rifiuta di lavorare su un volume dirty. È una protezione voluta, perché preferisce fermarsi piuttosto che scrivere su una struttura dati che potrebbe essere incoerente.

Verifica lo stato, sempre da PowerShell Amministratore:

```powershell
fsutil dirty query C:
```

Se il volume risulta dirty, esegui un controllo:

```powershell
chkdsk C: /f
```

Il controllo verrà pianificato al prossimo riavvio, perché il volume di sistema non può essere controllato "a caldo" mentre è in uso. Riavvia e lascia che Windows completi la scansione **prima** di toccare BIOS o partizioni: se salti questo passaggio, rischi di arrivare alla fase successiva con un filesystem che sembra sano ma non lo è davvero. Se dopo il primo `chkdsk /f` il volume risultasse ancora dirty al riavvio successivo esegui anche `chkdsk C: /scan` per una scansione online senza bisogno di riavviare, poi ripeti `/f` se necessario.

### 4. Disattiva completamente Ibernazione e Avvio rapido

L'Avvio rapido di Windows (Fast Startup) non è uno spegnimento vero: è un'ibernazione parziale, che congela il kernel su disco invece di chiuderlo del tutto. Questo lascia il filesystem NTFS in uno stato che gli strumenti Linux (e a volte lo stesso installer) si rifiutano di montare in scrittura, o segnalano come non sicuro da ridimensionare. È la causa numero uno dietro segnalazioni del tipo "la mia altra partizione sembra corrotta" che si leggono nei forum. Il comando che risolve entrambi i problemi in un colpo solo, da PowerShell Amministratore, è:

```powershell
powercfg /h off
```

Questo comando disattiva **contemporaneamente** l'ibernazione classica e l'Avvio rapido, perché condividono lo stesso meccanismo di fondo, ed elimina anche il file `hiberfil.sys` — qualche gigabyte di spazio recuperato come effetto collaterale gradito. Non serve quindi disattivare separatamente l'Avvio rapido dal Pannello di Controllo, come suggeriscono molte guide generiche: su un Windows 11 aggiornato questo unico comando è sufficiente.

### 5. Se riduci la partizione, preparati a un possibile ostacolo

Questo passaggio riguarda solo chi vuole fare dual boot; se stai cancellando Windows del tutto, saltalo e vai al punto 6.

Apri Gestione Disco (`diskmgmt.msc`), fai clic destro sulla partizione C: e scegli **Riduci volume**, ritagliando spazio non allocato per Fedora: **100 GB o più** è una misura comoda per un desktop KDE con un margine ragionevole. Lascia quello spazio **non allocato**, senza formattarlo: sarà l'installer di Fedora a occuparsene.

Qui però Gestione Disco spesso si rifiuta di ridurre il volume oltre un certo punto, anche con centinaia di gigabyte liberi sul disco. Il motivo non è quasi mai quello che si legge più spesso online (la Protezione di Sistema, cioè i punti di ripristino); è una causa possibile, ma non la principale. Il vero colpevole di solito sono alcuni **file non spostabili**: il file di paginazione (`pagefile.sys`), il file di ibernazione (che però hai già eliminato al punto 4) e le aree riservate alla copia shadow dei volumi. Questi file si trovano fisicamente sparsi sul disco, spesso verso la fine, e Gestione Disco non può "spingerli" per liberare spazio contiguo.

I tre interventi utili, in ordine di efficacia crescente rispetto allo sforzo richiesto:

1. **Riduci temporaneamente il file di paginazione**, invece di disattivare subito la Protezione di Sistema: vai in Impostazioni di sistema avanzate → Prestazioni → Impostazioni → Avanzate → Memoria virtuale, e imposta una dimensione fissa piccola (per esempio 2048 MB) al posto di "gestita automaticamente". Riavvia perché il cambiamento abbia effetto.
2. **Disattiva la Protezione di Sistema**, se il punto precedente non basta: Proprietà del sistema → Protezione sistema → Configura → Disattiva protezione sistema. Attenzione: questo elimina anche i punti di ripristino esistenti, quindi fallo solo *dopo* aver completato il backup del punto 1.
3. **Esegui una deframmentazione** del volume C: tramite Ottimizza unità → Ottimizza. Ha senso farlo anche su un SSD, non per le prestazioni ma per consolidare verso l'inizio del disco i metadati mobili che bloccano lo shrink; Windows riconosce che è un SSD e userà comunque TRIM, non un deframmentatore classico.

Dopo questi passaggi, riprova la riduzione in Gestione Disco: dovresti riuscire a liberare più spazio contiguo di quanto sembrasse possibile all'inizio.

### 6. Aggiorna Windows e il firmware

Prima di toccare qualsiasi impostazione di avvio, porta Windows Update fino in fondo, installando tutti gli aggiornamenti disponibili. Fatto questo, apri **MyASUS → Diagnosi di sistema / Live Update** e cerca aggiornamenti sia del **BIOS** sia, se elencato separatamente, del **firmware del controller NVMe/chipset**. Sui Ryzen AI 300 di prima generazione ci sono stati aggiornamenti BIOS importanti per la gestione energetica (`amd_pstate`) e per alcuni bug di risveglio dallo stato di sospensione: sono utili anche lato Linux, perché il firmware ACPI della scheda madre resta lo stesso sotto entrambi i sistemi operativi. Meglio farlo ora, con Windows ancora perfettamente funzionante: se qualcosa va storto in un aggiornamento firmware, avere già un sistema operativo funzionante a disposizione per il ripristino è molto più comodo.

### 7. Decidi in anticipo: dual boot o cancellazione completa?

Questa scelta determina cosa ti serve fare nei punti precedenti:

- **Cancellazione completa** → puoi saltare la riduzione della partizione (punto 5); concentrati solo sull'avere un backup solido.
- **Dual boot** → segui il punto 5 per ritagliare lo spazio non allocato, e tieni presente che dopo l'installazione di Fedora userai un boot manager (GRUB) che rileverà entrambi i sistemi operativi all'avvio.

### 8. Annota le particolarità del tuo modello specifico

Alcune funzioni della tastiera e dei sensori sui portatili sono gestite da software proprietario in Windows (MyASUS) e potrebbero comportarsi diversamente su Linux. Prima di passare a Fedora, prendi nota di cosa usi regolarmente: per esempio le combinazioni di tasti Fn per luminosità e volume, il sensore di impronte digitali, o le soglie di ricarica della batteria, così saprai cosa verificare dopo l'installazione. Su hardware AMD recente come questo, la maggior parte di queste funzioni è già coperta dai moduli del kernel Linux (`libinput`, `fprintd` per l'impronta digitale se presente, gestione energetica via `amd_pstate`); qualche funzione più specifica potrebbe invece richiedere una configurazione manuale o strumenti della comunità come `asusctl`, un argomento da affrontare comodamente dopo l'installazione, non ora.

---

## Parte 2 — Dentro il BIOS/UEFI: le impostazioni specifiche ASUS e AMD

Con Windows sistemato, riavvia il computer ed entra nel BIOS.

### 1. Come accedere al BIOS

La procedura ufficiale ASUS per questa generazione di Zenbook è questa: a PC **completamente spento**, tieni premuto il tasto **F2** e, mantenendolo premuto, premi anche il pulsante di accensione. Rilascia F2 solo quando compare la schermata del BIOS. È importante seguire l'ordine esatto: premere F2 *dopo* aver acceso il PC spesso non funziona sui sistemi moderni, perché l'avvio è talmente rapido che la finestra utile per intercettare il tasto è già chiusa. Il trucco è tenerlo premuto da prima ancora che il sistema si accenda.

Se preferisci un metodo che non dipende dal tempismo, puoi passare dal menu di Windows: **Impostazioni → Sistema → Ripristino → Avvio avanzato → Riavvia ora**, poi **Risoluzione dei problemi → Opzioni avanzate → Impostazioni firmware UEFI**. Windows riavvierà direttamente nel BIOS.

Una volta dentro, premi **F7** per passare dalla schermata semplificata "EZ Mode" alla **Advanced Mode**, dove si trovano tutte le voci descritte qui sotto.

### 2. Controller di storage: niente sorprese in stile RAID/RST

Su molti laptop con piattaforma Intel, questo è il passaggio più delicato dell'intera guida: se il controller storage è impostato su Intel RST (RAID) o Optane, l'installer di Fedora può non vedere affatto il disco, oppure vederlo in un modo che rischia di rompere l'avvio di Windows al riavvio successivo.

Sul tuo Zenbook questo problema semplicemente **non esiste**: la piattaforma è AMD, e l'SSD NVMe è gestito dal controller PCIe standard integrato nel System on Chip, non da un livello RAID/RST come sulle piattaforme Intel con Optane. Non troverai alcuna voce "Intel RST" da cercare nel BIOS, e non ti serve alcun intervento sul registro di Windows (il driver `storahci`, spesso citato nelle guide generiche per evitare la schermata blu al passaggio da RAID ad AHCI). Assicurati solo che l'unità NVMe risulti in modalità PCIe/NVMe standard, cosa che sui sistemi AMD è comunque il comportamento predefinito. È anche uno dei motivi per cui il dual boot su hardware AMD moderno è generalmente meno rischioso, dal punto di vista dello storage, rispetto a un portatile Intel con RST attivo.

### 3. Fast Boot lato BIOS

Attenzione a non confondere questa impostazione con quella disattivata al punto 4 della Parte 1: qui parliamo di una voce del firmware, non del sistema operativo. Alcune implementazioni del BIOS saltano l'enumerazione delle periferiche USB all'avvio per risparmiare uno o due secondi, il che può rendere la tua chiavetta di installazione invisibile nel menu di avvio. Nella scheda **Boot** del BIOS ASUS, imposta **Fast Boot** su **Disabled** per la finestra di tempo in cui farai l'installazione. Puoi riattivarlo in seguito, se ti interessa risparmiare quel secondo o due sui riavvii successivi.

### 4. Secure Boot: lascialo attivo

A differenza di alcune distribuzioni Linux, con Fedora non hai bisogno di disattivare il Secure Boot. Fedora firma il proprio bootloader (`shim`) con una chiave riconosciuta da Microsoft, quindi passa la verifica senza alcuna modifica da parte tua.

Vale la pena capire anche perché conviene lasciarlo attivo: su un sistema con fTPM o Pluton attivo che ha "legato" delle chiavi crittografiche allo stato del Secure Boot, disattivarlo e poi riattivarlo può in alcuni casi generare una richiesta di reinserimento della chiave di ripristino BitLocker: un motivo in più per preferire la disattivazione completa di BitLocker al punto 2 della Parte 1, piuttosto che tenerla attiva. Lascia il Secure Boot attivo a meno che in futuro tu non voglia installare moduli del kernel proprietari non firmati (per esempio certi percorsi del driver NVIDIA proprietario, tema comunque non rilevante su questo hardware con GPU AMD integrata, e che comunque i pacchetti RPM Fusion di Fedora gestiscono correttamente anche a Secure Boot attivo tramite `akmods` e l'iscrizione MOK, un argomento per il giorno dell'installazione, non per ora).

### 5. TPM: lascialo attivo

Non è richiesto da Fedora, ma non c'è motivo di disattivarlo, e alcuni sistemi diventano capricciosi se lo disattivi dopo che Windows ha già legato le chiavi di BitLocker al suo stato.

### 6. Conosci il tasto per il menu di avvio una tantum

Non serve modificare in modo permanente l'ordine di avvio: ti basta conoscere la scorciatoia per il menu di avvio "una tantum", che ti permette di scegliere la chiavetta USB senza toccare le impostazioni predefinite finché non sei sicuro che l'installazione funzioni.

Su questa generazione di Zenbook il tasto corretto è **Esc**, tenuto premuto nello stesso modo del punto 1: da PC spento, insieme al pulsante di accensione. Se per qualche motivo Esc non risponde, il metodo di riserva resta comunque il percorso guidato via Windows descritto al punto 1.

---

## Parte 3 — Creare la chiavetta di installazione

1. Scarica l'ISO ufficiale **Fedora KDE Plasma Desktop 44** dal sito Fedora.
2. Verifica il checksum: Fedora pubblica file `CHECKSUM` firmati con GPG. È una buona abitudine da avere ogni volta che si tocca il bootloader di un sistema, non un passaggio superfluo.
3. Scrivi l'immagine su una chiavetta USB da almeno 8 GB, scegliendo uno di questi due strumenti:
   - **Fedora Media Writer** (consigliato, strumento ufficiale): gestisce automaticamente il formato "ISO ibrida" di Fedora — un'immagine che può essere avviata sia come disco ottico sia come disco USB grezzo — e verifica la scrittura per te, senza bisogno di configurazioni.
   - **Rufus**, se preferisci questo strumento: scegli la **modalità DD** ("scrivi in modalità Immagine DD"), non la modalità ISO standard. Le immagini Fedora sono appunto ibride, e la modalità DD scrive byte per byte, preservando esattamente la struttura di partizionamento che l'installer Anaconda si aspetta di trovare. La modalità ISO standard di Rufus può in alcuni casi alterare la struttura EFI risultante, non sempre, ma non c'è motivo di rischiare quando la modalità DD funziona sempre per le immagini Fedora.

   Un dettaglio pratico da non scambiare per un errore: dopo aver scritto in modalità DD, la chiavetta smetterà di comparire come normalmente scrivibile in Esplora File di Windows. È previsto, non significa che la scrittura sia fallita.

---

## Checklist finale prima di avviare la USB

- [ ] Dati personali e configurazioni (SSH, VPN, licenze) copiati e, idealmente, verificati con un ripristino di prova
- [ ] BitLocker disattivato (`manage-bde -off C:` completato), oppure chiave di ripristino salvata fuori dal PC
- [ ] `fsutil dirty query C:` conferma un volume **non dirty** (eventualmente dopo `chkdsk C: /f` e un riavvio)
- [ ] Ibernazione e Avvio rapido disattivati con `powercfg /h off`
- [ ] Se dual boot: spazio non allocato ritagliato in Gestione Disco (100 GB+ consigliati), eventualmente dopo aver ridotto il pagefile o disattivato la Protezione di Sistema
- [ ] Windows aggiornato fino in fondo, e firmware/BIOS aggiornato via MyASUS
- [ ] BIOS: si accede con F2 tenuto premuto insieme all'accensione, poi F7 per la Advanced Mode
- [ ] BIOS: nessuna azione richiesta sul controller di storage (piattaforma AMD, nessun RAID/RST da gestire)
- [ ] BIOS: Fast Boot disattivato per la finestra di installazione
- [ ] Secure Boot lasciato attivo (nessuna azione necessaria)
- [ ] TPM lasciato attivo (nessuna azione necessaria)
- [ ] Tasto per il menu di avvio una tantum noto: **Esc**, tenuto premuto insieme all'accensione
- [ ] Chiavetta USB creata (Fedora Media Writer, oppure Rufus in modalità **DD**) e checksum verificato

---

## Cosa aspettarti dopo, in breve

Una volta superata questa lista, la parte difficile è finita: l'installer di Fedora (Anaconda) è pensato per essere semplice da qui in avanti, e in questa release il flusso è stato ulteriormente snellito: per esempio, l'installer non crea più automaticamente profili di rete per dispositivi non coinvolti nella configurazione iniziale, il che rende più semplice configurarli in seguito solo quando serve davvero. Al primo avvio dopo l'installazione, KDE Plasma ti guiderà con una procedura di configurazione dedicata (Plasma Setup).

Su un hardware AMD recente come questo Zenbook parti anche con qualche vantaggio pratico: nessun driver RAID da gestire, nessuna GPU NVIDIA proprietaria con cui armeggiare in combinazione con Secure Boot e `akmods`, e la scheda Wi-Fi MediaTek MT7922 è supportata nativamente dai kernel Linux recenti. L'unico componente che per ora resta sostanzialmente "silenzioso" sotto Linux è la NPU: il driver c'è, ma è presto per aspettarsi lo stesso livello di accelerazione AI che trovi su Windows.