# Il comando `sudo btrfs subvolume list /`

Il comando:

```bash
sudo btrfs subvolume list /
```

serve a visualizzare l’elenco dei sottovolumi Btrfs presenti nel filesystem montato a partire dal percorso `/` (la radice del sistema Linux). Per comprenderne il funzionamento, è necessario conoscere brevemente che cosa sia Btrfs e che cosa si intenda per subvolume.

## 1. Che cos'è Btrfs?

Btrfs (B-tree File System) è un filesystem moderno per Linux, progettato per offrire funzionalità avanzate, tra cui:

* Snapshot, cioè copie istantanee dello stato di un filesystem.
* Sottovolumi (subvolume).
* Gestione avanzata dello spazio su disco.
* Controllo dell'integrità dei dati tramite checksum.
* Compressione trasparente.
* Gestione dei dati tramite meccanismi di tipo copy-on-write.

A differenza di filesystem tradizionali come ext4, Btrfs permette di organizzare un singolo filesystem in più sottovolumi indipendenti dal punto di vista della gestione.

### 1.1 Che cos'è un sottovolume?

Un sottovolume Btrfs è una porzione logicamente separata di un filesystem Btrfs. Può essere paragonato, in modo semplificato, a una directory con caratteristiche speciali: contiene file e sottodirectory, ma può anche essere gestito autonomamente, per esempio tramite snapshot o backup. Immaginiamo un disco con un filesystem Btrfs che contiene:

```
Filesystem Btrfs
│
├── Subvolume principale
│   ├── etc/
│   ├── home/
│   ├── usr/
│   └── var/
│
├── Subvolume home
│   ├── alice/
│   └── bob/
│
└── Subvolume snapshot
    └── ...
```

Questa rappresentazione è concettuale: i sottovolumi possono essere montati in diversi percorsi e non devono necessariamente corrispondere a directory visibili nella struttura del sistema.

## 2. Analisi del comando

Il comando è composto da tre elementi principali:

```bash
sudo    btrfs    subvolume list    /
```

### `btrfs`

È il programma a riga di comando che permette di amministrare e interrogare i filesystem Btrfs.

### `subvolume list`

È il sottocomando che richiede a Btrfs di elencare i sottovolumi individuati nel filesystem di riferimento.

### `/`

È il percorso utilizzato come punto di riferimento per individuare il filesystem Btrfs da interrogare. Non significa necessariamente che verranno elencati tutti i filesystem Btrfs presenti nel computer.

## 3. Esempio di output

Su un sistema Linux che utilizza Btrfs, l'output potrebbe essere simile al seguente:

```
ID 256 gen 120 top level 5 path @
ID 257 gen 120 top level 5 path @home
ID 258 gen 115 top level 5 path @snapshots
ID 259 gen 110 top level 256 path @/var/lib/containers
```

Questi dati sono esemplificativi: i valori reali dipendono dalla struttura del filesystem e dalle operazioni effettuate su di esso. Vediamo il significato delle colonne.

### 3.1 `ID`

Esempio:

```
ID 256
```

È l'identificatore numerico del sottovolume all'interno del filesystem Btrfs. Ogni sottovolume possiede un proprio ID, utile per identificarlo in operazioni amministrative, come la creazione di snapshot o la sua eliminazione.

### 3.2 `gen`

Esempio:

```
gen 120
```

Indica il generation number, cioè il numero di generazione dei metadati associato al sottovolume nel momento in cui vengono mostrate le informazioni. Il valore è utilizzato internamente da Btrfs per tenere traccia dell'evoluzione dei metadati. Non rappresenta direttamente il numero di modifiche ai file né l'età del sottovolume.

### 3.3 `top level`

Esempio:

```
top level 5
```

Indica l'ID del sottovolume o del livello superiore nella gerarchia dei sottovolumi Btrfs. Il valore `5` è comunemente associato al top-level subvolume, cioè il livello superiore speciale del filesystem Btrfs. Questa informazione è utile per comprendere le relazioni gerarchiche tra i sottovolumi.

### 3.4 `path`

Esempio:

```
path @home
```

Indica il percorso del sottovolume rispetto al riferimento utilizzato da Btrfs per la sua struttura interna. Per esempio:

```
path @
path @home
path @snapshots
```

Su molte distribuzioni Linux, nomi come `@` e `@home` sono convenzioni utilizzate per organizzare rispettivamente il sottovolume principale e quello destinato alle directory personali. Attenzione: il valore di `path` non è necessariamente un percorso assoluto che puoi utilizzare direttamente nella shell. Il percorso di montaggio effettivo dipende dalla configurazione del sistema.

## 4. A che cosa serve nella pratica?

Il comando è particolarmente utile per l'amministrazione di sistemi Linux che utilizzano Btrfs.

1. Ispezionare la struttura del filesystem: Permette di capire quali sottovolumi sono presenti, quali ID possiedono e come sono organizzati.

2. Individuare gli snapshot: Gli snapshot Btrfs sono anch'essi sottovolumi. L'elenco può quindi aiutare a individuare snapshot creati manualmente o da strumenti di gestione del sistema.

3. Preparare operazioni amministrative: Gli ID e i percorsi mostrati possono essere utilizzati come riferimento per comandi successivi, ad esempio per ottenere informazioni dettagliate, creare snapshot o eliminare sottovolumi.

4. Diagnosticare la configurazione di Linux: È utile quando si vuole capire come una distribuzione abbia organizzato il filesystem, soprattutto se utilizza sottovolumi separati per `/home`, snapshot o directory di sistema.

## 5. Comandi correlati

Per approfondire la gestione dei sottovolumi, esistono altri comandi della stessa famiglia:


```bash
# Mostra informazioni dettagliate su un sottovolume
sudo btrfs subvolume show /

# Crea un nuovo sottovolume
sudo btrfs subvolume create /percorso/nuovo_subvolume

# Crea uno snapshot
sudo btrfs subvolume snapshot /origine /destinazione

# Elimina un sottovolume
sudo btrfs subvolume delete /percorso/subvolume
```

Gli ultimi tre comandi possono modificare il filesystem: devono quindi essere utilizzati con attenzione, soprattutto quando coinvolgono dati importanti.