# Post-installazione — Parte 2: RPM Fusion e i repository di terze parti

Dopo aver aggiornato il sistema e messo a punto DNF5, il passo successivo che ho fatto è stato abilitare **RPM Fusion**: il repository di complemento che raccoglie ciò che Fedora, per ragioni di licenza o di brevetti, non può includere nei propri repository ufficiali. Esempi tipici: la versione completa di `ffmpeg` con i codec non liberi, i driver proprietari NVIDIA, Steam.

Sul mio Zenbook (GPU AMD integrata) la parte NVIDIA non mi serve; quello che interessa è soprattutto il lato multimediale. Abilito comunque **entrambi i rami**, *Free* e *Nonfree*, perché sono pensati per essere usati in coppia (il Nonfree dipende dal Free).

Questo capitolo ha una particolarità: la procedura "da manuale" sono due righe, ma su Fedora 44 ho incontrato (e trovato segnalata da altri) una trappola che porta DNF a pescare pacchetti dal repository **rawhide**, cioè dalla versione di sviluppo di Fedora. Per questo il capitolo dedica molto più spazio alle **verifiche** che all'installazione vera e propria.

---

## 1. Da dove arrivano queste informazioni

Prima dei comandi, le fonti. Un comando che tocca la lista dei repository è una cosa che voglio poter ricontrollare nel tempo, non un incantesimo copiato da qualche parte.

