# Post-installazione — Flatpak, Flathub e Flatseal

Su Fedora KDE Plasma 44 ci sono due modi principali per installare un'applicazione grafica: un pacchetto **RPM** tramite `dnf5`, oppure un **Flatpak**. Il risultato visibile (un'icona nel menu, una finestra che si apre) è lo stesso, ma sotto il cofano sono due modelli diversi, e capire la differenza serve a scegliere con criterio caso per caso invece di installare "da dove capita".

⚠️ **Stato di verifica.** I comandi di questo capitolo sono ricavati dalla documentazione di Flatpak e dalla logica dei comandi, non tutti eseguiti da me su questo sistema. Dove scrivo "mi aspetto" o "da verificare", è una previsione da confrontare con l'output reale.

---

## 1. Flatpak e pacchetto `dnf`: che differenza c'è

Un'analogia: un pacchetto RPM è come un mobile che si monta **dentro casa tua**, usando le prese, i tubi e gli attrezzi che la casa già ha. Un Flatpak è come una **roulotte parcheggiata nel giardino**: porta con sé quello che le serve e comunica con la casa solo attraverso porte che decidi tu.

| | Pacchetto RPM (`dnf5`) | Flatpak |
|---|---|---|
| **Dove installa** | Nel sistema (`/usr`, `/etc`...), mescolato ai file degli altri pacchetti | In una zona separata: `/var/lib/flatpak` (a livello di sistema) o `~/.local/share/flatpak` (solo per il mio utente) |
| **Dipendenze** | Librerie **condivise con tutto il sistema**; DNF risolve il grafo e garantisce che tutto sia coerente | L'app dichiara un **runtime** (es. `org.kde.Platform`, `org.freedesktop.Platform`), un insieme di librerie che si scarica a parte e che più app possono condividere |
| **Isolamento** | Nessuno: il programma gira con i permessi del mio utente e vede tutta la mia home | **Sandbox**: l'app vede solo ciò che i suoi permessi dichiarano; il resto passa dai *portal* (vedi sotto) |
| **Dati dell'app** | Sparsi (`~/.config`, `~/.local/share`...) | Raccolti in `~/.var/app/<ID-applicazione>/` |
| **Ciclo di aggiornamento** | Legato ai repository Fedora/RPM Fusion e alla versione della distro | Indipendente dalla versione di Fedora: l'app si aggiorna quando lo pubblica chi la mantiene |
| **Chi costruisce il pacchetto** | I packager Fedora (o RPM Fusion/COPR) | Per Flathub, di solito chi mantiene l'app o la comunità attorno ad essa; per il repo Fedora, Fedora stessa a partire dai suoi RPM |
| **Permessi per installare** | `sudo` sempre | Dipende dall'installazione scelta: *system* (richiede autorizzazione) o *user* (nessun `sudo`) |
| **Costo** | Poco spazio, un'unica copia di ogni libreria | Più spazio su disco (runtime + app), anche se i runtime sono deduplicati tra le app |

Due conseguenze pratiche che mi interessano:

1. **Il Flatpak non usa le librerie di sistema.** Questo vuol dire che tutto ciò che ho fatto *sul sistema* non vale automaticamente per le app Flatpak: lo `swap` di `ffmpeg` in `codec-multimediali.md` e `mesa-va-drivers-freeworld` in `driver-grafici-accelerazione-video.md` modificano i pacchetti RPM, mentre un'app Flatpak usa i codec e i driver grafici forniti dal **proprio runtime** (a quanto so, tramite estensioni del runtime; da verificare come si comporta il mio caso). È lo stesso avvertimento che avevo già segnato per Firefox in Flatpak nel capitolo sulla grafica.
2. **Il sandbox non è un'etichetta, è una configurazione.** Un'app Flatpak con permesso di accedere a tutta la home (`--filesystem=home`) è isolata molto meno di quanto il nome faccia pensare. Per questo la parte più utile di questo capitolo è quella sui permessi (sezioni 5 e 6).

### Come scelgo: la regola che mi do

