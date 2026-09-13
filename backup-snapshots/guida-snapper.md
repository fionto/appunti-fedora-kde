# Snapshot Btrfs su Fedora 44 KDE Plasma — Snapper + dnf5 actions

Guida al setup di un sistema di snapshot automatici su Fedora 44 (KDE Plasma), filesystem Btrfs con root cifrata via LUKS. Copre due meccanismi complementari:

- **Snapper**: snapshot su timeline (orari/giornalieri/settimanali/...) + gestione/cleanup
- **dnf5 `actions` plugin**: snapshot automatico pre/post ad ogni transazione `dnf`

`/home` è volutamente **escluso** da questo setup: viene protetto separatamente con backup (Restic), non con snapshot locali.

---

## 1. Analisi del layout Btrfs esistente

Prima di installare qualsiasi cosa, è necessario capire come Fedora ha strutturato i subvolumi, perché Snapper si aspetta un layout specifico (diverso da quello di default di openSUSE, dove nasce).

```bash
sudo btrfs subvolume list /
```

Output tipico su Fedora Workstation:
```
ID 256 gen ... top level 5 path root
ID 257 gen ... top level 5 path home
```

`root` e `home` sono subvolumi **fratelli** (entrambi `top level 5`, cioè figli diretti del subvolume radice reale del filesystem, ID 5 — invisibile in questo elenco). Da non confondere: il subvolume *chiamato* `root` non è il subvolume radice del filesystem.

Verifica anche i mount point attivi:

```bash
findmnt -t btrfs
```

Su un sistema con root cifrata via LUKS, la sorgente sarà del tipo `/dev/mapper/luks-...`. Questo è trasparente per Snapper: lavora sopra il layer di decrittazione, nessuna configurazione aggiuntiva richiesta per la cifratura.

---

## 2. Installazione di Snapper

Verifica preliminare (evita installazioni/configurazioni duplicate):

```bash
rpm -q snapper dnf5-plugins
```

Installazione:

```bash
sudo dnf install snapper
```

---

## 3. Creazione della configurazione Snapper per `root`

```bash
sudo snapper -c root create-config /
```

Cosa fa:
1. Crea un **subvolume nidificato** `/.snapshots` dentro il subvolume `root` (non un subvolume fratello) — così gli snapshot restano logicamente legati al subvolume che proteggono.
2. Crea `/etc/snapper/configs/root` con i parametri di retention (default).

Verifica:

```bash
sudo snapper -c root list-configs
sudo btrfs subvolume list /
```

Nel secondo comando ci si aspetta una riga aggiuntiva con `top level 256` (nidificata dentro `root`, non fratella a `top level 5`):

```
ID 258 gen ... top level 256 path .snapshots
```

### Permessi su `.snapshots`

```bash
ls -la / | grep snapshots
```

Di default: `drwxr-x---`, proprietario `root:root` — solo root può accedervi. **Questo va bene così se si usa una GUI come Btrfs Assistant**, che eleva i privilegi via Polkit azione per azione (dialog di autenticazione), senza bisogno che l'utente normale abbia accesso diretto alla cartella. Aprire i permessi manualmente serve solo per uso diretto da CLI come utente non-root — non necessario in questo setup.

```bash
sudo dnf install btrfs-assistant
```

---

## 4. Tuning della retention (risparmio spazio)

I default di Snapper sono pensati per openSUSE e piuttosto generosi. Configurazione più snella, adatta a un laptop desktop dove il grosso della copertura "immediata" è comunque garantita dal plugin dnf5 (vedi sezione 5):

```bash
sudo snapper -c root set-config \
  "TIMELINE_LIMIT_HOURLY=0" \
  "TIMELINE_LIMIT_DAILY=5" \
  "TIMELINE_LIMIT_WEEKLY=2" \
  "TIMELINE_LIMIT_MONTHLY=3" \
  "TIMELINE_LIMIT_YEARLY=0" \
  "NUMBER_LIMIT=20" \
  "NUMBER_LIMIT_IMPORTANT=5" \
  "SPACE_LIMIT=0.3"
```

| Parametro | Valore | Significato |
|---|---|---|
| `TIMELINE_LIMIT_HOURLY` | 0 | Nessuna snapshot oraria di timeline (coperta comunque dagli snapshot pre/post di dnf) |
| `TIMELINE_LIMIT_DAILY` | 5 | Una settimana lavorativa di rollback giornaliero |
| `TIMELINE_LIMIT_WEEKLY` | 2 | Mezzo mese extra di margine |
| `TIMELINE_LIMIT_MONTHLY` | 3 | Rete lunga, tre mesi |
| `TIMELINE_LIMIT_YEARLY` | 0 | Non utile su un sistema desktop |
| `NUMBER_LIMIT` | 20 | Limite per snapshot non-timeline (es. quelli creati da dnf-actions) |
| `NUMBER_LIMIT_IMPORTANT` | 5 | Sottoinsieme "importanti" tra quelli sopra |
| `SPACE_LIMIT` | 0.3 | Tetto massimo: 30% dello spazio filesystem (default 50%) |

Verifica:

```bash
cat /etc/snapper/configs/root | grep -E "TIMELINE_LIMIT|NUMBER_LIMIT|SPACE_LIMIT"
```

> Nota: `set-config` è il modo corretto per modificare la configurazione — valida i valori, a differenza di un editing diretto del file.

---

## 5. Attivazione dei timer systemd

