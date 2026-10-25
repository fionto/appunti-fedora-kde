Lavorando con Snapper, due cose da sapere subito:

- Tutti i comandi di lettura e manutenzione richiedono `sudo`, perché gli snapshot appartengono a root e il file di configurazione è in `/etc/snapper/configs/`.
- Il nome `root` è quello della configurazione che hai creato, cioè il subvolume `/`. Lo ripeto in ogni comando con `-c root`. Puoi vedere tutte le configurazioni con il comando 1 qui sotto.

## 1. Lettura: capire cosa c'è

**Elenco delle configurazioni presenti:**

```bash
sudo snapper list-configs
```

Mi aspetto una sola riga, `root`, con il percorso `/`. Se compare altro, lo annoto.

**Elenco degli snapshot:**

```bash
sudo snapper -c root list
```

Ogni riga è uno snapshot. Le colonne principali sono:

- `#`: il numero identificativo, che serve per tutti gli altri comandi.
- `Type`: `single` per uno snapshot singolo, `pre` e `post` per la coppia creata attorno a una transazione DNF.
- `Pre #`: nelle righe `post`, indica il numero dello snapshot `pre` corrispondente.
- `Date` e `Description`: quando e perché è stato creato. Per gli snapshot DNF la descrizione contiene il comando.
- `Cleanup`: l'algoritmo che lo gestirà alla prossima pulizia (`number`, `timeline`, `empty-pre-post`). Se è vuoto, lo snapshot non verrà cancellato automaticamente.

Se l'output è largo, puoi scegliere le colonne:

```bash
sudo snapper -c root list --columns number,type,date,description,cleanup
```

**Dettagli della configurazione:**

```bash
sudo snapper -c root get-config
```

Qui ritrovi i valori che hai impostato (`TIMELINE_LIMIT_DAILY`, `NUMBER_LIMIT`, `SPACE_LIMIT` e così via). È la forma più rapida per verificare che il file non sia cambiato.

## 2. Lettura: cosa è cambiato tra due snapshot

Questa è la funzione più utile per capire cosa ha fatto una transazione.

**Lista dei file modificati:**

```bash
sudo snapper -c root status 42..43
```

Sostituisci `42` e `43` con il numero del `pre` e del `post`. Ogni riga ha una lettera che indica il tipo di modifica: `+` file aggiunto, `-` file rimosso, `c` contenuto cambiato, `t` tipo cambiato, `.` solo attributi cambiati.

**Differenze dettagliate, file per file:**

```bash
sudo snapper -c root diff 42..43 /usr/bin
```

Il secondo argomento limita il confronto a una cartella o a un file. Senza di esso l'output può essere enorme, quindi conviene sempre restringerlo.

**Confronto con lo stato attuale (senza numero):**

```bash
sudo snapper -c root status 42..0
```

Il numero `0` indica il filesystem attivo. È utile per vedere tutto ciò che è cambiato dopo uno snapshot, per esempio prima di decidere se ripristinare un file.

## 3. Lettura: stato del filesystem Btrfs

Snapper vive sopra Btrfs, quindi per capire lo spazio reale occupato dagli snapshot serve guardare anche il filesystem.

```bash
sudo btrfs subvolume list /
```

Elenca i subvolumi. Dovresti vedere `root` (ID 256) e `home` (ID 257), più i subvolumi di `.snapshots` creati da Snapper.

```bash
sudo btrfs filesystem usage /
```

Mostra lo spazio totale, usato e libero, e quanto è allocato. Il dato che mi interessa è `Free (estimated)`.

```bash
sudo btrfs qgroup show /
```

Serve a leggere l'occupazione per subvolume. Attenzione: funziona solo se le quote (qgroup) sono attive. Se il comando risponde con un errore tipo "quotas not enabled", significa che non lo sono. In quel caso `SPACE_LIMIT` nella configurazione di Snapper non ha effetto, e questo è un dettaglio da annotare nel capitolo.

## 4. Controllo: i timer che fanno il lavoro da soli

