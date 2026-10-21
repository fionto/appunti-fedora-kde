# Python di sistema con `uv`: un ambiente condiviso per data science

Su Fedora 44 ho voluto un ambiente Python "di tutti i giorni" per script, esperimenti e analisi di dati: un posto in cui ci sono già `pandas`, `numpy`, `scipy` e `matplotlib`, da riusare in più cartelle senza reinstallare tutto ogni volta. Per costruirlo uso **`uv`**, e questo capitolo racconta cosa ho fatto, in che ordine, e soprattutto perché.

È un capitolo su una scelta pratica, non su "il modo giusto" di gestire Python: l'ambiente condiviso che descrivo qui è comodo per imparare e per prove veloci, ma ha un limite preciso (non traccia le dipendenze in nessun file), di cui parlo nella sezione 5.

⚠️ **Stato di verifica.** A differenza di altri capitoli, qui i comandi **li ho eseguiti davvero** su questo sistema, e gli output riportati sono quelli reali. Quello che non ho eseguito, e che arriva da documentazione o da ragionamento, è segnato con "da verificare" o "a quanto so". Le versioni dei pacchetti sono quelle del giorno in cui ho fatto la prova: cambiano in fretta, quindi se rifaccio la procedura tra qualche mese vedrò numeri diversi.

---

## 1. Da dove arrivano le informazioni (consultato il 21 ottobre 2026)

