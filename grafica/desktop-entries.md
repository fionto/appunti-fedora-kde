# Lanciatore per Filen su Fedora KDE

Su Fedora KDE Plasma 44 uso **Filen** come cloud, tramite l'app Filen Desktop che monta il mio spazio remoto in `~/filen`. Volevo un'icona sul desktop che aprisse quella cartella. La soluzione più semplice (un collegamento) ha un difetto: se l'app è chiusa, il mount non c'è e l'icona apre una cartella locale vuota, senza avvisare.

Questo capitolo raccoglie come sono arrivato ad una soluzione che funziona nel mio caso: cosa sono davvero un symlink e una *desktop entry*, che cosa c'è dietro `~/filen`, e il percorso passo per passo, con una verifica prima di ogni modifica.

⚠️ **Stato di verifica.** I comandi dei passi 1-8 li ho eseguiti sul mio sistema e riporto gli esiti reali. Le parti segnate come *proposta* o *non verificato* (sezioni 7 e 8) no.

---

## 1. Symlink e desktop entry: due cose diverse

Sul desktop due icone possono sembrare uguali ma essere fatte in modo molto diverso.

**Symlink.** È un collegamento a livello di **filesystem**: una voce che contiene solo un percorso verso un altro oggetto. Quando la apro, il sistema segue il percorso e lavora direttamente sull'originale.

```bash
ln -s /home/matteo/filen ~/Scrivania/filen
```

Nome e icona sono ereditati dal target, non ci sono metadati propri e, se il target viene spostato, il link si rompe.

**Desktop entry.** È un file di testo con estensione `.desktop` che descrive *come presentare e come avviare* qualcosa: nome, icona, comando da eseguire. Non contiene l'oggetto da aprire, contiene una **descrizione**, che l'ambiente desktop (qui Plasma) legge e trasforma in un'icona.

Un'analogia: il symlink è una scorciatoia che porta dritto alla porta di casa; la desktop entry è un biglietto da visita che dice "questa si chiama così, ha questa icona, e per entrarci devi fare X". Quella "X" può includere dei controlli, ed è esattamente quello che mi serve.

| | Symlink | Desktop entry |
|---|---|---|
| Livello | Filesystem | Ambiente desktop |
| Contenuto | Un percorso | Nome, icona, tipo, azione |
| Icona e nome | Ereditati dal target | Scelti da me |
| Può eseguire uno script | No | Sì, con `Type=Application` |
| Può controllare lo stato di ciò che apre | No | Sì, se lo script lo fa |
| Funziona fuori dal desktop (terminale, script) | Sì | No, la legge solo il desktop |

---

## 2. Cos'è freedesktop.org e cosa dice sulle desktop entry

Il formato `.desktop` non l'ha inventato KDE: è uno **standard condiviso**, pubblicato come *Desktop Entry Specification* da **freedesktop.org**. freedesktop.org è il progetto che ospita le specifiche con cui i vari ambienti desktop Linux (KDE, GNOME e altri) si mettono d'accordo per interoperare, in modo che un'applicazione installata una volta compaia correttamente in tutti. L'introduzione della specifica lo dice in modo esplicito: KDE e GNOME avevano adottato un formato simile, e lo standard unifica le due strade per semplificare l'interoperabilità.

Quello che la specifica fissa, in sintesi (ho controllato la pagina ufficiale il 23 ottobre 2026, dove la versione più recente che ho trovato è la **1.5, del 27 aprile 2020**):

- **Estensione:** i file hanno estensione `.desktop` (`.directory` per il tipo `Directory`).
- **Codifica:** UTF-8.
- **Struttura:** un file in stile INI. Deve esserci un gruppo con intestazione `[Desktop Entry]`, e prima non c'è nulla se non, al limite, dei commenti.
- **Valori localizzati:** una chiave si può ripetere con un suffisso di lingua, ad esempio `Name[it_IT]=Filen`. Il sistema sceglie la variante in base alla lingua della sessione.
- **Icone:** il valore di `Icon=` può essere un **percorso assoluto** oppure un **nome simbolico** (come `filen`), che viene cercato nei temi icone secondo l'*Icon Theme Specification*, un'altra specifica freedesktop.
- **Chiavi riconosciute:** `Type`, `Name`, `Exec`, `Icon`, `Terminal`, `URL`, `TryExec` e molte altre.