Fino a qui abbiamo configurato *quante* snapshot tenere, ma non il meccanismo che le *crea* periodicamente.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now snapper-timeline.timer snapper-cleanup.timer
```

- `snapper-timeline.timer` — crea snapshot di timeline secondo i `TIMELINE_LIMIT_*`
- `snapper-cleanup.timer` — esegue il cleanup periodico secondo `NUMBER_LIMIT`/`TIMELINE_LIMIT_*`

`daemon-reload` è necessario se systemd segnala che i file unit sono cambiati su disco (es. dopo installazione di pacchetti correlati) ma non ancora ricaricati in memoria.

Verifica:

```bash
systemctl status snapper-timeline.timer snapper-cleanup.timer
```

Stato atteso: `enabled` + `active (waiting)` per entrambi (per un timer, "waiting" è lo stato normale tra un trigger e l'altro).

---

## 6. Perché `/home` non è incluso

1. **Scopo diverso del rollback**: gli snapshot su `root` servono a recuperare da un aggiornamento di sistema andato male (kernel, driver, pacchetti). Includere `home` nello stesso rollback rischierebbe di far perdere modifiche recenti ai file personali solo per riparare un problema di sistema.
2. **Volume di dati**: `home` cambia molto più velocemente di `root` (download, cache applicazioni, progetti), quindi snapshottarlo con la stessa frequenza consuma spazio molto più rapidamente anche con la condivisione dei blocchi (copy-on-write).
3. **Protezione più adatta**: uno snapshot vive sullo stesso disco fisico — non protegge da guasto hardware. Per i dati personali la protezione corretta è un backup su storage separato (Restic verso NAS), non uno snapshot locale.

---

## 7. Plugin `dnf5 actions` — snapshot automatici su transazioni pacchetti

Su Fedora, con il passaggio a DNF5, il vecchio plugin `python3-dnf-plugin-snapper` non è più disponibile. Il sostituto è il plugin generico **`actions`** (incluso in `dnf5-plugins`), che permette di agganciare comandi shell arbitrari agli hook delle transazioni.

### Formato del file di configurazione

I file vanno in `/etc/dnf/libdnf5-plugins/actions.d/*.actions`. Ogni riga (non commento) ha 5 campi separati da `:`:

```
callback_name:package_filter:direction:options:command
```

Per intercettare *qualsiasi* transazione, i campi filtro restano vuoti (`::::`).

### File di configurazione

```bash
sudo tee /etc/dnf/libdnf5-plugins/actions.d/snapper.actions > /dev/null << 'EOF'
# Cattura il comando che ha scatenato la transazione, per usarlo come descrizione
pre_transaction::::/usr/bin/sh -c echo\ "tmp.cmd=$(ps\ -o\ command\ --no-headers\ -p\ '${pid}')"

# Crea lo snapshot "pre" e salva il suo numero in una variabile temporanea
pre_transaction::::/usr/bin/sh -c echo\ "tmp.snapper_pre_number=$(snapper\ create\ -t\ pre\ -c\ number\ -p\ -d\ '${tmp.cmd}')"

# Se esiste la variabile, crea lo snapshot "post" collegato e ripulisce le variabili temporanee
post_transaction::::/usr/bin/sh -c [\ -n\ "${tmp.snapper_pre_number}"\ ]\ &&\ snapper\ create\ -t\ post\ --pre-number\ "${tmp.snapper_pre_number}"\ -c\ number\ -d\ "${tmp.cmd}"\ ;\ echo\ tmp.snapper_pre_number\ ;\ echo\ tmp.cmd
EOF
```

Punti chiave:
- **`sudo tee ... << 'EOF'`**: necessario perché `sudo comando > file` fallirebbe — il redirect `>` verrebbe valutato dalla shell dell'utente normale *prima* che `sudo` entri in gioco, senza permessi sul file di root. Gli apici singoli attorno a `'EOF'` disattivano l'espansione delle variabili durante l'incollaggio, lasciando `$(...)` letterale nel file.
- **`-c number`**: assegna gli snapshot creati dal plugin all'algoritmo di cleanup `number`, cioè li fa contare/ripulire secondo `NUMBER_LIMIT`/`NUMBER_LIMIT_IMPORTANT`. **Senza questo parametro gli snapshot da dnf non vengono mai ripuliti automaticamente**, accumulandosi indefinitamente.
- **`tmp.cmd`**: usato come descrizione dello snapshot (es. `dnf install cowsay`), utile per orientarsi in `snapper list` a distanza di tempo invece di vedere righe generiche identiche.

### Verifica

```bash
cat /etc/dnf/libdnf5-plugins/actions.d/snapper.actions
```

---

## Riepilogo stato del setup

- [x] Layout subvolumi verificato (`root` + `home`, fratelli a top level 5)
- [x] Snapper installato e configurato su `root`
- [x] Subvolume `.snapshots` creato (nidificato in `root`)
- [x] Permessi `.snapshots` lasciati root-only (coerente con uso via Btrfs Assistant/Polkit)
- [x] Btrfs Assistant installato (GUI)
- [x] Retention personalizzata per risparmio spazio
- [x] Timer systemd (`timeline` + `cleanup`) attivi
- [x] `/home` escluso deliberatamente (coperto da Restic)
- [x] Plugin dnf5 `actions` — file creato, test pre/post in corso di verifica

## Prossimi passi possibili (non ancora affrontati)

- Integrazione `grub-btrfs` per rollback selezionabile direttamente dal menu di boot GRUB