| Fonte | A cosa mi serve |
|---|---|
| Documentazione di `uv`, pagina *Installing uv* ([docs.astral.sh/uv](https://docs.astral.sh/uv/getting-started/installation/)) | Come funziona l'installer, cosa modifica, quando `uv self update` è disponibile. L'ho letta tramite una copia del file nel repository del progetto, non dal sito. |
| Fedora Packages, scheda del pacchetto [`uv` per Fedora 44](https://packages.fedoraproject.org/pkgs/uv/uv/fedora-44.html) e advisory di aggiornamento (es. `FEDORA-2026-407b957a3b`) | Per sapere se `uv` esiste nei repository Fedora e quanto segue da vicino le release upstream. |

---

## 2. Cos'è `uv`, e dove si colloca rispetto a `dnf`

`uv` è uno strumento per gestire **progetti e ambienti Python**, sviluppato da **Astral** (gli stessi autori del linter `ruff`) e scritto in **Rust**. Il motivo per cui l'ho scelto è che raccoglie in un solo comando funzioni che di solito richiedono strumenti separati:

| Funzione | Strumento "tradizionale" | Equivalente in `uv` |
|---|---|---|
| Creare ambienti virtuali | `venv`, `virtualenv` | `uv venv` |
| Installare pacchetti | `pip` | `uv pip install` |
| Gestire versioni di Python | `pyenv` | `uv python` |
| Gestire le dipendenze di un progetto | `pip` + `requirements.txt` | `uv add` + `pyproject.toml` |
| Bloccare le versioni esatte (lockfile) | `pip-tools`, `poetry` | `uv lock` |

Sul fronte velocità, il progetto dichiara una risoluzione e un'installazione molto più rapide di `pip` (si parla di 10-100 volte): non ho fatto confronti, ma i numeri che ho visto nella sezione 8 (tredici pacchetti installati in poche centinaia di millisecondi, dopo il download) rendono l'idea. Un'altra differenza è la **cache globale**: i pacchetti scaricati una volta vengono riusati da tutti gli ambienti, invece di essere riscaricati ogni volta.

`uv pip install` ha la stessa interfaccia di `pip install`, quindi non devo disimparare niente: cambia il motore, non il modo di usarlo.

### Un'analogia con quanto già visto

In `appendice-package-manager.md` ho separato i package manager in livelli: il formato del pacchetto, il tool di basso livello, il risolutore di dipendenze, i repository. `uv` occupa il ruolo di **risolutore di dipendenze**, ma per un ecosistema diverso: non gestisce i pacchetti `.rpm` dei repository Fedora (quello è `dnf5`), gestisce i pacchetti **Python** di **PyPI**, il catalogo pubblico dove vengono pubblicati. Sono due mondi paralleli, con repository, formati e regole propri. Tenerli distinti in testa evita un errore tipico: pensare che `uv` e `dnf` si "aiutino" a vicenda. Non si parlano.

---

## 3. Perché un ambiente virtuale e non installare tutto nel sistema

Il Python di Fedora è quello che il sistema stesso usa: alcuni strumenti di sistema sono scritti in Python e dipendono dai pacchetti Python installati tramite `dnf`. Mescolarci sopra librerie installate a mano con `pip` può rompere quelle dipendenze, in modi difficili da diagnosticare. A quanto so, per questa ragione le distribuzioni recenti segnano il Python di sistema come "gestito dall'esterno" (è la convenzione PEP 668) e `pip install` fuori da un ambiente virtuale viene rifiutato. Non l'ho provato su questo sistema, quindi è un punto da verificare.

Un **ambiente virtuale** (*virtual environment*, o *venv*) risolve il problema. Un'analogia: il Python di sistema è l'attrezzatura dell'officina, usata da tutti e manutenuta dalla distribuzione; un venv è la **cassetta degli attrezzi personale** che porto sul banco da lavoro. Dentro posso aggiungere, cambiare e togliere strumenti quanto voglio senza toccare quelli dell'officina.

---

## 4. Installazione di `uv`

### 4.1 Due strade, e cosa cambia davvero

Su Fedora ci sono due modi principali:

- **Pacchetto `dnf`** (`sudo dnf install uv`): `uv` esiste nei repository Fedora 44. Si aggiorna insieme al resto del sistema.
- **Installer ufficiale** dello sviluppatore (script `curl | sh`): è il primo metodo elencato nella documentazione di `uv`. Installa il binario nella mia home, quindi non serve `sudo`.

| | Installer ufficiale | Pacchetto `dnf` |
|---|---|---|
| Dove finisce il binario | `~/.local/bin/uv`, nella mia home | Nel sistema |
| Come si aggiorna | `uv self update` | `sudo dnf upgrade` |
| `uv self update` | Disponibile | **Disabilitato**: la documentazione dice che con altri metodi di installazione va usato l'aggiornamento del package manager |
| Chi lo "possiede" | Io | RPM, che ne tiene traccia (`rpm -qf`) |

Ho scelto l'**installer ufficiale**, per il solo motivo che mi piace poter aggiornare `uv` in autonomia, indipendentemente dai tempi del pacchetto Fedora. È una preferenza personale, non una questione di correttezza: con `dnf` avrei avuto un binario tracciato da RPM e aggiornato insieme al sistema, che è un vantaggio altrettanto concreto.

Una cosa da non fare: avere **entrambi** installati. Quale dei due parte dipende dall'ordine in `$PATH`, e un giorno mi ritroverei a lanciare una versione diversa da quella che penso. Se un giorno cambio idea, prima disinstallo uno dei due; per vedere cosa risponde il sistema, `type -a uv` elenca tutte le occorrenze nel `$PATH`.

### 4.2 Il comando

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Cosa fa, pezzo per pezzo:

- `curl -LsSf <url>` scarica lo script di installazione. I flag combinati sono:
  - `-L`: segue eventuali redirect dell'URL;
  - `-s`: modalità silenziosa, niente barra di avanzamento;
  - `-S`: ma mostra comunque gli errori, se ci sono;
  - `-f`: se il server risponde con un errore HTTP, `curl` fallisce invece di consegnare la pagina d'errore come se fosse lo script.
- `| sh` (la *pipe*) prende l'output di `curl`, cioè lo script, e lo passa come input alla shell, che lo esegue.

Lo script mette il binario in `~/.local/bin/uv` e sistema il `$PATH` perché quella cartella venga trovata. Secondo la documentazione lo fa aggiornando i "profili della shell"; nel mio caso, stando al report, il file toccato è `~/.bashrc`.

⚠️ **Sicurezza.** `curl | sh` esegue codice scaricato da internet con i permessi del mio utente, **senza che io lo legga prima**. L'ho usato perché la fonte è il sito ufficiale degli sviluppatori e il metodo è quello documentato, ma è un compromesso consapevole. La documentazione stessa dice che lo script si può ispezionare prima dell'uso: basta scaricarlo in un file, leggerlo, e solo dopo eseguirlo.

```bash
curl -LsSf https://astral.sh/uv/install.sh -o /tmp/uv-install.sh
less /tmp/uv-install.sh
sh /tmp/uv-install.sh
```

(`-o` salva l'output in un file invece di stamparlo. Non è quello che ho fatto: l'ho eseguito direttamente. Lo riporto come alternativa.)

### 4.3 Far vedere il nuovo `$PATH` alla shell

Appena finita l'installazione la shell in cui sto lavorando non vede ancora `~/.local/bin`, perché `~/.bashrc` viene letto da bash **all'avvio** di una sessione. Due strade: aprire un nuovo terminale, oppure rileggere il file nella sessione corrente:

```bash
source ~/.bashrc
```

`source` esegue il contenuto del file **dentro la shell corrente** (invece di avviare un processo figlio, le cui modifiche sparirebbero alla fine): per questo le nuove variabili restano.

Per controllare cosa ha modificato l'installer, invece di fidarmi di quello che ricordo:

```bash
grep -n 'local/bin' ~/.bashrc ~/.profile ~/.bash_profile 2>/dev/null
```

(`2>/dev/null` scarta l'errore per i file che non esistono. Il comando non l'ho eseguito: è un controllo da fare.)

### 4.4 Verifica

```bash
uv --version
```

```text
uv 0.12.3 (x86_64-unknown-linux-gnu)
```

---

## 5. Ambiente condiviso o ambiente per-progetto

`uv` permette di lavorare in due modi diversi, e prima di procedere vale la pena capire quale sto usando.

**Flusso per-progetto (quello "standard" di `uv`).** Si lancia `uv init` dentro una cartella, e `uv` crea:

- un file `pyproject.toml`, l'elenco delle dipendenze del progetto;
- un file di lock, con le versioni esatte risolte;
- un ambiente virtuale `.venv` dedicato solo a quel progetto.

È l'approccio adatto quando un progetto va condiviso, versionato o ripetuto altrove, perché chiunque può ricreare lo stesso ambiente.

**Flusso a ambiente condiviso (quello che ho fatto qui).** Un solo ambiente "generalista", con i pacchetti scientifici più comuni già dentro, che uso da più cartelle e script. Per costruirlo uso i comandi di basso livello (`uv venv` e `uv pip`), non `uv init`.

Il limite da tenere presente: l'ambiente condiviso **non traccia le dipendenze in nessun file**. Se tra sei mesi voglio passare uno script a qualcuno, o rifarlo su un'altra macchina, dovrò ricostruire a mano l'elenco di cosa serve, oppure migrare quello script al flusso per-progetto. Un modo per ridurre il problema è salvare l'elenco dei pacchetti con `uv pip freeze` (vedi sezione 11), che però ancora non ho fatto.

---

## 6. Creazione dell'ambiente

Ho scelto `~/venvs/data-science`: una cartella dedicata, separata dai progetti veri e propri.

```bash
uv venv ~/venvs/data-science
```

`uv venv <percorso>` crea un ambiente virtuale in quel percorso, anche con cartelle intermedie mancanti. Dentro c'è una struttura analoga a quella di un `venv` standard (`bin/`, `lib/`, `pyvenv.cfg`), ma la creazione è molto più rapida.

```text
Using CPython 3.14.6 interpreter at: /usr/bin/python
Creating virtual environment at: venvs/data-science
Activate with: source venvs/data-science/bin/activate
```

Due cose da notare nell'output:

- `uv` ha usato **il Python di sistema** di Fedora 44 (`/usr/bin/python`, versione 3.14.6), senza scaricarne un altro.
- Il percorso è stampato in forma relativa (`venvs/data-science`) perché ho lanciato il comando dalla mia home. Il percorso reale è `~/venvs/data-science`.

### Una conseguenza che mi segno: il venv dipende da quel Python

Siccome l'ambiente è stato creato partendo dal Python di sistema, a quanto capisco il suo interprete punta a quello (un collegamento, non una copia). Quando passerò a una versione di Fedora con un Python di versione diversa, può darsi che l'ambiente smetta di funzionare e vada ricreato. È un ragionamento, non un fatto che ho verificato: per controllarlo adesso ho un comando, e lo ripeterò dopo il prossimo upgrade di Fedora:

```bash
readlink -f ~/venvs/data-science/bin/python
```

Se l'ambiente si rompe, la soluzione più semplice è ricrearlo e reinstallare i pacchetti (ragione in più per tenere l'elenco della sezione 11). `uv` può anche scaricare e gestire interpreti Python propri (`uv python`, vedi tabella nella sezione 2), che sarebbero indipendenti dalla versione di Fedora: non l'ho provato.

---

## 7. Attivazione

Finché un ambiente non è **attivato**, il terminale continua a usare il Python di sistema. Attivarlo vuol dire dire alla shell corrente: "per questa sessione, usa il Python e i pacchetti che stanno in questa cartella". È un cambiamento **temporaneo e locale alla sessione**: in un nuovo terminale devo attivarlo di nuovo.

```bash
source ~/venvs/data-science/bin/activate
```

Dentro l'ambiente c'è uno script `bin/activate` che modifica le variabili d'ambiente della shell (in particolare mette `~/venvs/data-science/bin` in testa al `$PATH`) e definisce una funzione `deactivate` per tornare indietro. Serve `source` per lo stesso motivo visto sopra: lo script deve modificare la shell **corrente**, non un processo figlio.

Si riconosce che l'ambiente è attivo dal prompt, che mostra `(data-science)` davanti al solito testo.

---

## 8. Installazione dei pacchetti

Con l'ambiente attivo, i pacchetti finiscono **dentro** quell'ambiente, e il Python di sistema non viene toccato.

```bash
uv pip install pandas numpy scipy matplotlib
```

| Pacchetto | A cosa serve |
|---|---|
| `pandas` | Dati tabellari, tramite i `DataFrame` |
| `numpy` | Calcolo numerico e array multidimensionali: è la base su cui poggiano gli altri |
| `scipy` | Algoritmi scientifici (statistica, ottimizzazione, algebra lineare) costruiti sopra `numpy` |
| `matplotlib` | Grafici e visualizzazione dei dati |

```text
Using Python 3.14.6 environment at: venvs/data-science
Resolved 13 packages in 685ms
Prepared 13 packages in 6.97s
Installed 13 packages in 149ms
 + contourpy==1.3.3
 + cycler==0.12.1
 + fonttools==4.63.0
 + kiwisolver==1.5.0
 + matplotlib==3.11.1
 + numpy==2.5.2
 + packaging==26.3
 + pandas==3.0.5
 + pillow==12.3.0
 + pyparsing==3.3.2
 + python-dateutil==2.9.0.post0
 + scipy==1.18.0
 + six==1.17.0
```

Le tre righe di riepilogo raccontano le tre fasi: **risoluzione** (calcolare cosa serve), **preparazione** (scaricare, ed è la parte lenta, 6,97 s) e **installazione** (copiare i file, 149 ms). Si vede bene che il tempo se ne va nel download e non nel lavoro di `uv`.

Ho chiesto quattro pacchetti e ne sono arrivati tredici: gli altri nove sono le **dipendenze indirette**, cioè i pacchetti di cui i quattro hanno bisogno. A grandi linee: `contourpy`, `cycler`, `fonttools`, `kiwisolver`, `pillow` e `pyparsing` servono a `matplotlib` (curve di livello, stili, font, immagini); `packaging` è un'altra sua dipendenza; `python-dateutil` serve a `pandas` per le date, e `six` a sua volta è una dipendenza di `python-dateutil`. (Nella bozza di partenza `six` era attribuito a `pandas`: è un'imprecisione, il legame passa da `python-dateutil`.)

📌 `jupyterlab` **non l'ho installato**, di proposito: voglio valutarlo più avanti, insieme a un client per notebook nativo di KDE Plasma.

---

## 9. Collegare VSCode all'ambiente

VSCode non trova da solo `~/venvs/data-science`: gli devo dire quale interprete usare. Lo faccio a livello di **cartella di progetto**, con un file `settings.json` dentro una cartella nascosta `.vscode/`.

Per il progetto di prova `~/GitRepositories/test/` ho eseguito (nella forma reale, con il percorso assoluto della mia home al posto di `<utente>`):

```bash
mkdir -p /home/<utente>/GitRepositories/test/.vscode && cat > /home/<utente>/GitRepositories/test/.vscode/settings.json << 'EOF'
{
    "python.defaultInterpreterPath": "/home/<utente>/venvs/data-science/bin/python"
}
EOF
```

Come leggerlo:

- `mkdir -p`: crea `.vscode` (con `-p` non dà errore se c'è già e crea le cartelle intermedie mancanti);
- `&&`: esegue il comando successivo solo se il precedente è andato a buon fine;
- `cat > file << 'EOF' ... EOF`: un **heredoc** (spiegato in `primi-passi.md`) che scrive il testo tra i due `EOF` nel file. Le virgolette singole attorno al primo `EOF` dicono a bash di non espandere nulla nel blocco: `$variabili` e simili restano letterali. (La bozza di partenza diceva che servivano per le parentesi graffe del JSON: non è così, le graffe dentro un heredoc non vengono interpretate comunque. Le virgolette contano per `$`, i backtick e i backslash.)

Questa stessa proprietà si può rovesciare a proprio vantaggio: se **tolgo** le virgolette (`<< EOF`), bash espande `$HOME` e il file contiene il percorso completo senza che io lo scriva a mano. È una variante che non ho eseguito:

```bash
mkdir -p ~/GitRepositories/test/.vscode && cat > ~/GitRepositories/test/.vscode/settings.json << EOF
{
    "python.defaultInterpreterPath": "$HOME/venvs/data-science/bin/python"
}
EOF
```

Dopo aver creato il file ho dovuto ricaricare la finestra di VSCode (`Ctrl+Shift+P` → *Developer: Reload Window*) perché l'interprete `data-science` comparisse nella barra di stato in basso a destra.

In alternativa, lo stesso risultato si ottiene dall'interfaccia: `Ctrl+Shift+P` → *Python: Select Interpreter* → scelgo il percorso a mano. È comodo per i progetti futuri.

⚠️ **Se la cartella è un repository Git.** Il file `.vscode/settings.json` contiene un percorso assoluto con il mio nome utente. Se non voglio finirlo in un commit, lo aggiungo a `.gitignore`. (Promemoria, non ancora fatto sul progetto di prova.)

---

## 10. Verifica finale

File `main.py`:

```python
import pandas as pd
print(pd.__version__)
```

Eseguito dal terminale integrato di VSCode:

```text
3.0.5
```

Coincide con la versione di `pandas` installata nell'ambiente (sezione 8): VSCode, il terminale e l'ambiente sono collegati. Un limite di questo test: dimostra che quel `pandas` è raggiungibile, non *quale interprete* l'ha eseguito. Un controllo più diretto, da fare la prossima volta:

```python
import sys
print(sys.executable)
```

Dovrebbe stampare `~/venvs/data-science/bin/python` (con la home espansa).

---

## 11. Comandi utili

| Comando | Cosa fa |
|---|---|
| `uv --version` | Mostra la versione di `uv` |
| `uv self update` | Aggiorna `uv` (solo con l'installer ufficiale, vedi sezione 4.1) |
| `uv venv <percorso>` | Crea un ambiente virtuale in quel percorso |
| `source <percorso>/bin/activate` | Attiva un ambiente nella shell corrente |
| `deactivate` | Disattiva l'ambiente attivo (funzione creata da `activate`) |
| `uv pip install <pacchetti>` | Installa pacchetti nell'ambiente attivo |
| `uv pip list` | Elenca i pacchetti installati nell'ambiente attivo |
| `uv pip freeze` | Elenca i pacchetti con le versioni esatte, nel formato di un `requirements.txt` (non ancora provato qui) |

---

## 12. Cose che restano aperte

- Valutare `jupyterlab` in questo ambiente o in uno dedicato, con un client notebook nativo di KDE Plasma.
- Imparare il flusso per-progetto (`uv init`, `pyproject.toml`, lockfile) per i progetti che avranno bisogno di dipendenze isolate e riproducibili.
- Eventualmente creare altri ambienti condivisi tematici (per esempio `~/venvs/networking`, `~/venvs/automation`) con lo stesso schema.
- Verificare cosa succede all'ambiente al prossimo upgrade di Fedora (sezione 6).
- Salvare un elenco dei pacchetti dell'ambiente (`uv pip freeze`) in un file, per poterlo ricreare.

---

## 13. Riepilogo operativo

- [ ] Scegliere come installare `uv` (installer ufficiale o `dnf`) e **non** averli entrambi
- [ ] Se uso l'installer: eventualmente leggere lo script prima di eseguirlo (`-o` + `less`)
- [ ] Ricaricare la shell (`source ~/.bashrc` o nuovo terminale) e verificare `uv --version`
- [ ] Creare l'ambiente: `uv venv ~/venvs/data-science`
- [ ] Attivarlo: `source ~/venvs/data-science/bin/activate` (prompt con `(data-science)`)
- [ ] Installare i pacchetti: `uv pip install pandas numpy scipy matplotlib`
- [ ] In VSCode, per ogni progetto: impostare `python.defaultInterpreterPath` (file o *Select Interpreter*) e ricaricare la finestra
- [ ] Verificare con `import pandas` e, meglio, con `sys.executable`
- [ ] Se il progetto è un repository Git, escludere `.vscode/settings.json` con `.gitignore`
- [ ] Dopo il prossimo upgrade di Fedora, controllare che l'ambiente funzioni ancora (`readlink -f ~/venvs/data-science/bin/python`)
