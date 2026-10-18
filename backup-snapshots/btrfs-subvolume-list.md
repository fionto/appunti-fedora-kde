# Leggere la lista dei subvolumi: `sudo btrfs subvolume list /`

Questo capitolo prende un comando che useremo spesso e ne spiega l'output riga per riga, usando il mio sistema come esempio. Il comando non modifica nulla: legge la struttura del filesystem e la stampa. I concetti di base (subvolume, snapshot, copy-on-write) sono nel capitolo introduttivo, qui li rendiamo concreti.

Il comando va lanciato con `sudo`. Senza privilegi btrfs può rifiutarsi di mostrare la struttura completa o mostrarne solo una parte.

Ecco l'output del mio sistema, abbreviato nelle righe centrali:

```text
ID 256 gen 7616 top level 5 path root
ID 257 gen 7616 top level 5 path home
ID 258 gen 7565 top level 256 path .snapshots
ID 267 gen 1182 top level 258 path .snapshots/9/snapshot
ID 308 gen 3237 top level 258 path .snapshots/45/snapshot
...
ID 390 gen 7448 top level 258 path .snapshots/118/snapshot
ID 391 gen 7565 top level 258 path .snapshots/119/snapshot
```

Ogni riga descrive un subvolume. Di base il comando mostra quattro campi: ID, generazione (gen), genitore (top level) e percorso (path). Il significato di ciascuno va chiarito con attenzione, perché alcuni nomi sono fuorvianti.

---

## 1. Il filesystem come albero di subvolumi

Il filesystem btrfs è organizzato come un albero. In cima c'è un subvolume speciale, la radice, con ID 5. Ogni altro subvolume è figlio di qualcun altro.

Nell'output, `root` e `home` hanno entrambi `top level 5`: sono figli diretti della radice. `.snapshots` ha `top level 256`, cioè è figlio di `root`. Gli snapshot di Snapper hanno `top level 258`, cioè sono figli di `.snapshots`.

Il nome "top level" è quindi fuorviante. Non indica "il livello più alto", ma **l'ID del genitore**. Solo il valore 5 corrisponde alla radice del filesystem. Quando leggi una riga, la domanda giusta è: "di quale subvolume è figlio?". La risposta è nel numero dopo `top level`.

Questa struttura spiega anche `findmnt`. Montare un subvolume significa rendere visibile un ramo dell'albero in un punto preciso del sistema: il ramo `root` diventa `/`, il ramo `home` diventa `/home`. Gli snapshot annidati sotto `root` esistono nell'albero, ma non sono montati da nessuna parte: il sistema non legge i file da loro.

---

## 2. L'ID

L'ID è un numero intero che btrfs assegna a ogni subvolume alla creazione. È l'identità reale del subvolume nel filesystem: il nome (`root`, `home`) è un'etichetta per noi, mentre il sistema ragiona con l'ID.

Gli ID sono assegnati da un unico contatore crescente per tutto il filesystem. Non esiste un contatore separato per tipo di subvolume: `root` ha 256, `home` ha 257, `.snapshots` ha 258, e il primo snapshot elencato ha 267. Ogni subvolume creato, di qualunque tipo, fa avanzare il contatore.

Ne deriva che gli ID non sono consecutivi. Tra 258 e 267 mancano nove numeri, e tra 267 e 308 ne mancano quaranta. Quei numeri corrispondono a subvolumi creati e poi cancellati: Snapper ne elimina molti con la pulizia periodica, e quei numeri non tornano. Un buco nella sequenza è la traccia di qualcosa che è esistito e non esiste più.

Gli ID 256 e 257 hanno un significato in più: sono i subvolumi montati all'avvio come `/` e `/home`, come vedremo nella sezione sul subvolume predefinito del capitolo introduttivo.

---

## 3. Il path

Il campo `path` indica il nome del subvolume, espresso come percorso **relativo al genitore** indicato da `top level`. Non è relativo al punto di montaggio nel sistema.

