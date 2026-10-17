# Post-installazione — Parte 1: i primi passi su Fedora 44 KDE Plasma

Appena finita l'installazione, prima ancora di pensare a software, repository di terze parti o configurazioni estetiche, conviene fare tre cose nell'ordine giusto: **capire esattamente cosa si ha davanti**, **aggiornare tutto**, e **mettere a punto il gestore di pacchetti** che userai per ogni cosa da qui in avanti. Sono passaggi "noiosi", ma è proprio l'ordine in cui vengono fatti che evita un sacco di problemi più avanti: installare software terzo su un sistema non aggiornato, ad esempio, è uno dei modi più comuni per finire a inseguire conflitti di dipendenze che in realtà non hanno nulla a che fare col pacchetto che si sta installando.

---

## Step 1 — Apri un terminale e orientati

Su KDE Plasma il terminale predefinito è **Konsole** (lo trovi nel menu applicazioni, o con la scorciatoia di default `Ctrl+Alt+T`). Da qui in avanti, quasi tutto passerà da qui.

### Capire cosa hai installato

I tre comandi di base:

```bash
cat /etc/fedora-release
uname -r
rpm -E %fedora        # stampa "44" — riutilizzerai questo valore più avanti
```

- `cat /etc/fedora-release` stampa semplicemente una riga di testo leggibile, tipo "Fedora release 44 (...)". È la forma più immediata per confermare la versione.
- `uname -r` stampa la versione del **kernel** attualmente in esecuzione (es. `6.x.x-...fc44.x86_64`) — non la versione di Fedora, ma quella del cuore del sistema operativo. Sono due numeri diversi che spesso si confondono da principianti: Fedora 44 è la "distribuzione" (l'insieme di pacchetti, configurazioni e convenzioni), il kernel è un singolo componente che Fedora stessa aggiorna periodicamente, anche più volte nella vita della stessa release.
- `rpm -E %fedora` è leggermente diverso dai primi due: non stampa informazioni a sé, stampa il valore di una **variabile di RPM** usata internamente dal sistema (in questo caso, il numero di versione di Fedora). Si usa raramente da solo — il motivo per cui conta davvero è quello che segue.

### Perché quella variabile torna utile

Molte guide online (e molti comandi che userai tu stesso in futuro) includono percorsi tipo:

```
https://download.example.org/repo/fedora/44/x86_64/
```

con il numero di versione scritto a mano, "hardcoded". Il problema è che tra qualche mese, quando farai l'upgrade a Fedora 45, quel comando copiato da una guida vecchia punterà ancora a `44` — e o fallisce silenziosamente, o (peggio) installa pacchetti pensati per la versione sbagliata. La soluzione è la **command substitution** di bash: la sintassi `$(comando)` dice alla shell "esegui questo comando, e sostituisci qui il suo output testuale". Quindi invece di scrivere:

```bash
# da evitare: valore fisso, scade al prossimo upgrade
sudo dnf config-manager addrepo --from-repofile=https://esempio.org/repo-44.repo
```

scriverai:

```bash
# si aggiorna automaticamente con la versione di Fedora in uso
sudo dnf config-manager addrepo --from-repofile=https://esempio.org/repo-$(rpm -E %fedora).repo
```

Quando bash incontra `$(rpm -E %fedora)`, esegue quel comando per primo, ottiene `44` come output, e lo inserisce letteralmente al posto suo nella riga — risultato finale identico a scriverlo a mano, ma generato in automatico ogni volta che il comando gira. È un pattern che vedrai ripetuto spessissimo nelle guide per Fedora (e in generale ovunque in bash), quindi vale la pena fissarlo bene da subito.

### Salvare uno "scatto" dello stato iniziale del sistema

Appena finita l'installazione è il momento migliore per fotografare lo stato del sistema — prima che inizi ad aggiungere repository, pacchetti e configurazioni che lo allontanano dallo stato "pulito". Questo file ti torna utile in due casi concreti: se tra un anno ti chiedi "ma io inizialmente che impostazioni avevo?", oppure se devi ricostruire il sistema da zero su un'altra macchina e vuoi un riferimento di partenza.

Creiamo una cartella dedicata e un file di testo con data e ora nel nome, così ogni scatto successivo (ad esempio prima di un upgrade di versione) non sovrascrive quelli precedenti:

```bash
mkdir -p ~/post-install-info

SNAPSHOT=~/post-install-info/snapshot-$(date +%Y-%m-%d_%H-%M).txt

{
  echo "=== Snapshot di sistema — $(date '+%A %d %B %Y, %H:%M:%S') ==="
  echo

  echo "--- Utente e macchina ---"
  echo "Utente corrente : $(whoami)"
  echo "UID / gruppi    : $(id)"
  echo "Hostname        : $(hostnamectl hostname)"
  echo

  echo "--- Versione Fedora e kernel ---"
  cat /etc/fedora-release
  echo "Kernel          : $(uname -r)"
  echo "Architettura    : $(uname -m)"
  echo

  echo "--- CPU e memoria ---"
  lscpu | grep -E "Model name|CPU\(s\)|Thread|Socket"
  free -h
  echo

  echo "--- Dischi, partizioni e filesystem ---"
  lsblk -f
  echo
  df -h --output=source,fstype,size,used,avail,pcent,target
  echo

  echo "--- Rete ---"
  ip -brief address
  echo

  echo "--- Lingua, tastiera, fuso orario ---"
  localectl status
  timedatectl status
  echo

  echo "--- Repository DNF abilitati ---"
  dnf5 repolist
  echo

  echo "--- Numero totale di pacchetti installati ---"
  rpm -qa | wc -l

} > "$SNAPSHOT"

echo "Snapshot salvato in: $SNAPSHOT"
```

Qualche nota su cosa succede in questo blocco, utile se bash non ti è ancora familiare:

- `mkdir -p` crea la cartella `post-install-info` nella tua home (`~`) senza dare errore se esiste già (`-p` sta per "parents", crea anche eventuali cartelle intermedie mancanti).
- `date +%Y-%m-%d_%H-%M` formatta data e ora secondo il formato che gli passi (anno-mese-giorno_ora-minuto). È lo stesso comando `date` usato dentro `echo "=== ... $(date ...) ==="`, ma con un formato diverso (`'%A %d %B %Y, %H:%M:%S'`, pensato per essere leggibile da umani) — due chiamate allo stesso comando, due scopi diversi: una per il nome del file (niente spazi o caratteri strani), una per l'intestazione leggibile dentro al file.
- Le parentesi graffe `{ ... }` raggruppano tutti i comandi al loro interno in un unico blocco. Il simbolo `>` subito dopo la parentesi graffa di chiusura reindirizza **l'intero output del blocco** in un solo file, invece di dover scrivere `>>` (append, "aggiungi in fondo") dopo ogni singola riga. È una scorciatoia molto comune quando si vuole salvare l'output di più comandi in sequenza in un unico file.

Il file conterrà, tra le altre cose, anche il tuo **username** (`whoami`), così come richiesto — utile se in futuro reinstalli e vuoi ricreare lo stesso nome utente per evitare problemi di permessi su backup o configurazioni già pronte.

Separatamente, dato che la lista completa dei pacchetti installati può essere lunga (migliaia di righe), conviene tenerla in un file a parte invece di affollare lo snapshot principale:

```bash
rpm -qa --qf '%{NAME}-%{VERSION}-%{RELEASE}.%{ARCH}\n' | sort \
  > ~/post-install-info/pacchetti-$(date +%Y-%m-%d).txt
```

`rpm -qa` elenca tutti i pacchetti installati; il parametro `--qf` (query format) controlla come viene stampata ogni riga — qui si chiede esplicitamente nome, versione, release e architettura, che è un formato più stabile e leggibile rispetto all'output di default. `sort` ordina alfabeticamente il risultato prima di salvarlo, così il file è facilmente confrontabile (con `diff`, ad esempio) con uno scatto preso in un secondo momento, per vedere subito cosa è cambiato.

---

## Step 2 — Aggiorna il sistema *prima* di aggiungere altro

Un'installazione fresca di Fedora non è mai davvero "aggiornata al minuto": tra il momento in cui è stata creata l'immagine ISO che hai usato e il momento in cui stai leggendo questa riga, Fedora ha quasi certamente pubblicato nuovi aggiornamenti (di sicurezza, di pacchetti, a volte anche del kernel). Aggiornare subito, prima di installare qualsiasi altra cosa, significa partire da una base coerente — aggiungere repository di terze parti o software extra *prima* di questo passaggio è uno dei modi più comuni per ritrovarsi a risolvere conflitti di dipendenze che in realtà sono solo sintomi di un sistema di base non allineato.

```bash
sudo dnf upgrade --refresh -y
```

- `--refresh` forza DNF a scartare la cache dei metadati che ha attualmente e a riscaricarla da zero dai repository, prima di controllare se ci sono aggiornamenti disponibili (vedi il riquadro sulla cache qui sotto per capire bene cosa significa).
- `-y` risponde automaticamente "sì" a tutte le richieste di conferma (dimensione del download, importazione di chiavi GPG, ecc.), così il processo va avanti senza fermarsi ad aspettare input.

### Capire la cache (perché conta, non solo per questo comando)

"Cache" è un termine che tornerà spessissimo lavorando con DNF, quindi vale la pena fissarlo bene una volta per tutte, con un'analogia semplice.

Immagina una biblioteca. C'è il **catalogo** — un elenco di tutti i libri disponibili, dove si trovano, quante copie ci sono — e ci sono i **libri fisici** che hai effettivamente preso in prestito e tieni sulla scrivania. Sono due cose diverse, gestite in modo diverso:

- Il **catalogo** corrisponde ai **metadati** dei repository: un indice (non i pacchetti veri e propri, solo l'elenco di cosa esiste, con nomi e versioni) che DNF scarica dai server dei repository abilitati e tiene in cache localmente, così non deve riscaricarlo ogni volta che lanci un comando. Per default questo indice viene considerato "ancora valido" per un certo periodo di tempo (circa 48 ore) prima di essere aggiornato automaticamente — un compromesso ragionevole nell'uso quotidiano, ma subito dopo un'installazione vuoi essere sicuro di vedere l'elenco più recente possibile, da cui il flag `--refresh`.
- I **libri fisici sulla scrivania** corrispondono ai **pacchetti scaricati** (i file `.rpm` veri e propri): quando installi qualcosa, DNF lo scarica e, a seconda della configurazione, lo tiene anche dopo l'installazione in una cartella di cache locale (`/var/cache/dnf`). Se in futuro reinstalli lo stesso pacchetto, o fai un downgrade a una versione che avevi già scaricato in precedenza, DNF può "ripescarlo" da lì invece di riscaricarlo da internet.

Tenerli distinti in testa aiuta a capire due cose che altrimenti sembrano comportamenti strani di DNF: perché a volte un pacchetto appena pubblicato non si vede subito (il catalogo/metadati non si è ancora aggiornato) e perché la cartella `/var/cache/dnf` può crescere nel tempo anche se non la usi esplicitamente (sono i "libri" accumulati). Si può sempre svuotare con `sudo dnf5 clean packages` (solo i pacchetti scaricati) o `sudo dnf5 clean all` (pacchetti e metadati), se un giorno serve recuperare spazio.

### Riavvia se è stato aggiornato il kernel

Se `dnf upgrade` ha incluso un aggiornamento del kernel, conviene riavviare **prima** di procedere con qualsiasi altra configurazione:

```bash
sudo reboot
```

Il motivo: alcuni passaggi successivi (ad esempio l'installazione di driver proprietari che compilano moduli per il kernel, come spesso accade con le GPU NVIDIA) costruiscono quei moduli contro il kernel **effettivamente in esecuzione** in quel momento. Se li installi prima di riavviare su un kernel più vecchio, e poi riavvii su quello nuovo, quei moduli potrebbero non caricarsi più correttamente. Conviene quindi essere già sul kernel definitivo prima di compilare qualsiasi cosa contro di esso.

---

## Step 3 — Metti a punto DNF5

### Cosa si vuole ottenere con questo passaggio, e perché

DNF5 funziona perfettamente anche con la configurazione di default, quindi questo step non è "obbligatorio" nel senso stretto del termine — è più una questione di adattare uno strumento che userai quotidianamente al modo in cui vuoi effettivamente lavorarci. Le tre modifiche che seguono toccano tre aspetti diversi dell'esperienza d'uso:

1. **la velocità** con cui DNF scarica i pacchetti (download in parallelo da più mirror invece che uno alla volta);
2. **l'attrito** nell'uso quotidiano (non dover confermare manualmente ogni singola transazione);
3. **il riutilizzo** di quanto già scaricato (collegandosi direttamente al concetto di cache visto nello Step 2).

Nessuna di queste modifiche cambia *cosa* DNF installa o come risolve le dipendenze — toccano solo *come* si comporta durante il processo. È una configurazione dichiarativa: un file di testo che DNF rilegge a ogni esecuzione, non un comando una tantum.

### Il file di configurazione

Il file è `/etc/dnf/dnf.conf`, di proprietà di `root` — da qui la necessità di `sudo` per modificarlo. Lo si può aprire con un editor di testo, ma qui usiamo un approccio non interattivo via terminale, utile anche per scriptare la configurazione su più macchine in futuro:

```bash
sudo tee -a /etc/dnf/dnf.conf > /dev/null <<'EOF'
max_parallel_downloads=10
defaultyes=True
keepcache=True
EOF
```

Vale la pena capire bene questa riga, perché contiene un paio di trucchi di bash che vedrai usati spesso:

- **Perché `tee` e non semplicemente `sudo echo "..." >> file`?** È una trappola molto comune per chi inizia con bash: il reindirizzamento (`>>`) viene gestito dalla shell *prima* di eseguire il comando, e la shell che apre il file è la tua shell normale, non quella con i permessi di `sudo` — quindi `sudo echo "..." >> /etc/dnf/dnf.conf` fallisce con un errore di permessi, anche se hai scritto `sudo` giusto davanti. Il comando `tee`, invece, è lui stesso il programma che scrive sul file, e viene eseguito *con* i permessi di `sudo` — per questo si usa `sudo tee` invece di mettere `sudo` davanti a un reindirizzamento.
- `-a` dice a `tee` di scrivere in **append** (aggiungere in fondo al file esistente), non di sovrascriverlo — fondamentale, dato che `dnf.conf` ha già contenuto che vuoi conservare.
- `> /dev/null` alla fine serve solo a silenziare l'output "di cortesia" che `tee` stamperebbe anche a schermo (di norma `tee` scrive sia su file sia sul terminale) — qui non serve vederlo due volte, quindi lo si butta nel "cestino" di Linux, `/dev/null`.
- `<<'EOF' ... EOF` è un **heredoc**: un modo per passare un blocco di testo multi-riga come input a un comando, senza dover scrivere `echo` riga per riga. Le virgolette singole attorno a `EOF` (`<<'EOF'`, non `<<EOF`) dicono a bash di trattare il testo dentro al blocco come letterale, senza provare a espandere eventuali `$variabili` al suo interno — qui non ne servono, ma è un'abitudine utile da prendere, perché evita sorprese se in futuro un blocco simile contenesse per sbaglio un simbolo `$`.

⚠️ Nota pratica: questo comando **aggiunge** righe, non le sostituisce. Se lo lanci due volte per sbaglio, finisci con `max_parallel_downloads=10` scritto due volte nel file (DNF probabilmente userà l'ultima occorrenza, ma è comunque disordinato). Prima di rilanciarlo, conviene controllare se le righe ci sono già:

```bash
grep -E "max_parallel_downloads|defaultyes|keepcache" /etc/dnf/dnf.conf
```

### Cosa fa ciascuna impostazione, e a cosa prestare attenzione

- **`max_parallel_downloads=10`** — DNF normalmente scarica i pacchetti da un mirror alla volta; questa opzione gli permette di scaricarne più di uno in parallelo. Su una connessione decente è una delle modifiche con il miglior rapporto beneficio/rischio: si sente la differenza soprattutto durante un `dnf upgrade` con molti pacchetti da aggiornare. Un valore come 10 è un buon punto di partenza; esagerare (valori molto più alti) di solito non porta benefici ulteriori e in alcuni casi può solo affaticare inutilmente la connessione o il mirror scelto.
- **`defaultyes=True`** — fa sì che DNF risponda automaticamente "sì" alle domande di conferma, un po' come il flag `-y` visto nello Step 2, ma applicato di default a *ogni* transazione, non solo a quella specifica. È comodo, ma vale una precisazione importante: **non salta il riepilogo della transazione**, salta solo la pressione del tasto di conferma — DNF ti mostrerà comunque cosa sta per installare, rimuovere o aggiornare prima di procedere. Attivalo solo se sei comunque abituato a leggere quel riepilogo prima che scorra via: è lì che si notano, ad esempio, rimozioni indesiderate di pacchetti causate da conflitti di dipendenze.
- **`keepcache=True`** — collegandosi a quanto visto nello Step 2: con questa opzione, i pacchetti `.rpm` scaricati restano in `/var/cache/dnf` anche dopo che l'installazione è andata a buon fine (il comportamento di default li elimina subito per risparmiare spazio). Il vantaggio si vede se capita di reinstallare lo stesso pacchetto, o di fare un downgrade con `dnf5 downgrade` verso una versione già scaricata in precedenza: niente da riscaricare da internet. Il costo è lo spazio su disco occupato da quella cache nel tempo — da tenere d'occhio se il disco è piccolo, recuperabile in qualsiasi momento con `sudo dnf5 clean packages`.

Per confermare che le righe siano state scritte correttamente:

```bash
cat /etc/dnf/dnf.conf
```