I tre valori di `Type=` che mi interessano (li elenco dalla mia conoscenza della specifica, senza averli riletti uno per uno):

| `Type=` | Cosa fa |
|---|---|
| `Application` | Esegue il comando di `Exec=`. |
| `Link` | Apre un URL (chiave `URL=`) con l'applicazione predefinita. Non esegue comandi. |
| `Directory` | Descrive una cartella (nome, icona) nei menu. |

### Una nota su KDE e le estensioni

Nel file che Dolphin aveva riscritto compariva `URL[$e]=file:$HOME/filen`. Per quanto ho capito, `[$e]` non è parte della sintassi base della specifica: è una convenzione di KDE per marcare un valore in cui vanno **espanse le variabili d'ambiente** (qui `$HOME`) quando il file viene letto. Non l'ho verificato sulla documentazione di KDE, quindi la considero un'ipotesi ragionevole, non un fatto. La specifica ha comunque un'appendice dedicata alle chiavi riservate all'uso in KDE.

Conseguenza pratica: **la sintassi `[$e]` non vale ovunque**. Nei file che scrivo io a mano uso percorsi assoluti.

---

## 3. Il campo `Exec` e il comando `grep` per leggerlo

Uno dei passaggi della procedura è leggere **come l'app Filen si avvia**, invece di indovinarlo. L'app installa un proprio file `.desktop` di sistema, e lì c'è la risposta:

```bash
grep -E '^(Exec|TryExec)=' /usr/share/applications/Filen.desktop
```

Pezzo per pezzo:

- `grep` stampa le righe di un file che corrispondono a un criterio.
- `-E` attiva le **espressioni regolari estese**, che permettono di usare `( | )` per l'alternativa senza scrivere backslash.
- `'^(Exec|TryExec)='` è il criterio. `^` significa "inizio riga", quindi non catturo per sbaglio righe come `Name=...Exec...` o commenti. `(Exec|TryExec)` significa "`Exec` oppure `TryExec`", e il `=` finale evita di prendere chiavi con nomi simili (per esempio `X-Exec-qualcosa`). Le virgolette singole impediscono alla shell di interpretare `^`, `(`, `|`.
- `/usr/share/applications/` è la cartella di sistema dove i pacchetti mettono le loro desktop entry. Per le entry dell'utente c'è una cartella analoga in `~/.local/share/applications/`.

### Cosa sono `Exec` e `TryExec`

| Chiave | Significato |
|---|---|
| `Exec=` | Il comando da eseguire per avviare l'applicazione, con eventuali argomenti. |
| `TryExec=` | Percorso di un eseguibile da controllare *prima* di mostrare la voce: se non esiste o non è eseguibile, la voce viene ignorata. Serve a nascondere le app non più installate. È facoltativa. |

### Il mio risultato

```text
Exec=/opt/Filen/Filen %U
```

Due informazioni:

1. **L'app è in `/opt/Filen/`**, quindi fuori dal database RPM: non la gestisce `dnf`, e gli aggiornamenti dipendono da come la distribuisce Filen. Non ho `TryExec`, e nessun problema: la chiave è facoltativa.
2. **`%U` è un *field code***, un segnaposto che l'ambiente desktop sostituisce al lancio. Per quanto ricordo dalla specifica, `%U` diventa l'elenco degli URL passati all'avvio (uno per argomento), e se non c'è nulla da passare sparisce. Nel mio caso lancio l'app senza argomenti, quindi `%U` non produce niente.

Per approfondire i field code basta cercare la sezione "The Exec key" nella specifica.

---

## 4. Cosa c'è dietro `~/filen`

Prima di automatizzare qualcosa ho verificato che cosa fosse davvero quella cartella.

```bash
mountpoint /home/matteo/filen
findmnt /home/matteo/filen
```

Esiti sul mio sistema:

```text
/home/matteo/filen is a mountpoint

TARGET             SOURCE FSTYPE      OPTIONS
/home/matteo/filen Filen  fuse.rclone rw,nosuid,nodev,relatime,user_id=1000,group_id=1000
```

Come si legge:

- **`is a mountpoint`**: la cartella è la radice di un filesystem montato, non una directory qualsiasi.
- **`fuse.rclone`**: il filesystem è implementato in *user space* tramite FUSE (prefisso `fuse.`), e il programma che fa da ponte è rclone (sottotipo).
- **`Filen` come `SOURCE`**: non è un device come `/dev/sda1`, è il nome del *remote* rclone.
- **`user_id=1000,group_id=1000`**: il mount appartiene al mio utente. `rw` indica lettura e scrittura.

### Ma il pacchetto rclone non c'è

Su questo sistema rclone **non è installato** con `dnf`. Per capire chi gestiva il mount ho guardato il processo:

```bash
ps aux | grep -i '[r]clone'
```

Il comando (accorciato) mostrava questo:

```text
/home/matteo/.config/@filen/desktop/rclone/bin/1.74.3/rclone mount Filen: /home/matteo/filen/
  --config .../rclone.conf --vfs-cache-mode full --cache-dir .../networkDrive/cache
  --vfs-cache-max-size 32Gi --vfs-write-back 5s ... --log-file .../logs/rclone-drive.log
```

Il binario è quello **incluso nell'app Filen Desktop**, in `~/.config/@filen/desktop/rclone/`. Quindi il mount lo gestisce l'app, non io. Nota: la parte `[r]clone` nel `grep` è un trucco classico, la parentesi quadra impedisce al `grep` di trovare se stesso nell'elenco dei processi.

Cose da annotare, lette dalla riga di comando:

- **Cache locale:** `--vfs-cache-mode full`, fino a 32 GiB, in `~/.config/@filen/desktop/networkDrive/cache`. I file aperti vengono copiati in locale.
- **Write-back:** `--vfs-write-back 5s`, le modifiche partono verso Filen dopo circa 5 secondi.
- **Log:** `~/.config/@filen/logs/rclone-drive.log`.
- **Processo di monitoraggio:** c'è anche uno script `monitor.v3.drive.sh` che riceve due PID. Il secondo è rclone; il primo suppongo sia il processo principale dell'app, ma non l'ho verificato.

### La conseguenza che conta

Il mount esiste **solo finché l'app è in esecuzione**. Con l'app chiusa, `~/filen` torna una cartella locale normale. Un collegamento semplice la aprirebbe vuota, senza segnalare niente, e per un attimo sembrerebbe che i file siano spariti. Peggio: uno script che scrivesse lì dentro scriverebbe nel disco locale, non su Filen.

Per questo la prova affidabile è `mountpoint`. Un `ls -ld ~/filen` non basta: mostra permessi e data della directory *come li vede il sistema in quel momento*, quindi con il mount attivo sono quelli del filesystem remoto.

---

## 5. Il percorso, passo per passo

### Passo 0 — Primo tentativo: `Type=Link`

```bash
cat > "$(xdg-user-dir DESKTOP)/Filen.desktop" <<'EOF'
[Desktop Entry]
Type=Link
Name=Filen
URL=file:///home/matteo/filen
EOF
chmod +x "$(xdg-user-dir DESKTOP)/Filen.desktop"
```

- `$(xdg-user-dir DESKTOP)` chiede al sistema il percorso della cartella Desktop. Sul mio sistema restituisce `/home/matteo/Scrivania`, non `~/Desktop`. È un esempio di command substitution, già vista in `primi-passi.md`.
- `<<'EOF'` è un **heredoc**: passa un blocco di testo a `cat`. Le virgolette singole su `EOF` dicono a bash di non espandere eventuali `$` al suo interno.

Dolphin poi ha riscritto il file con `URL[$e]=file:$HOME/filen` e `Icon=filendesktop`. È normale (vedi sezione 2). **Limite:** `Type=Link` non esegue niente, quindi non può controllare il mount.

### Passo 1 — Backup e stato attuale

```bash
cp ~/Scrivania/Filen.desktop ~/Scrivania/Filen.desktop.bak
ls -l ~/Scrivania/Filen.desktop
mountpoint ~/filen
```