Per questo `.snapshots/9/snapshot` appare con `top level 258`: il percorso parte da `.snapshots`. Snapper crea per ogni snapshot una cartella numerata dentro `.snapshots`, e dentro quella cartella mette il subvolume chiamato `snapshot`. Accanto ad esso salva un file `info.xml` con i metadati dello snapshot (data, tipo, descrizione). Il file `info.xml` non è un subvolume, quindi non compare nella lista.

Il comando `show`, che vedremo più avanti, usa invece un percorso **assoluto**, cioè a partire dalla radice del filesystem. Lo stesso snapshot 119 viene indicato come `root/.snapshots/119/snapshot`. Il percorso è lo stesso, cambia solo il punto di partenza: il genitore nel primo caso, la radice nel secondo.

Il path ci permette anche di capire perché `home` non ha snapshot. Se Snapper avesse creato snapshot per `home`, questi sarebbero annidati dentro `home`, in un `.snapshots` con `top level 257`. Nell'output non compare nessuna riga con `top level 257` oltre a `home` stesso. Questo è coerente con la scelta che ho fatto: `home` non è configurato in Snapper, quindi non ha snapshot btrfs. I dati di `home` saranno protetti in futuro da un altro strumento, come backup restic.

---

## 4. Generazione, ultima modifica e creazione

Nell'output compare `gen`, la generazione del subvolume. La generazione è il contatore che avanza a ogni transazione di scrittura del filesystem, come abbiamo visto nel capitolo introduttivo.

Il campo `gen` indica la generazione dell'**ultima modifica** del subvolume. Se un file dentro `root` cambia, la generazione di `root` si aggiorna alla transazione in cui è avvenuto il cambiamento. Per questo `root` e `home` hanno entrambi `gen 7616`: sono stati modificati nella stessa transazione recente, cosa normale in un sistema in uso, dove log, cache e documenti cambiano in continuazione.

Accanto a `gen` esiste la **ogeneration**, cioè la generazione di origine, che indica quando il subvolume è stato **creato**. Nell'help del comando è richiesta con l'opzione `-c`. Attenzione a un punto: nei capitoli precedenti l'avevo chiamata `cgen`, ma questo nome non esiste nel comando. Il termine corretto è ogeneration. Il comando `show`, che vedremo, la indica come `Gen at creation`.

La differenza tra le due è la differenza tra "quando è nato" e "quando è stato toccato l'ultima volta". Ci si aspetterebbe che per uno snapshot in sola lettura, mai modificato dopo la creazione, le due coincidano. Il mio caso dimostra che non è sempre così. Lo snapshot 119 ha questi valori:

```text
Generation:             7565
Gen at creation:        7564
```

La generazione di creazione è 7564, mentre l'ultima modifica registrata è 7565. Lo snapshot è stato creato in una transazione e il suo record è stato aggiornato in quella successiva. Non ho verificato la causa precisa: potrebbe dipendere dal modo in cui btrfs registra la creazione di un subvolume, oppure da un'operazione di Snapper avvenuta subito dopo. Quello che si può affermare con sicurezza è che `gen` e `ogen` possono differire anche per uno snapshot che non sembra essere stato modificato.

Un'osservazione utile riguarda `.snapshots`. Il subvolume `.snapshots` (ID 258) e lo snapshot 119 hanno entrambi `gen 7565`. Creare uno snapshot significa aggiungere una voce dentro `.snapshots`, e questo modifica il genitore nella stessa transazione. Quindi la generazione di `.snapshots` si aggiorna all'ultimo snapshot creato. È un esempio di come `gen` registri anche le modifiche avvenute dentro un subvolume figlio.