- **Applicazioni grafiche di cui non conosco bene la provenienza**, o che voglio tenere più isolate dal resto: Flatpak è una buona scelta, proprio per il sandbox.
- **Strumenti di base**, tutto ciò che deve integrarsi a fondo con la shell e col sistema (compilatori, strumenti da riga di comando, linguaggi, driver, servizi di sistema): pacchetto `dnf`. L'integrazione col filesystem è più diretta e non devo ragionare sui permessi del sandbox.
- **La stessa app non la tengo due volte** (RPM e Flatpak insieme): confonde i profili, i file associati e il menu applicazioni.

È una regola pragmatica per il mio uso, non un principio universale: ci sono casi in cui l'RPM è meglio anche per un'app grafica, e viceversa.

---

## 2. Cosa c'è già su Fedora e come verificarlo

Fedora include Flatpak nell'installazione e configura un proprio repository di Flatpak, chiamato `fedora`, costruito a partire dai pacchetti Fedora. È un catalogo più piccolo e più conservativo di **Flathub**, che è il catalogo comunitario più grande, dove la maggior parte degli sviluppatori di applicazioni pubblica direttamente.

Nella pagina `appendice-package-manager.md` avevo già notato che Flathub può essere stato abilitato durante l'installazione oppure no, a seconda delle risposte date. Lo verifico invece di darlo per scontato:

```bash
flatpak remotes --show-details
```

Mi aspetto di vedere almeno `fedora`; `flathub` compare solo se è già stato abilitato. La colonna delle opzioni indica anche se il remote è a livello `system` o `user`.

⚠️ **Da verificare:** se `flathub` risulta già presente ma l'elenco di app che vedo è più ristretto del catalogo completo, è possibile che sia abilitato con un **filtro** (Fedora può proporre Flathub in forma filtrata nel flusso dei repository di terze parti; non ho verificato come si comporti la mia installazione). Se serve, `flatpak remote-modify --no-filter flathub` rimuove il filtro. Prima di farlo, controllo l'output di `flatpak remotes --show-details` e decido io.

---

## 3. Aggiungere Flathub

```bash
flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
```

Cosa fa, pezzo per pezzo:

- `remote-add` registra una **sorgente** di applicazioni (un *remote*, l'equivalente di un repository per DNF);
- `--if-not-exists` evita l'errore se un remote con quel nome esiste già: il comando si può rilanciare senza danni;
- `flathub` è il nome locale che gli do (quello che userò poi in `flatpak install flathub ...`);
- l'URL punta a un file `.flatpakrepo`, un piccolo file di descrizione che contiene indirizzo del repository e chiave di firma.

Senza altre opzioni, `flatpak remote-add` lavora a livello **di sistema**, quindi può chiedere l'autorizzazione (il prompt della password via polkit). Per aggiungerlo solo per il mio utente, senza privilegi:

```bash
flatpak remote-add --user --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
```

### Installazione *system* o *user*?

| | `--system` (default) | `--user` |
|---|---|---|
| Visibile a | Tutti gli utenti della macchina | Solo il mio utente |
| Permessi per installare | Richiede autorizzazione | Nessuno |
| Dati | `/var/lib/flatpak` | `~/.local/share/flatpak` |

Su un portatile con un solo utente la differenza pratica è piccola. Quello che conta è la **coerenza**: un remote aggiunto a livello *user* non è visibile a un'installazione *system* e viceversa. Se un'app non si trova, uno dei primi controlli è proprio questo.

### Cosa cambia in Discover

Dopo questo passaggio, Discover (il software center di KDE, che si appoggia a PackageKit, come visto nell'appendice) mostra le app di Flathub accanto ai pacchetti RPM. Se non compaiono subito, chiudo e riapro Discover.

---

## 4. Installare, avviare, aggiornare

```bash
flatpak search <nome>
flatpak install flathub <ID-applicazione>
flatpak run <ID-applicazione>
```

L'**ID applicazione** ha la forma di un nome a dominio invertito (es. `org.mozilla.firefox`) ed è l'identificativo univoco dell'app: quello che si usa nei comandi e che dà il nome alla cartella dei dati in `~/.var/app/`.

Quando installo da terminale, vale la pena guardare il riepilogo: oltre all'app, Flatpak elenca i **runtime** che deve scaricare (se non li ha già). Se è il primo Flatpak che installo, il download è consistente; dai successivi molto meno, perché i runtime sono condivisi.

Gli aggiornamenti si fanno con:

```bash
flatpak update
```

Questi sono **separati** da `sudo dnf upgrade`: aggiornare il sistema non aggiorna i Flatpak e viceversa. Discover mostra entrambi nella scheda degli aggiornamenti.

---

## 5. Flatseal: vedere (e cambiare) i permessi di un Flatpak

Ogni Flatpak dichiara in un **manifest** a cosa ha bisogno di accedere: file, rete, dispositivi, servizi di sistema. Quei permessi sono quelli *predefiniti* scelti da chi ha impacchettato l'app, e non è detto che siano il minimo indispensabile per il mio uso. **Flatseal** è un'applicazione grafica, a sua volta distribuita come Flatpak, che permette di **leggere e modificare i permessi di ciascuna app** senza ricordare le opzioni da riga di comando.

```bash
flatpak install flathub com.github.tchx84.Flatseal
```

Come funziona, in due righe:

- scelgo un'app dalla colonna di sinistra;
- a destra trovo i permessi raggruppati per categoria (accesso ai file, rete, dispositivi, socket come Wayland/X11, servizi sul bus di sistema o di sessione...) con un interruttore per ciascuno. Le modifiche che faccio vengono salvate come **override** per quell'app, senza toccare il pacchetto originale.

Un dettaglio che conta: poiché gli override stanno separati dal manifest, un aggiornamento dell'app non li cancella. Se un'app smette di funzionare dopo che ho tolto un permesso, so dove guardare. In Flatseal c'è anche un modo per riportare un'app ai permessi originali (se non lo trovo subito, da riga di comando c'è `flatpak override --reset`, vedi tabella).

### Cosa guardare per prima cosa

- **File:** `Filesystem` è la sezione più importante. Il permesso `All user files` (o `home`) dà all'app accesso a quasi tutto il mio spazio personale; `host` (o `All system files`) è ancora più ampio. Se un'app ha uno di questi, il sandbox protegge molto meno.
- **Rete:** una calcolatrice con accesso alla rete merita almeno una domanda.
- **Dispositivi:** accesso a tutti i dispositivi (`All devices`) apre più di quanto serva alla maggior parte delle app.
- **Servizi di sessione/sistema:** voci come accesso al bus di sessione vanno lette con attenzione, perché permettono di comunicare con altri servizi fuori dal sandbox.

Non tolgo permessi a caso: se un'app ne ha bisogno per funzionare, toglierlo la rompe. Procedo per tentativi, un permesso alla volta, e annoto quello che ho cambiato (vedi sezione 7).

### Alternativa dentro KDE

Plasma ha anche un modulo nelle Impostazioni di sistema per i permessi delle applicazioni Flatpak (se il pacchetto è installato sul mio sistema: non ho verificato). Offre meno dettaglio di Flatseal, ma è già integrato nel desktop. Flatseal resta, per quello che so, lo strumento più completo.

### Perché alcune cose "funzionano lo stesso": i portal

Anche con pochi permessi, un'app Flatpak può aprire un file che scelgo io. Succede tramite i **portal**: quando l'app vuole aprire un file, il sistema mostra *il suo* selettore di file e passa all'app **solo il file scelto**. Per questo restringere `Filesystem` non impedisce necessariamente di aprire e salvare quello che serve: spesso basta il selettore. Se un'app lo ignora e pretende un accesso diretto a percorsi specifici, è lì che serve un permesso esplicito.

---

## 6. Buone pratiche che mi do

1. **Guardo la provenienza prima di installare.** Su Flathub, la pagina di un'app indica se è **verificata** (cioè se il publisher è stato verificato come chi mantiene davvero l'app). Un'app non verificata non è per forza pericolosa, ma lo segnalo a me stesso: chi l'ha impacchettata non è necessariamente chi la sviluppa.
2. **Lavoro con il minimo dei permessi.** Dopo ogni installazione apro Flatseal e guardo almeno `Filesystem`, `Network` e `Devices`. Tolgo i permessi che non servono per l'uso che ne faccio, uno alla volta.
3. **Preferisco cartelle specifiche a `home`/`host`.** Se un'app deve accedere a una cartella (es. `~/Documenti/Progetti`), glielo concedo solo per quel percorso, non per tutta la home.
4. **Non mescolo RPM e Flatpak dello stesso programma.** Se già ho l'RPM, non installo anche il Flatpak "per provare" senza un motivo preciso: i due hanno dati separati (il Flatpak in `~/.var/app/`) e si finisce a non sapere quale si sta usando.
5. **Tengo d'occhio lo spazio.** Runtime e app occupano spazio, e le versioni vecchie dei runtime restano finché non le tolgo. Ogni tanto: `flatpak uninstall --unused` (vedi tabella) e un controllo con `du -sh ~/.var/app/*`.
6. **Faccio backup dei dati giusti.** Se un Flatpak contiene dati che non voglio perdere, quelli sono in `~/.var/app/<ID>/`, non nella cartella di installazione dell'app. Disinstallare un Flatpak non cancella quella cartella a meno che non lo chieda esplicitamente (`--delete-data`).
7. **Sull'hardware, penso al sandbox.** Un'app Flatpak che deve usare GPU, stampanti, webcam o dispositivi USB può aver bisogno di permessi specifici. Se qualcosa non funziona, parto da lì prima di sospettare un bug dell'app.
8. **Aggiorno con regolarità.** I Flatpak non seguono `dnf upgrade`: li aggiorno io con `flatpak update` (o da Discover). Un runtime non aggiornato può restare indietro con correzioni di sicurezza.
9. **Annoto gli override.** Come per le modifiche con `config-manager setopt` in `rpm-fusion.md`, un permesso modificato a mano resta lì anche mesi dopo e me lo dimentico. Mi segno quali app ho personalizzato e perché.

---

## 7. Fotografia dello stato (prima e dopo)

Come per gli altri capitoli, un piccolo checkpoint per confrontare i due stati con `diff`:

```bash
mkdir -p ~/post-install-info
ETICHETTA=prima      # cambia in "dopo" quando rilanci a configurazione fatta

CHECK=~/post-install-info/flatpak-checkpoint-$ETICHETTA-$(date +%Y-%m-%d_%H-%M).txt

{
  echo "=== Checkpoint Flatpak ($ETICHETTA) — $(date '+%A %d %B %Y, %H:%M:%S') ==="
  echo

  echo "--- Versione di Flatpak ---"
  flatpak --version
  echo

  echo "--- Remotes (system e user) ---"
  flatpak remotes --show-details
  echo

  echo "--- Applicazioni installate ---"
  flatpak list --app --columns=application,version,origin,installation
  echo

  echo "--- Runtime installati ---"
  flatpak list --runtime --columns=application,version,origin,installation
  echo

  echo "--- Override attivi (system) ---"
  flatpak override --system --show 2>&1
  echo

  echo "--- Override attivi (user) ---"
  flatpak override --user --show 2>&1
  echo

  echo "--- Spazio occupato dai dati delle app ---"
  du -sh ~/.var/app/* 2>/dev/null || echo "nessuna cartella in ~/.var/app"
  echo
} > "$CHECK" 2>&1

echo "Checkpoint salvato in: $CHECK"
```

Note sulla sintassi:

- `--columns=...` sceglie le colonne dell'output di `flatpak list`;
- `2>&1` porta i messaggi di errore nello stesso file dell'output normale;
- `2>/dev/null` sul `du` scarta l'errore che comparirebbe se `~/.var/app` è ancora vuoto, e il `||` stampa una riga leggibile al suo posto.

Per confrontare i due stati:

```bash
diff ~/post-install-info/flatpak-checkpoint-prima-*.txt \
     ~/post-install-info/flatpak-checkpoint-dopo-*.txt
```

---

## 8. Tabella dei comandi utili

| Cosa voglio fare | Comando |
|---|---|
| Elencare i remote (con dettagli e livello system/user) | `flatpak remotes --show-details` |
| Aggiungere Flathub (system) | `flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo` |
| Aggiungere Flathub (solo per me) | `flatpak remote-add --user --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo` |
| Cercare un'app | `flatpak search <testo>` |
| Installare un'app | `flatpak install flathub <ID>` |
| Installare solo per il mio utente | `flatpak install --user flathub <ID>` |
| Avviare un'app | `flatpak run <ID>` |
| Elencare le app installate | `flatpak list --app` |
| Elencare anche i runtime | `flatpak list --runtime` |
| Info su un'app (versione, origine, runtime, dimensione) | `flatpak info <ID>` |
| Vedere i permessi *dichiarati* da un'app | `flatpak info --show-permissions <ID>` |
| Vedere gli override che ho impostato (utente) | `flatpak override --user --show <ID>` |
| Aggiungere un permesso a un'app | `flatpak override --user --filesystem=<percorso> <ID>` |
| Togliere un permesso a un'app | `flatpak override --user --nofilesystem=<percorso> <ID>` |
| Annullare tutti gli override di un'app | `flatpak override --user --reset <ID>` |
| Aggiornare tutto | `flatpak update` |
| Vedere cosa è aggiornabile senza applicare | `flatpak remote-ls --updates` |
| Disinstallare un'app | `flatpak uninstall <ID>` |
| Disinstallare e cancellare anche i dati | `flatpak uninstall --delete-data <ID>` |
| Rimuovere runtime non più usati | `flatpak uninstall --unused` |
| Controllare e riparare l'installazione | `flatpak repair` |
| Cronologia di installazioni/aggiornamenti | `flatpak history` |
| Aprire una shell dentro il sandbox di un'app | `flatpak run --command=sh <ID>` |
| Vedere i processi Flatpak in esecuzione | `flatpak ps` |
| Fermare un'app Flatpak | `flatpak kill <ID>` |
| Spazio occupato dai dati delle app | `du -sh ~/.var/app/*` |
| Vedere i permessi concessi tramite portal | `flatpak permissions` |

Alcune note:

- `override` **senza** `--user` o `--system` va a livello di sistema e richiede autorizzazione. Preferisco `--user`, per tenere le mie scelte confinate al mio utente.
- `--filesystem=<percorso>` accetta anche scorciatoie come `home` e `host`, che sono proprio i permessi ampi da evitare quando possibile (sezione 5). Accetta anche il suffisso `:ro` per l'accesso in sola lettura (es. `--filesystem=~/Documenti:ro`).
- `flatpak run --command=sh <ID>` è utile per capire **cosa vede davvero** l'app: da quella shell controllo se un percorso esiste o no, e quindi se il sandbox lo lascia passare.
- `flatpak permissions` mostra i permessi gestiti dai portal (es. file già concessi a un'app). Non ho verificato l'output esatto su questo sistema.

---

## 9. Riepilogo operativo

- [ ] Controllare i remote presenti: `flatpak remotes --show-details`
- [ ] Salvare il checkpoint con `ETICHETTA=prima`
- [ ] Aggiungere Flathub se manca (e, se è filtrato, decidere se togliere il filtro)
- [ ] Decidere *system* o *user* e restare coerenti
- [ ] Installare Flatseal: `flatpak install flathub com.github.tchx84.Flatseal`
- [ ] Per ogni app installata: guardare `Filesystem`, `Network`, `Devices` in Flatseal e restringere ciò che non serve
- [ ] Annotare gli override personalizzati
- [ ] Ricordare che gli aggiornamenti Flatpak sono separati da `dnf upgrade`
- [ ] Rilanciare il checkpoint con `ETICHETTA=dopo` e confrontare con `diff`
- [ ] Annotare qui cosa effettivamente ho visto sul mio sistema (remote presenti, filtro, permessi)