**Atteso e ottenuto:** il file ha il bit di esecuzione (`-rwxr-xr-x`) e `~/filen` è un mountpoint (l'app era aperta). Il punto dopo i permessi (`-rwxr-xr-x.`) indica solo la presenza di un contesto SELinux.

Il backup è il primo passo perché tutto quel che segue riscrive il file.

### Passo 2 — Leggere il comando di avvio dell'app

Il `grep` della sezione 3, più il controllo dei programmi che userà lo script:

```bash
grep -E '^(Exec|TryExec)=' /usr/share/applications/Filen.desktop
command -v gio dolphin kdialog
```

**Ottenuto:** `Exec=/opt/Filen/Filen %U`, e i percorsi di `/usr/bin/gio`, `/usr/bin/dolphin`, `/usr/bin/kdialog`. `command -v` stampa il percorso di un comando se esiste, quindi tre percorsi significano tre programmi presenti. Nessuna installazione necessaria.

### Passo 3 — Icona e tema

```bash
kreadconfig6 --file kdeglobals --group Icons --key Theme
ls ~/.local/share/icons/Papirus/*/apps/filen*
```

**Ottenuto:** il tema attivo è `Papirus`, e `filen.svg` esiste in tutte le dimensioni, da 16 a 128 px (più le varianti `@2x`). C'è anche `filendesktop.svg`, che è quello che Dolphin aveva scritto nel file. Nel lanciatore finale uso `Icon=filen`.

Nota: all'inizio avevo ipotizzato che `Icon=Filen` (maiuscola, che trova un PNG a 48 px in `hicolor`) e `Icon=filen` (che trova l'SVG di Papirus) dessero risultati diversi per via della distinzione maiuscole/minuscole. È una spiegazione plausibile ma **non l'ho verificata**.

### Passo 4 — Lo script

```bash
mkdir -p ~/.local/bin
cat > ~/.local/bin/apri-filen.sh <<'EOF'
#!/bin/bash
MOUNT="$HOME/filen"

if ! mountpoint -q "$MOUNT"; then
    # avvia l'app Filen usando il suo desktop file di sistema
    gio launch /usr/share/applications/Filen.desktop
    # attendi fino a 60 secondi che il mount compaia
    for _ in $(seq 1 60); do
        mountpoint -q "$MOUNT" && break
        sleep 1
    done
fi

if mountpoint -q "$MOUNT"; then
    exec dolphin "$MOUNT"
else
    kdialog --error "Filen non risulta montato in $MOUNT"
    exit 1
fi
EOF
chmod +x ~/.local/bin/apri-filen.sh
bash -n ~/.local/bin/apri-filen.sh && echo "sintassi OK"
```

Cosa fa, e perché:

- `mountpoint -q` non stampa niente e comunica l'esito col **codice di uscita** (0 = è un mountpoint). Per questo si può usare direttamente dentro un `if`.
- `gio launch` avvia un'applicazione a partire dal suo file `.desktop`, invece di chiamare l'eseguibile a mano. L'ho scelto perché passa dal meccanismo di avvio del desktop. Non ho verificato che sia indispensabile.
- Il ciclo `for` con `sleep 1` attende fino a 60 secondi; `break` esce non appena il mount compare.
- `exec dolphin ...` **sostituisce** il processo dello script con Dolphin, quindi lo script non resta in attesa.
- `kdialog --error` mostra una finestra di errore se il mount non arriva.
- `~/.local/bin/` è una cartella per gli script dell'utente. Su Fedora normalmente è già nel `PATH`, ma qui non serve, perché nel lanciatore uso il percorso assoluto.
- `bash -n` controlla solo la **sintassi**, senza eseguire nulla.

**Ottenuto:** `sintassi OK`.

### Passo 5 — Test con il mount attivo

```bash
~/.local/bin/apri-filen.sh
```

**Ottenuto:** Dolphin si è aperto su `~/filen`, senza riavviare l'app. Un controllo successivo con `ps aux | grep '[d]olphin'` ha senso solo a finestra aperta: l'ho lanciato a finestra chiusa e ovviamente non ha trovato nulla.

### Passo 6 — Test con il mount assente

Questo è il test delicato, perché richiede di chiudere l'app. Prima ho aspettato che il write-back di rclone fosse sicuramente finito (la configurazione dice 5 secondi, ma conviene lasciare un margine più largo), senza modificare file in `~/filen`. Poi ho chiuso Filen dal suo menu, **non** con `kill`.

```bash
mountpoint ~/filen; echo "exit=$?"
ps aux | grep -i '[f]ilen' | grep -v grep
```

**Ottenuto:**