Il confronto tra generazioni dà infine un'informazione pratica: uno snapshot con `gen` molto più basso di quella attuale del filesystem è fermo da tempo. Lo snapshot 9 ha `gen 1182`, mentre il filesystem è a 7616. È proprio il comportamento atteso per un punto di ritorno. Il valore diventa interessante quando si vuole capire quali subvolumi cambiano più spesso.

---

## 5. Il collegamento tra ID btrfs e numeri di Snapper

Nell'output convivono due numerazioni. Quella di btrfs è l'ID, globale e crescente, che nel mio caso va da 256 a 391. Quella di Snapper è il numero della cartella dentro `.snapshots`, che va da 9 a 119.

Le due numerazioni sono collegate dal percorso, ma non coincidono. Snapper numera gli snapshot della propria configurazione in sequenza, e il numero ha senso solo dentro Snapper. Btrfs assegna un ID a ogni subvolume creato nel filesystem, anche quelli che Snapper non vede. Lo snapshot 9 di Snapper corrisponde all'ID 267 di btrfs, ma la corrispondenza è stabilita dal percorso `.snapshots/9/snapshot`, non da un calcolo.

Il numero di Snapper non ha alcun legame con l'ID btrfs. Se cancellassi lo snapshot 45, la sua cartella sparirebbe e il subvolume 308 non esisterebbe più. Lo snapshot 46, se esistesse, manterrebbe il suo numero e il suo ID.

Esiste poi un collegamento più profondo, che non passa dal nome ma dall'identità del subvolume. Ogni subvolume ha un **UUID**, un identificativo univoco. Uno snapshot ha anche un **parent UUID**, che punta all'UUID del subvolume da cui è stato creato. È il meccanismo con cui btrfs sa che uno snapshot discende da un certo subvolume, anche quando il nome cambia. Per lo snapshot 119 il `Parent UUID` è `d542a623-00a1-f147-a4e2-6643d1041e1f`, mentre il `Parent ID` è 258, cioè `.snapshots`. Sono due cose diverse: il parent ID è il genitore nell'albero, il parent UUID è la sorgente da cui lo snapshot è stato copiato. Per verificare che la sorgente sia `root`, si può confrontare quel UUID con quello di `root`, cosa che non ho ancora fatto.

---

## 6. Opzioni utili

Il comando accetta diverse opzioni. Riporto quelle indicate dall'help del mio sistema, raggruppate per funzione.

Per filtrare per percorso, `-o` mostra solo i subvolumi sotto il percorso indicato, mentre `-a` mostra tutti i subvolumi del filesystem distinguendo percorsi assoluti e relativi rispetto al percorso dato. Senza filtri il comando elenca tutto, quindi `-o` è utile quando vuoi concentrarti su una parte dell'albero.

Per scegliere i campi da stampare, `-p` aggiunge l'ID del genitore, `-c` la ogeneration (generazione di creazione), `-g` la generazione di ultima modifica, `-u` l'UUID del subvolume, `-q` il parent UUID degli snapshot e `-R` l'UUID dei subvolumi ricevuti con `btrfs send` e `receive`, un caso che nel mio sistema non ricorre.

Per filtrare per tipo, `-s` mostra solo gli snapshot, `-r` solo i subvolumi in sola lettura (snapshot compresi), e `-d` elenca i subvolumi cancellati che non sono ancora stati ripuliti dal filesystem. Quest'ultima è utile per capire perché un'eliminazione di snapshot non libera lo spazio subito.

Per la formattazione, `-t` stampa il risultato come tabella.

Per ordinare e filtrare per generazione, `-G` e `-C` accettano un valore preceduto da `+` (maggiore o uguale), `-` (minore o uguale) o senza segno (uguale). Ad esempio, `-G -1000` mostra i subvolumi modificati per l'ultima volta a una generazione non superiore a 1000. L'opzione `--sort` ordina per `gen`, `ogen`, `rootid` o `path`, con `+` o `-` per crescente o decrescente.