| Fonte | A cosa mi serve |
|---|---|
| [rpmfusion.org/Configuration](https://rpmfusion.org/Configuration) | La pagina del progetto con i comandi di installazione. È **il riferimento** per la procedura. |
| [github.com/rpmfusion/rpmfusion-free-release](https://github.com/rpmfusion/rpmfusion-free-release) (e il gemello `-nonfree-`) | I sorgenti dei pacchetti `release`: da qui si vedono il contenuto dei file `.repo` e la cronologia delle modifiche per ogni ramo di Fedora. |
| [Fedora Discussion — "Fedora 44 and RPMFusion"](https://discussion.fedoraproject.org/t/fedora-44-and-rpmfusion/189911) | Il thread (aprile–maggio 2026) in cui il problema dei repo `rawhide` su Fedora 44 viene descritto, con l'intervento di chi gestisce i file di RPM Fusion. |
| [Fedora Discussion — "Cannot update f42 to f43 with rpmfusion repos installed"](https://discussion.fedoraproject.org/t/cannot-update-f42-to-f43-with-rpmfusion-repos-installed/162878) | Lo stesso tipo di problema su Fedora 43 (agosto 2025): serve a capire che **non è un caso isolato di Fedora 44**. |

---

## 2. Repository di terze parti: cosa sono e come vedere a quali sono "iscritto"

RPM Fusion non sarà l'ultimo repository che aggiungo. È molto probabile che prima o poi ne arrivino altri: ad esempio per installare **Visual Studio Code**, che Microsoft distribuisce tramite un proprio repository, o altri software che non stanno nei repository Fedora. Ogni volta il meccanismo è lo stesso, ed è utile averlo chiaro adesso:

- un file `.repo` in `/etc/yum.repos.d/` dice a DNF *dove* guardare;
- una chiave GPG dice a DNF *di chi fidarsi* per quei pacchetti;
- da quel momento, DNF considera **tutti** i repository abilitati insieme, quando cerca un pacchetto o risolve dipendenze.

Quest'ultimo punto è quello che rende pericoloso un repository configurato male. Un'analogia: i repository Fedora sono il supermercato di quartiere, quelli di terze parti sono negozi aggiuntivi in cui decidi di fare la spesa. Se cerchi "latte" e due negozi lo vendono, DNF di default sceglie **la versione più recente** tra quelle disponibili, senza badare a quale negozio la offra. Quindi un repository che ha pacchetti "più nuovi" di quelli adatti al tuo sistema li vince, anche se sono incompatibili. È esattamente quello che succede col problema descritto più sotto.

Per questo, ogni volta che aggiungo un repository, voglio poter rispondere a una domanda: **quali repository ha il sistema, e quali sono attivi?**

### I comandi per vedere i repository

```bash
dnf repo list                  # solo i repository abilitati
dnf repo list --all            # abilitati e disabilitati
dnf repo info rpmfusion-free   # dettagli di un singolo repository (per ID)
ls /etc/yum.repos.d/           # i file .repo presenti sul disco
```

(`dnf repolist` senza lo spazio funziona ancora come alias, ed è la forma che incontrerai nelle guide più vecchie.)

Il primo comando mostra l'**ID** (la sigla breve, tipo `fedora` o `updates`) e il nome descrittivo di ogni repository. L'ID è quello che serve poi per abilitarlo o disabilitarlo.

Altre due verifiche utili:

```bash
# pacchetti installati il cui nome inizia per "rpmfusion", con il repo di provenienza
dnf list --installed 'rpmfusion*'

# chiavi GPG importate nel sistema (una per ogni repository "firmato" che ho usato)
rpm -qa gpg-pubkey --qf '%{NAME}-%{VERSION}-%{RELEASE}  %{SUMMARY}\n'
```

Le virgolette attorno a `'rpmfusion*'` servono a impedire che sia la shell a interpretare l'asterisco (come wildcard sui nomi dei file nella cartella corrente) prima che arrivi a DNF: l'asterisco deve arrivare a DNF così com'è.

### Fotografia *prima* di aggiungere RPM Fusion

Come per lo snapshot iniziale del sistema, salvo lo stato dei repository **prima** di toccare qualcosa, così da poter fare un confronto con `diff` dopo:

```bash
dnf repo list --all > ~/post-install-info/repo-prima-rpmfusion-$(date +%Y-%m-%d).txt
```

---

## 3. Installazione dei pacchetti release

La procedura che trovo sulla pagina ufficiale è questa:

```bash
sudo dnf install \
  https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
  https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```

Cosa succede, riga per riga:

- il `\` a fine riga dice a bash "il comando continua sulla riga successiva": serve solo a rendere leggibile un comando lungo;
- `dnf install` accetta anche **URL di file `.rpm`** al posto del nome di un pacchetto: scarica il file e lo installa direttamente;
- `$(rpm -E %fedora)` è la command substitution già vista in `primi-passi.md`: diventa `44` e ricalcola da solo il numero giusto quando passerò a Fedora 45;
- quello che installo **non è RPM Fusion**, ma solo i pacchetti `rpmfusion-free-release` e `rpmfusion-nonfree-release`. Sono piccoli (una decina di KB) e contengono i file `.repo` e le chiavi GPG. È DNF, poi, a scaricare da lì i pacchetti veri.

Nota sulla fiducia: questo è il momento in cui mi fido del sito (HTTPS, dominio ufficiale) perché le chiavi di RPM Fusion non sono ancora nel sistema. Dopo l'installazione, i pacchetti scaricati dal repository vengono invece verificati con la chiave GPG, come in qualsiasi altro repository.

🛑 **Qui mi fermo.** Prima di installare qualsiasi altra cosa (codec, appstream, driver) controllo che i repository siano quelli giusti. Il motivo è nella sezione seguente.

---

## 4. Il problema: repository `rawhide` al posto di quello di Fedora 44

### Il sintomo

Dopo l'installazione dei pacchetti release su Fedora 44, c'è chi si ritrova con i repository RPM Fusion **di rawhide** abilitati, e con quelli corretti assenti o disabilitati. Il sintomo tipico si vede quando si prova a installare qualcosa di multimediale, per esempio:

```text
nothing provides libm.so.6(GLIBC_2.44)(64bit) needed by ffmpeg-libs-...fc45.x86_64 from rpmfusion-free-rawhide
nothing provides glibc >= 2.43.9000-11 needed by ffmpeg-libs-...fc45.x86_64 from rpmfusion-free-rawhide
```

Letto con calma, è un messaggio chiaro: DNF sta cercando un `ffmpeg-libs` costruito per **Fedora 45** (`fc45`, quello che oggi è rawhide), che richiede una `glibc` più nuova di quella che ho su Fedora 44. Non è un errore di `ffmpeg`, né di `glibc`: è DNF che sta guardando nel posto sbagliato.

Per capire perché DNF scelga proprio quei pacchetti, vedi l'analogia nella sezione 2: tra due repository che offrono lo stesso pacchetto, vince la versione più recente, e quella di rawhide lo è sempre.

### Lo stato dell'arte (consultato il 17 ottobre 2026)

- **È un fenomeno documentato per Fedora 44.** Nel thread di Fedora Discussion di fine aprile 2026, un utente con un'installazione pulita ottiene solo i repo `rpmfusion-free-rawhide` e `rpmfusion-nonfree-rawhide` e i pacchetti release installati risultano alla versione `44-0.2`.
- **Non è la prima volta, né solo Fedora 44.** Thread analoghi esistono per Fedora 40 e 41 (2024) e per Fedora 43 (agosto 2025); in quest'ultimo caso un utente cita una segnalazione sul bugtracker di RPM Fusion (`#7165`, "mirror manager directs f43 to rawhide") e dice che l'amministratore l'ha sistemata il 20/8/2025. Il pattern sembra legato al periodo **subito dopo il branching** di una nuova release, quando RPM Fusion usa ancora la configurazione di rawhide.
- **La versione "giusta" del pacchetto release per Fedora 44 non è `44-0.x`.** Nel thread, chi gestisce i file di RPM Fusion mostra un sistema sano con `rpmfusion-free-release-44-3` e `rpmfusion-nonfree-release-44-3`, e precisa che `44-0.3` è la versione "di branch" (cioè pre-rilascio, in base alla convenzione RPM per cui una release che inizia con `0.` indica un pacchetto preliminare). Un listing del server, pubblicato nel thread, mostra che il link `rpmfusion-nonfree-release-44.noarch.rpm` punta alla `44-3` già dal 24 aprile 2026.
- **La causa esatta non è stabilita.** Nel thread ci sono più ipotesi: un mirror che serve un file obsoleto (ipotesi di un collaboratore, che propone di usare `download1` invece di `mirrors` nelle istruzioni), la necessità di usare `update` invece di `install` seguito da `distro-sync` (suggerimento di chi gestisce i file del repo), e un utente che dichiara di aver risolto con reinstallazione + `dnf makecache`. C'è anche chi segnala che, dopo l'installazione, i repo `rpmfusion-free` e `rpmfusion-free-updates` risultavano **disabilitati**, senza che si capisca chi li abbia disabilitati.
- **Non ho trovato conferme né smentite per i mesi successivi.** L'ultimo messaggio che ho letto è del 1° maggio 2026. Non so quindi se oggi, a ottobre 2026, la procedura standard dia sempre il risultato corretto. Non lo do per risolto.

Conclusione pratica: **non do per scontato né che il problema ci sia, né che non ci sia**. Faccio la procedura, poi verifico sempre.

---

## 5. Diagnosi: i repository sono quelli giusti?

Due comandi:

```bash
rpm -qa 'rpmfusion*release*' | sort
dnf repo list --all | grep -i rpmfusion
```

Cosa mi aspetto in un sistema sano:

| Cosa guardo | Atteso | Segnale d'allarme |
|---|---|---|
| Versione dei pacchetti `release` | `44-N` senza `0.` (nel thread: `44-3`) | `44-0.2`, `44-0.3` (versioni di branch) |
| Repo **abilitati** | `rpmfusion-free`, `rpmfusion-free-updates`, `rpmfusion-nonfree`, `rpmfusion-nonfree-updates` | Abilitati solo i `-rawhide`, oppure i quattro sopra risultano disabilitati |
| Repo `*-rawhide` | Assenti, oppure disabilitati | Abilitati |

Un dettaglio rispetto alla prima versione dei miei appunti: su Fedora 44 non esiste una sezione chiamata "fedora-44" nei file `.repo`. Gli ID sono quelli della tabella qui sopra, e vanno gestiti **tutti e quattro**: la coppia `-updates` conta quanto la coppia base, perché è lì che finiscono gli aggiornamenti.

Se tutto è come atteso, passo direttamente alla sezione 7.

---

## 6. Correzione

Se la diagnosi dice che qualcosa non va, prima capisco **quale dei due casi** ho davanti.

### Caso A — pacchetti release alla versione sbagliata (`44-0.x`)

È l'approccio che nel thread suggerisce chi gestisce i file di RPM Fusion: aggiornare i pacchetti release **dall'URL**, e poi riallineare i pacchetti installati ai repository corretti.

```bash
sudo dnf update \
  https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
  https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

sudo dnf clean metadata
sudo dnf distro-sync
```

- `update` invece di `install`: se i pacchetti sono già installati, `install` si limita a dire "già installato" (è quello che è successo nel thread); `update` confronta le versioni e aggiorna se il file scaricato è più recente.
- `clean metadata` scarta il "catalogo" in cache (vedi il riquadro sulla cache in `primi-passi.md`), così che DNF lo riscarichi dai repository corretti invece di ragionare su quello vecchio.
- **`distro-sync`** è diverso da `upgrade`: porta i pacchetti installati **alla versione presente nei repository abilitati, anche se questo significa tornare indietro** (downgrade). È proprio quello che serve per riportare a `fc44` eventuali pacchetti che erano arrivati da rawhide.

⚠️ Con `defaultyes=True` nel mio `dnf.conf`, premere Invio alla richiesta di conferma equivale a dire sì. Qui **il riepilogo della transazione va letto**: se tra i pacchetti "da riportare indietro" compare qualcosa che non mi aspetto, mi fermo.

Se, dopo `update`, la versione resta `44-0.x`, significa che l'URL mi ha servito di nuovo il file vecchio. Nel thread si ipotizza un mirror non aggiornato; la forma storica dell'URL è `download1.rpmfusion.org` al posto di `mirrors.rpmfusion.org` (stesso percorso, cambia solo il dominio). Va verificato sul momento che risponda, non lo do per garantito.

### Caso B — versione giusta, ma repo disabilitati o rawhide abilitato

Se i pacchetti sono alla `44-N` ma lo stato dei repository è sbagliato, li correggo a mano con `config-manager`:

```bash
sudo dnf config-manager setopt rpmfusion-free-rawhide.enabled=0
sudo dnf config-manager setopt rpmfusion-nonfree-rawhide.enabled=0

sudo dnf config-manager setopt rpmfusion-free.enabled=1
sudo dnf config-manager setopt rpmfusion-free-updates.enabled=1
sudo dnf config-manager setopt rpmfusion-nonfree.enabled=1
sudo dnf config-manager setopt rpmfusion-nonfree-updates.enabled=1
```

Il formato è `setopt <id-repo>.<opzione>=<valore>`. Se un ID non esiste sul mio sistema (ad esempio perché i `-rawhide` non ci sono), DNF lo segnala: non è un problema, vuol dire che quella riga non serviva.

Una cosa da tenere a mente: `setopt` su un'opzione di repository, per quanto ricordo dalla documentazione di DNF5, non modifica il file `.repo` originale, ma scrive un **override** separato. Lo verifico, invece di fidarmi:

```bash
ls /etc/dnf/repos.override.d/ 2>/dev/null
cat /etc/dnf/repos.override.d/* 2>/dev/null
```

Va annotato nei miei appunti, perché un override resta lì anche dopo che il pacchetto release verrà aggiornato: se un giorno mi chiedo perché un repository è abilitato (o no), la risposta potrebbe essere in quella cartella e non nel file `.repo`.

Dopo aver corretto, anche in questo caso: `sudo dnf clean metadata` e, se serve riallineare i pacchetti, `sudo dnf distro-sync`.

### Un controllo extra sulle versioni

In caso di dubbio sull'origine di un pacchetto già installato, cerco i pacchetti costruiti per la release **successiva** a quella che ho (se sono su 44, `fc45`):

```bash
NEXT=$(( $(rpm -E %fedora) + 1 ))
rpm -qa | grep -F ".fc$NEXT"
```

`$(( ... ))` (doppia parentesi) è l'**espansione aritmetica** di bash: calcola l'espressione e inserisce il risultato. Qui prende `44`, aggiunge 1 e ottiene `45`, senza scriverlo a mano. Se il comando non stampa nulla, nessun pacchetto installato arriva da rawhide.

---

## 7. Appstream data per Discover

Solo **dopo** aver verificato che i repository sono quelli giusti, installo i dati appstream: sono i metadati che permettono a Discover di mostrare nel proprio catalogo le applicazioni presenti in RPM Fusion (nome, descrizione, icone).

```bash
sudo dnf install rpmfusion-free-appstream-data rpmfusion-nonfree-appstream-data
```

L'ordine conta per un motivo concreto: nel thread, chi aveva rawhide attivo si ritrova anche `rpmfusion-free-appstream-data` alla versione `45-1.fc45`. Installarli con i repository sbagliati significa portarsi in casa un pacchetto di Fedora 45.

Se Discover non mostra subito le nuove applicazioni, lo chiudo e lo riapro; se non basta, un refresh forzato di PackageKit:

```bash
pkcon refresh force
```

(Lo riporto come tentativo ragionevole, non come passaggio che ho verificato.)

---

## 8. Verifiche finali e checkpoint

L'obiettivo è dimostrare, con dati e non con una sensazione, che il lavoro è andato a buon fine. Raccolgo tutto in un file di checkpoint, nello stesso stile dello snapshot iniziale.

Un paio di note sul blocco qui sotto:

- `sudo -v` all'inizio chiede la password **subito**. Dentro il blocco `{ ... } > file` l'output viene ridirezionato e un eventuale prompt di `sudo` sarebbe poco chiaro: così la password è già in cache.
- `--assumeno` fa rispondere "no" a DNF alla richiesta di conferma: ottengo la **simulazione** di una transazione, senza modificare nulla.
- `|| echo "nessuno"`: se `grep` non trova niente esce con errore, e `echo` stampa una riga leggibile al posto del vuoto.

```bash
mkdir -p ~/post-install-info
CHECK=~/post-install-info/rpmfusion-checkpoint-$(date +%Y-%m-%d_%H-%M).txt
NEXT=$(( $(rpm -E %fedora) + 1 ))

sudo -v

{
  echo "=== Checkpoint RPM Fusion — $(date '+%A %d %B %Y, %H:%M:%S') ==="
  echo

  echo "--- Fedora e kernel ---"
  cat /etc/fedora-release
  echo "Kernel: $(uname -r)"
  echo

  echo "--- Pacchetti rpmfusion installati (atteso: release 44-N senza '0.') ---"
  rpm -qa 'rpmfusion*' | sort
  echo

  echo "--- Repository RPM Fusion, abilitati e non ---"
  dnf repo list --all | grep -i rpmfusion
  echo

  echo "--- Repository rawhide abilitati (atteso: nessuno) ---"
  dnf repo list | grep -i rawhide || echo "nessuno"
  echo

  echo "--- Pacchetti installati costruiti per fc$NEXT (atteso: nessuno) ---"
  rpm -qa | grep -F ".fc$NEXT" || echo "nessuno"
  echo

  echo "--- Override di DNF su repository ---"
  ls -l /etc/dnf/repos.override.d/ 2>/dev/null || echo "cartella assente o vuota"
  echo

  echo "--- Chiavi GPG ---"
  ls /etc/pki/rpm-gpg/ | grep -i rpmfusion
  rpm -qa gpg-pubkey --qf '%{NAME}-%{VERSION}-%{RELEASE}  %{SUMMARY}\n'
  echo

  echo "--- Appstream data (atteso: versione fc44, non fc$NEXT) ---"
  rpm -q rpmfusion-free-appstream-data rpmfusion-nonfree-appstream-data
  echo

  echo "--- Simulazione: distro-sync (atteso: nulla da fare) ---"
  sudo dnf distro-sync --assumeno 2>&1 | tail -n 20
  echo

  echo "--- Simulazione: risoluzione di ffmpeg completo da RPM Fusion ---"
  echo "(solo prova di risoluzione, nessuna installazione)"
  sudo dnf install ffmpeg --allowerasing --assumeno 2>&1 | tail -n 30
  echo

} > "$CHECK"

echo "Checkpoint salvato in: $CHECK"
```

### Come leggere il risultato

| Sezione del file | Esito positivo | Esito negativo |
|---|---|---|
| Pacchetti `rpmfusion*` | Release `44-N`, appstream `fc44` | `44-0.x`, oppure appstream `fc45` |
| Repo RPM Fusion | I quattro (`-free`, `-free-updates`, `-nonfree`, `-nonfree-updates`) abilitati | Disabilitati, oppure presenti solo i `-rawhide` |
| Repo rawhide abilitati | `nessuno` | Qualsiasi riga |
| Pacchetti `fc45` | `nessuno` | Qualunque pacchetto elencato |
| `distro-sync` simulato | Nulla da fare | Una lista di downgrade o pacchetti da sincronizzare |
| Risoluzione `ffmpeg` | Il riepilogo mostra pacchetti `fc44` | Errori `nothing provides ... glibc` o riferimenti a `fc45` |

L'ultima riga della tabella è il test "vero": è esattamente il tipo di errore che il problema produce. Se la simulazione si risolve senza errori e con pacchetti `fc44`, i repository stanno lavorando come devono. L'**installazione** di `ffmpeg` e dei codec è un passaggio a parte, che qui non faccio.

Per confrontare con lo stato precedente:

```bash
diff ~/post-install-info/repo-prima-rpmfusion-*.txt <(dnf repo list --all)
```

`<( ... )` è la **process substitution**: bash esegue il comando e lo presenta a `diff` come se fosse un file. Mi evita di creare un file temporaneo solo per questo confronto.

---

## 9. Riepilogo operativo

- [ ] Salvare la pagina ufficiale (PDF) e creare il file `fonti-rpmfusion-<data>.txt` in `~/post-install-info/`
- [ ] Fotografare i repository **prima**: `dnf repo list --all > ...repo-prima-rpmfusion-<data>.txt`
- [ ] Installare i due pacchetti `release` (Free e Nonfree)
- [ ] **Fermarsi e controllare**: versione `44-N` senza `0.`, quattro repo abilitati, nessun `rawhide`
- [ ] Se serve: `dnf update <URL>` + `dnf clean metadata` + `dnf distro-sync` (Caso A), oppure `config-manager setopt` (Caso B), leggendo il riepilogo della transazione
- [ ] Solo dopo, installare i pacchetti `*-appstream-data`
- [ ] Generare il file `rpmfusion-checkpoint-<data>.txt` e leggerlo
- [ ] Annotare eventuali override creati con `setopt` (cartella `/etc/dnf/repos.override.d/`)

Quando in futuro aggiungerò altri repository (ad esempio per VS Code), la parte riutilizzabile di questo capitolo è la **2** (come vedere i repository), la **8** (il checkpoint, adattabile cambiando i nomi dei repository) e il criterio: *prima* faccio la fotografia, *poi* aggiungo, *poi* verifico, *poi* archivio.