```text
/home/matteo/filen is not a mountpoint
exit=32
```

e nessun processo Filen. ⚠️ **Correzione:** nelle istruzioni avevo scritto che mi aspettavo `exit=1`, ma `mountpoint` restituisce **32** per "non è un mountpoint". Quello che conta è che il valore sia diverso da zero.

Poi ho lanciato lo script con l'app chiusa. **Ottenuto:** l'app si è avviata, il mount è comparso entro il tempo previsto, Dolphin si è aperto su `~/filen`. Un nuovo `mountpoint` ha dato `is a mountpoint` ed `exit=0`.

Nel terminale sono comparsi anche questi messaggi:

```text
ERROR:ui/ozone/platform/wayland/host/wayland_wp_color_manager.cc:277] Unable to set image transfer function.
ERROR:base/nix/mime_util_xdg.cc:139] Invalid mime.cache file does not contain null prior to ALIAS_LIST_OFFSET=44
```

Vengono da **Chromium/Electron**, il motore su cui gira Filen Desktop: li stampa l'app, e finiscono nel terminale perché ho lanciato lo script da lì. Le righe `wayland_wp_color_manager` riguardano la gestione dei colori su Wayland. L'ultima segnala un file `mime.cache` malformato, ma non dice quale; per cercarlo si può guardare `ls -l /usr/share/mime/mime.cache ~/.local/share/mime/mime.cache`. Non l'ho indagato: non impedisce il funzionamento.

### Passo 7 — Riscrivere il lanciatore come `Type=Application`

Backup della versione `Link`, poi riscrittura **intera** (meglio che modificare con `sed` un file già riformattato da Dolphin):

```bash
cp ~/Scrivania/Filen.desktop ~/Scrivania/Filen.desktop.link.bak
cat > ~/Scrivania/Filen.desktop <<'EOF'
[Desktop Entry]
Type=Application
Name=Filen
Name[it_IT]=Filen
Icon=filen
Exec=/home/matteo/.local/bin/apri-filen.sh
Terminal=false
EOF
chmod +x ~/Scrivania/Filen.desktop
desktop-file-validate ~/Scrivania/Filen.desktop && echo "desktop file OK"
```

Note:

- `Exec=` richiede il **percorso assoluto**: non uso `~` né `$HOME`, perché non sono sicuro che vengano espansi in quel campo.
- `Terminal=false` evita che si apra una finestra di terminale quando faccio doppio clic.
- `chmod +x` è necessario per `Type=Application`, altrimenti Plasma può rifiutarsi di avviarlo.
- `desktop-file-validate` (pacchetto `desktop-file-utils`) controlla il file contro la specifica. Silenzioso = valido.

**Ottenuto:** `desktop file OK`.

### Passo 8 — Verifica finale

Ho chiuso Filen dal suo menu, ho controllato che il mount fosse sparito e ho fatto doppio clic sull'icona. **Ottenuto:** l'app si avvia, il mount compare, Dolphin si apre su `~/filen`. Con Plasma, alla prima esecuzione, può comparire una richiesta di conferma per un file `.desktop` eseguibile: è il comportamento previsto.

---

## 6. Come tornare indietro

```bash
mv ~/Scrivania/Filen.desktop.link.bak ~/Scrivania/Filen.desktop
```

Per rimuovere tutto:

```bash
rm ~/Scrivania/Filen.desktop ~/Scrivania/Filen.desktop.bak ~/Scrivania/Filen.desktop.link.bak
rm ~/.local/bin/apri-filen.sh
```

Lo script e il lanciatore non toccano il mount né l'app: rimuoverli non cambia il comportamento di Filen Desktop.

---

## 7. Miglioramenti (proposte, non ancora testate)

La versione attuale funziona nel mio caso. Questi sono i punti che migliorerei, nella variante di script da provare **prima** di sostituire quella attuale.

- **Evitare una seconda istanza.** Se l'app è già in avvio ma il mount non è ancora pronto, lo script attuale ne lancia un'altra. Un controllo con `pgrep` prima di `gio launch` evita il problema.
- **Un log.** Dal doppio clic non vedo l'output: un file di log mi dice cosa è successo.
- **Un errore che rimanda al log.** Il messaggio di `kdialog` può indicare dove guardare.