Le opzioni cambiano tra versioni di `btrfs-progs`. La lista qui sopra corrisponde all'help del mio sistema, e conviene sempre rileggerla con `btrfs subvolume list --help` prima di usarla in uno script.

---

## 7. Comandi correlati per approfondire

Il comando `list` dà una panoramica. Per capire un singolo subvolume serve `show`, e per il subvolume predefinito serve `get-default`. Sono tutti comandi di sola lettura.

### `btrfs subvolume show`

Il comando `sudo btrfs subvolume show <percorso>` mostra le informazioni dettagliate di un subvolume. Sul mio sistema, per lo snapshot 119, restituisce questi campi:

```text
root/.snapshots/119/snapshot
        Name:                   snapshot
        UUID:                   ca244564-bf17-7841-ae87-a66b89a37ebf
        Parent UUID:            d542a623-00a1-f147-a4e2-6643d1041e1f
        Received UUID:          -
        Creation time:          2026-10-18 16:00:15 +0200
        Subvolume ID:           391
        Generation:             7565
        Gen at creation:        7564
        Parent ID:              258
        Top level ID:           258
        Flags:                  readonly
        Send transid:           0
        Send time:              2026-10-18 16:00:15 +0200
        Receive transid:        0
        Receive time:           -
        Snapshot(s):
        Quota group:            n/a
```

Alcuni campi li conosciamo già: `Subvolume ID` è l'ID, `Generation` la generazione di ultima modifica, `Gen at creation` la ogeneration, `Parent ID` e `Top level ID` il genitore, `Parent UUID` la sorgente. `Flags: readonly` indica che lo snapshot è in sola lettura. `Snapshot(s)` è vuoto perché lo snapshot 119 non ha snapshot figli. `Quota group: n/a` significa che la quota non è attiva su questo filesystem. I campi `Send` e `Receive` riguardano l'invio di snapshot con `btrfs send`, che non stiamo usando, quindi restano a zero o vuoti.

### `btrfs subvolume get-default`

Il comando `sudo btrfs subvolume get-default /` mostra quale subvolume è il predefinito. Nel mio sistema restituisce:

```text
ID 5 (FS_TREE)
```

Il subvolume predefinito è la radice del filesystem, cioè ID 5, e non `root`. Questo è il valore iniziale di btrfs: nessuno lo ha cambiato. Conta solo quando il sistema non indica esplicitamente quale subvolume montare, cosa che nel mio caso non succede, come vedremo nella sezione finale.

### Comandi non ancora provati

Esistono altri due comandi utili, che non ho ancora verificato sul mio sistema. `sudo btrfs inspect-internal subvolid-resolve <ID> /` restituisce il percorso di un subvolume dato il suo ID, ed è utile quando un log cita solo un numero. `sudo btrfs inspect-internal rootid <percorso>` fa l'operazione opposta. Li inserirò nel capitolo quando li avrò provati.

### Comandi di Snapper

`snapper list` mostra gli snapshot dal punto di vista di Snapper, con numero, tipo, data e descrizione. È il complemento di `btrfs subvolume list`: il primo racconta la storia degli snapshot, il secondo la struttura del filesystem.

Non sono inclusi i comandi per creare, eliminare o impostare il subvolume predefinito. Modificano lo stato del sistema e appartengono ai capitoli pratici.

---

## 8. Cosa il comando non dice

`btrfs subvolume list` descrive la struttura, ma non dice tutto. Non mostra quanto spazio occupa ogni subvolume: per quello servono `btrfs filesystem du` e, se configurati, i gruppi di quota. Non dice quali subvolumi sono montati e dove: per questo si usa `findmnt`. E non distingue gli snapshot creati da Snapper da quelli creati in altri modi, perché la lista non conserva questa informazione. Per ricostruirla si combina `snapper list` con il percorso.

Il comando è un punto di partenza per leggere la struttura. Per capire lo stato del sistema va combinato con gli altri strumenti, ognuno dei quali risponde a una domanda diversa.