Snapper crea e pulisce gli snapshot tramite timer di systemd. Verificare che siano attivi evita sorprese.

```bash
systemctl list-timers | grep -i snapper
```

Mi aspetto `snapper-timeline.timer` (crea snapshot orari o giornalieri) e `snapper-cleanup.timer` (applica i limiti di retention). Se uno dei due manca, gli snapshot timeline non vengono creati né cancellati.

```bash
systemctl status snapper-timeline.timer snapper-cleanup.timer
```

Ti dice se sono `active` e quando è prevista la prossima esecuzione.

## 5. Manutenzione: creare e cancellare

**Snapshot manuale prima di un'operazione rischiosa:**

```bash
sudo snapper -c root create --description "prima di sperimentare X"
```

Usa una descrizione che ti ricordi a cosa serviva: tra un mese sarà l'unico indizio. Il comando stampa il numero creato.

**Pulizia secondo le regole configurate:**

```bash
sudo snapper -c root cleanup number
sudo snapper -c root cleanup timeline
sudo snapper -c root cleanup empty-pre-post
```

Ogni comando applica un algoritmo specifico:

- `number`: tiene al massimo `NUMBER_LIMIT` snapshot (con il tuo valore, 20).
- `timeline`: applica i limiti daily, weekly e monthly.
- `empty-pre-post`: rimuove le coppie pre/post in cui non è cambiato nulla.

Snapper non ha una modalità di simulazione per questi comandi. Quindi prima guardo con `list` quali snapshot rientrerebbero nei criteri, e solo dopo lancio la pulizia.

**Cancellazione di un singolo snapshot:**

```bash
sudo snapper -c root delete 42
```

Questa operazione è **irreversibile**. Prima di lanciarla controllo sempre con `list` e `status` cosa rappresenta lo snapshot. Non cancello mai la coppia pre/post di una transazione che non so spiegare.

## 6. Manutenzione Btrfs

Snapper dipende dalla salute del filesystem, quindi anche questi comandi rientrano nella manutenzione.

```bash
sudo btrfs scrub start /
sudo btrfs scrub status /
```

Lo *scrub* legge tutti i dati e verifica i checksum, segnalando eventuali corruzioni. Su un SSD di un portatile può durare un po', quindi lo avvio senza aspettarlo in primo piano. `status` mostra lo stato e gli errori trovati. Fedora di norma lo pianifica già con un timer: lo verifico prima di lanciarlo a mano.

```bash
systemctl list-timers | grep -i scrub
```

## Riferimento rapido

| Obiettivo | Comando |
|---|---|
| Elenco configurazioni | `sudo snapper list-configs` |
| Elenco snapshot | `sudo snapper -c root list` |
| Configurazione attiva | `sudo snapper -c root get-config` |
| File cambiati in una transazione | `sudo snapper -c root status PRE..POST` |
| Differenze dettagliate su una cartella | `sudo snapper -c root diff PRE..POST /percorso` |
| Creare snapshot manuale | `sudo snapper -c root create --description "..."` |
| Pulizia secondo i limiti | `sudo snapper -c root cleanup number` (o `timeline`, `empty-pre-post`) |
| Cancellare uno snapshot | `sudo snapper -c root delete N` |
| Subvolumi | `sudo btrfs subvolume list /` |
| Spazio del filesystem | `sudo btrfs filesystem usage /` |
| Occupazione per subvolume | `sudo btrfs qgroup show /` |
| Timer Snapper | `systemctl list-timers \| grep -i snapper` |
| Verifica dati Btrfs | `sudo btrfs scrub status /` |

Un suggerimento per la routine: ogni tanto, con il comando `list`, controlla che non si accumulino snapshot senza descrizione e senza cleanup. È il segnale che qualcosa nei timer o nei limiti non funziona come pensi.

Quando vuoi, possiamo passare al prossimo passo, cioè ripristinare un singolo file da uno snapshot. È un'operazione che si fa in sicurezza solo dopo aver letto questi comandi, quindi la lascerei per ultima.