```bash
#!/bin/bash
# apri-filen.sh: apre ~/filen, avviando Filen Desktop se il mount non c'è

MOUNT="$HOME/filen"
APP_DESKTOP="/usr/share/applications/Filen.desktop"
TIMEOUT=60
LOG="${XDG_STATE_HOME:-$HOME/.local/state}/apri-filen.log"

mkdir -p "$(dirname "$LOG")"
log() { echo "$(date '+%F %T') $*" >> "$LOG"; }

if mountpoint -q "$MOUNT"; then
    log "mount già attivo"
else
    if pgrep -f '/opt/Filen/Filen' >/dev/null; then
        log "app già in esecuzione, attendo il mount"
    else
        log "avvio dell'app"
        gio launch "$APP_DESKTOP" >>"$LOG" 2>&1
    fi
    for _ in $(seq 1 "$TIMEOUT"); do
        mountpoint -q "$MOUNT" && break
        sleep 1
    done
fi

if mountpoint -q "$MOUNT"; then
    log "mount ok, apro Dolphin"
    exec dolphin "$MOUNT"
else
    log "mount assente dopo ${TIMEOUT}s"
    kdialog --error "Filen non risulta montato in $MOUNT (log: $LOG)"
    exit 1
fi
```

Cosa c'è di nuovo, per chi non ha già familiarità:

- `"${XDG_STATE_HOME:-$HOME/.local/state}"` usa la variabile `XDG_STATE_HOME` se è definita, altrimenti il valore dopo `:-`. È un valore predefinito in stile bash.
- `log() { ...; }` definisce una **funzione**; `$*` sono tutti i suoi argomenti.
- `>> "$LOG"` aggiunge in fondo al file; `2>&1` manda nello stesso posto anche gli errori.
- `pgrep -f` cerca tra i processi per riga di comando completa.

**Alternativa a monte:** far partire Filen Desktop al login (*Impostazioni di sistema → Avvio automatico*), così il mount è già pronto e il lanciatore potrebbe restare un semplice `Type=Link`. Il compromesso è che l'app gira sempre. Non l'ho provata.

**Il timeout.** Il valore di 60 secondi l'ho scelto a occhio. Quanto impiega davvero il mount a comparire non l'ho misurato: lo farei prima di tenerlo così.

---

## 8. Cosa non ho verificato

- Il motivo della differenza tra `Icon=Filen` e `Icon=filen` (maiuscole/minuscole).
- Se `Type=Link` richieda il bit di esecuzione per essere avviato da Plasma: l'ho citato ma non l'ho testato.
- Che `[$e]` sia una convenzione KDE e non della specifica base (lo deduco dalla struttura, non l'ho trovato scritto).
- Che i messaggi di Chromium non compaiano quando avvio il lanciatore dal desktop: li ho visti solo lanciando lo script da terminale.
- Che `pgrep -f '/opt/Filen/Filen'` riconosca bene l'app (e i suoi processi figli).
- Il ruolo del primo PID passato a `monitor.v3.drive.sh`.
- I `Type=` della specifica e il significato dei field code `%U`: scritti dalla mia conoscenza della specifica, non riletti punto per punto.

---

## 9. Riepilogo operativo

- [ ] Capire cosa c'è dietro la cartella: `mountpoint`, `findmnt`, `ps aux | grep '[r]clone'`
- [ ] Fare il backup del lanciatore esistente
- [ ] Leggere `Exec`/`TryExec` con `grep -E '^(Exec|TryExec)=' /usr/share/applications/Filen.desktop`
- [ ] Verificare che `gio`, `dolphin`, `kdialog` esistano (`command -v`)
- [ ] Controllare il tema icone e la presenza di `filen.svg`
- [ ] Scrivere lo script in `~/.local/bin/` e controllarlo con `bash -n`
- [ ] Testare con il mount attivo (deve aprire Dolphin)
- [ ] Chiudere l'app, attendere, verificare con `mountpoint` (codice di uscita diverso da 0)
- [ ] Testare con il mount assente (deve avviare l'app e aprire Dolphin)
- [ ] Riscrivere il lanciatore come `Type=Application` e validarlo con `desktop-file-validate`
- [ ] Doppio clic dal desktop, a app chiusa
- [ ] Valutare i miglioramenti della sezione 7 e testarli prima di sostituire lo script
