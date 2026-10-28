# Post-installazione — Gestione dei font

Questo capitolo nasce da un dubbio molto pratico: avevo dei font "in più" nel sistema (alcuni visti in LibreOffice, come Comic Sans) e non ricordavo più da dove arrivassero: se da un pacchetto `dnf`, da un repository di terze parti, o da una copia manuale fatta mesi prima. Ricostruire la provenienza mi ha richiesto di capire dove Linux cerca i font e come le diverse strade di installazione lasciano (o non lasciano) traccia.

---

## 1. Le posizioni principali dei font

Il meccanismo che si occupa dei font su Linux si chiama **fontconfig**: è una libreria di sistema che tutte le applicazioni (da LibreOffice a Firefox, passando per KDE) interrogano per sapere quali font esistono e dove si trovano i rispettivi file. Fontconfig non guarda "ovunque sul disco": scansiona un elenco preciso di cartelle, definito nel suo file di configurazione principale, `/etc/fonts/fonts.conf` (installato dal pacchetto `fontconfig`).

Controllando la sezione che elenca le cartelle sul mio sistema:

```xml
<!-- Font directory list -->

<dir>/usr/share/fonts</dir>
<dir>/usr/local/share/fonts</dir>

<dir>/usr/share/X11/fonts/Type1</dir>
<dir>/usr/share/X11/fonts/TTF</dir>
<dir>/usr/local/share/fonts</dir>

<dir prefix="xdg">fonts</dir>
<!-- the following element will be removed in the future -->
<dir>~/.fonts</dir>
```

Qui c'è un dettaglio che vale la pena correggere rispetto a come l'avevo descritto in una prima stesura di questo capitolo: **`~/.local/share/fonts/` non compare come percorso scritto letteralmente**. Compare invece come `<dir prefix="xdg">fonts</dir>`: l'attributo `prefix="xdg"` dice a fontconfig di comporre il percorso usando la variabile d'ambiente `$XDG_DATA_HOME` (che, quando non è impostata esplicitamente, vale per convenzione `~/.local/share`) seguita da `fonts`. Il risultato pratico è lo stesso — la cartella scansionata è comunque `~/.local/share/fonts/` — ma ci si arriva tramite lo standard **XDG Base Directory**, non da un percorso hardcoded nel file.

Due altri dettagli visibili in questo estratto:

- **`~/.fonts`** è un vecchio percorso, mantenuto per compatibilità con configurazioni precedenti all'adozione dello standard XDG. Il commento nel file stesso (`<!-- the following element will be removed in the future -->`) avverte che è destinato a sparire in una futura versione di fontconfig: non è quindi il percorso su cui fare affidamento per nuove installazioni, ma spiega perché, su un sistema più datato o migrato da una vecchia configurazione, font messi lì possono ancora funzionare.
- **`/usr/share/X11/fonts/Type1`** e **`/usr/share/X11/fonts/TTF`** sono residui dell'epoca del server X11 "puro" — su un sistema Wayland/KDE moderno raramente ci si installa qualcosa manualmente, ma fontconfig continua a includerli nella scansione.

Su una Fedora KDE Plasma 44 "normale", le cartelle che contano per l'uso quotidiano restano le tre della tabella sotto — cambia solo *come* fontconfig arriva a `~/.local/share/fonts/`, non il risultato pratico:

| Cartella | Ambito | Permessi per scriverci | Chi la popola tipicamente |
|---|---|---|---|
| `/usr/share/fonts/` | Sistema — visibile a **tutti** gli utenti della macchina | `root` (serve `sudo`) | Pacchetti installati con `dnf` (ogni pacchetto font crea una propria sottocartella, es. `/usr/share/fonts/liberation/`) |
| `/usr/local/share/fonts/` | Sistema — visibile a tutti gli utenti, ma pensata per installazioni **non** gestite da un pacchetto | `root` (serve `sudo`) | Font di sistema aggiunti a mano, quando li vuoi disponibili a ogni account della macchina |
| `~/.local/share/fonts/` | Personale — visibile **solo** al tuo utente | Il tuo utente, nessun `sudo` | Font copiati a mano, strumenti grafici come "Gestione font" di KDE (kfontinst), download singoli |

Un punto che vale la pena fissare subito, perché è la radice di molta confusione quando si cerca di capire "da dove viene" un font: **fontconfig tratta queste tre cartelle allo stesso identico modo**. Scansiona tutte e tre e ti presenta un elenco unico: da un'applicazione non puoi distinguere "questo font arriva da un pacchetto gestito da `dnf`" da "questo font è stato trascinato qui a mano due anni fa". L'informazione sulla provenienza va cercata altrove (sezione 5), non nell'elenco dei font disponibili.

---

## 2. Tre modi diversi per arricchire il sistema di font

Quando voglio aggiungere un font nuovo, ho essenzialmente tre strade, con conseguenze molto diverse sul "chi se ne ricorda" nel tempo.

### 2.1 Pacchetto dei repository ufficiali Fedora

```bash
sudo dnf install liberation-fonts-all
```

I file finiscono in `/usr/share/fonts/<nome-pacchetto>/` (es. `/usr/share/fonts/liberation/`). È la strada "più pulita" nel senso gestionale: il pacchetto è **tracciato dal database RPM**, quindi in futuro posso chiedere a `rpm` quali file contiene, da dove viene, e rimuoverlo in modo pulito con `sudo dnf remove`. Segue inoltre il normale ciclo di `dnf upgrade`.

### 2.2 Pacchetto da un repository di terze parti (COPR)

```bash
sudo dnf copr enable <utente>/<progetto>
sudo dnf install <pacchetto-font>
```

Molti font piuttosto diffusi (es. JetBrains Mono, Inter, varianti dei font Microsoft) **non sono nei repository ufficiali Fedora**, ma solo su COPR — la piattaforma di build comunitaria già descritta in `appendice-package-manager.md`. Tecnicamente il risultato finale è identico al caso precedente: un file in `/usr/share/fonts/`, tracciato da RPM. La differenza è **chi cura il pacchetto**: non il progetto Fedora, ma un singolo maintainer della community, con le garanzie di qualità/supporto che questo comporta (nessuna, se non la reputazione di chi lo mantiene). Vale la stessa raccomandazione fatta per RPM Fusion: prima di abilitare un COPR, è lecito guardare chi lo mantiene e con che frequenza lo aggiorna.

### 2.3 Copia manuale in `~/.local/share/fonts/`

```bash
mkdir -p ~/.local/share/fonts/nome-font
cp percorso/ai/file/*.ttf ~/.local/share/fonts/nome-font/
```

Nessun `sudo`, nessun pacchetto, nessuna registrazione in RPM. È la strada più rapida quando un font non è pacchettizzato da nessuna parte (ad esempio font proprietari che si possiedono già come file, o font scaricati da un sito che distribuisce solo l'archivio `.ttf`/`.otf`). Il costo è esattamente quello che ho sperimentato in prima persona: **nessuno strumento di gestione pacchetti sa che quei file esistono**. Tra un anno, l'unica traccia della loro provenienza è la mia memoria (o, meglio, questi appunti).

### Confronto rapido

| | `dnf` (repo ufficiali) | `dnf` + COPR | Copia manuale |
|---|---|---|---|
| Richiede `sudo` | Sì | Sì | No |
| Visibile a tutti gli utenti | Sì | Sì | No, solo al mio utente |
| Tracciato da RPM (`rpm -qa`, `rpm -ql`) | Sì | Sì | **No** |
| Segue `dnf upgrade` | Sì | Sì | No, aggiornamento manuale |
| Rimozione pulita | `sudo dnf remove <pacchetto>` | `sudo dnf remove <pacchetto>` | Cancellazione manuale della cartella |
| Curato da | Fedora Project | Un singolo maintainer community | Me stesso |

---

## 3. Aggiornare la cache dei font dopo un'installazione

Fontconfig, per non dover riscansionare ogni file ogni volta che un'applicazione si apre (un'operazione che richiede di leggere i metadati di ogni singolo font: nome, stile, caratteri supportati), mantiene una **cache** — lo stesso concetto, concettualmente, della cache dei metadati DNF descritta in `primi-passi.md`: un indice già pronto, invece di dover rileggere tutto da zero ogni volta.

Quando un font arriva tramite `dnf` (sia da repository ufficiale sia da COPR), l'aggiornamento della cache è **automatico**: l'installazione del pacchetto RPM include uno script che la rigenera da solo. Quando invece copio font a mano in `~/.local/share/fonts/`, nessuno avvisa fontconfig: il font è lì sul disco, ma potrebbe non comparire subito nelle applicazioni finché la cache non viene aggiornata.

Il comando:

```bash
fc-cache -fv
```

- `-f` (*force*) forza la rigenerazione anche se fontconfig pensa che la cache sia già aggiornata — utile subito dopo una copia manuale, per non doversi chiedere se l'ha già notata da solo;
- `-v` (*verbose*) stampa cosa sta scansionando, cartella per cartella — comodo per confermare che la cartella nuova è stata effettivamente letta.

Due comandi di verifica che uso spesso dopo:

```bash
fc-list | grep -i "nome del font"
```

elenca tutti i font che fontconfig conosce *in questo momento*, con il percorso del file — se il font che ho appena copiato non compare qui, la cache non lo ha ancora preso, oppure il file ha un problema.

```bash
fc-match "Nome Font"
```

risponde con il font che verrebbe effettivamente usato richiedendo quel nome — utile quando sospetto che più copie dello stesso font (una di sistema, una personale) possano "fare a gara" per lo stesso nome.

---

## 4. Caso reale: cosa ho trovato in `~/.local/share/fonts/`

Per documentare la situazione di partenza, prima di qualunque pulizia, ho guardato cosa contenesse la cartella sul mio sistema:

```text
matteo@fedora-zen:~$ ls -la ~/.local/share/fonts/
totale 0
drwxr-xr-x. 1 matteo matteo  126 12 ago 17.18 .
drwxr-xr-x. 1 matteo matteo  940 25 ott 20.05 ..
drwxr-xr-x. 1 matteo matteo 1132 12 ago 08.56 aptos
drwxr-xr-x. 1 matteo matteo  902 12 ago 17.12 Bahnschrift
drwxr-xr-x. 1 matteo matteo  176 12 ago 17.12 Garamond
drwxr-xr-x. 1 matteo matteo  152 12 ago 17.12 Georgia
drwxr-xr-x. 1 matteo matteo  276 12 ago 17.12 Helvetica
drwxr-xr-x. 1 matteo matteo 1012 12 ago 17.17 msttcore
drwxr-xr-x. 1 matteo matteo  532 12 ago 17.12 Segoe-UI
drwxr-xr-x. 1 matteo matteo  136 12 ago 17.12 Verdana
```

Otto sottocartelle, tutte con data di modifica dello stesso pomeriggio (12 agosto) — segno di un'unica sessione di copia manuale, probabilmente seguita a uno script o a una guida per "avere i font Microsoft su Linux". Nessuna di queste risulta tracciata da RPM, per il motivo spiegato nella sezione 1: è lo spazio personale, fuori dalla gestione pacchetti per definizione.

### Il dettaglio che ha risolto il mistero di Comic Sans

La cartella `msttcore` aveva un nome che non era casuale. Guardandone il contenuto:

```text
matteo@fedora-zen:~$ ls ~/.local/share/fonts/msttcore/
andalemo.ttf  arial.ttf     calibri.ttf   cambriaz.ttf  candaraz.ttf  consolai.ttf  constani.ttf  corbeli.ttf  courbi.ttf  tahoma.ttf   times.ttf     trebuc.ttf
arialbd.ttf   ariblk.ttf    calibriz.ttf  candarab.ttf  comicbd.ttf   consola.ttf   constan.ttf   corbel.ttf   couri.ttf   timesbd.ttf  trebucbd.ttf  webdings.ttf
arialbi.ttf   calibrib.ttf  cambriab.ttf  candarai.ttf  comic.ttf     consolaz.ttf  constanz.ttf  corbelz.ttf  cour.ttf    timesbi.ttf  trebucbi.ttf
ariali.ttf    calibrii.ttf  cambriai.ttf  candara.ttf   consolab.ttf  constanb.ttf  corbelb.ttf   courbd.ttf   impact.ttf  timesi.ttf   trebucit.ttf
```

`comic.ttf` e `comicbd.ttf` — ecco da dove arrivava Comic Sans in LibreOffice. Ricostruendo con la cronologia della shell (`history | grep -i font`), ho trovato che in passato avevo effettivamente eseguito:

```bash
sudo dnf install msttcorefonts
sudo dnf install liberation-fonts-all
```

Controllando però lo stato attuale del pacchetto:

```text
matteo@fedora-zen:~$ rpm -ql msttcorefonts
il pacchetto msttcorefonts non è stato installato
```

Il pacchetto non c'è più — a un certo punto deve essere stato rimosso, probabilmente preferendo tenere una copia manuale indipendente dal gestore pacchetti. Il risultato è una cartella `msttcore/` "orfana": contiene font reali, funzionanti, usati ogni giorno da LibreOffice — ma **nessun comando di gestione pacchetti sa che esiste**. L'ho confermato con:

```bash
rpm -qf ~/.local/share/fonts/msttcore/comic.ttf
# → file ... non è di nessun pacchetto
```

`liberation-fonts-all`, invece, essendo tuttora installato via `dnf`, resta regolarmente tracciato (vedi sezione successiva) e aggiornato insieme al resto del sistema.

### Lezione pratica per la documentazione

Il nome della cartella (`msttcore`, non casuale) è stato l'indizio decisivo. Se in futuro copio font a mano, ha senso **nominare la cartella in modo che ricordi la provenienza** (es. `ms-corefonts-manuale/` invece di un nome generico), proprio perché è l'unico "metadato" che mi resterà una volta che il pacchetto RPM originale, se mai c'è stato, sarà sparito.

---

## 5. Verificare quali font sono effettivamente pacchetti RPM

Il comando diagnostico di base, per vedere cosa il sistema di gestione pacchetti sa davvero:

```bash
rpm -qa | grep -i font | sort
```

Spiegato pezzo per pezzo:

- `rpm -qa` (*query all*) elenca **tutti** i pacchetti installati sul sistema, uno per riga, senza filtri — è l'elenco completo del database RPM;
- `grep -i font` filtra solo le righe che contengono la stringa `font`, **ignorando maiuscole/minuscole** (`-i`) — necessario perché alcuni pacchetti usano `Font`, altri `font`, altri ancora `fonts`;
- `sort` ordina il risultato alfabeticamente, per leggerlo più comodamente (altrimenti l'ordine è quello interno del database, non prevedibile).

Sul mio sistema, questo comando mostrerebbe `liberation-fonts-*` e qualunque altro pacchetto font installato via `dnf` o COPR — ma **non** mostrerebbe nulla delle otto cartelle in `~/.local/share/fonts/`, per il motivo già visto: quei file non appartengono a nessun pacchetto.

### Il resto della "cassetta degli attrezzi" diagnostica

Per ricostruire la cronologia completa (non solo "cosa c'è ora", ma "cosa è successo nel tempo"), gli altri comandi che mi sono serviti:

| Comando | A cosa serve |
|---|---|
| `rpm -ql <pacchetto>` | Elenca i file contenuti in un pacchetto specifico già installato — utile per sapere esattamente dove un pacchetto ha copiato i suoi font |
| `rpm -qf <percorso-file>` | Il contrario: dato un file, dice a quale pacchetto appartiene (o conferma che non appartiene a nessuno) |
| `dnf repoquery --qf '%{from_repo}' <pacchetto>` | Dice da quale repository (Fedora ufficiale, COPR, ecc.) è arrivato un pacchetto |
| `dnf copr list` | Elenca i repository COPR attualmente abilitati sul sistema |
| `sudo dnf history list` / `sudo dnf history info <ID>` | Cronologia delle transazioni DNF — utile per ritrovare *quando* un pacchetto è stato installato o rimosso |
| `grep -i font ~/.bash_history` / `history \| grep -i font` | Cronologia della shell — funziona solo se non è mai stata ripulita e se i comandi sono stati lanciati da quell'utente/terminale |

Nessuno di questi, da solo, racconta la storia intera — è stata proprio la combinazione (cronologia shell → stato RPM → contenuto delle cartelle) a chiarire la situazione in questo caso.

---

## 6. Font che potrebbero arricchire il sistema base

Oltre a quelli già presenti (Liberation Fonts, Google Noto — quest'ultimo è il font di default su Fedora 44 per buona parte delle lingue, già installato senza bisogno di fare nulla), alcuni font utili per casi d'uso specifici. Per ciascuno indico se è un pacchetto **ufficiale Fedora**, disponibile solo via **COPR**, o da installare **manualmente** — la stessa distinzione fatta nella sezione 2, perché è quella che decide quanto sarà facile mantenerli nel tempo.

### Per la scrittura di codice / terminale

| Font | A cosa serve | Disponibilità |
|---|---|---|
| **Fira Code** | Monospace con legature per sequenze comuni in programmazione (`->`, `!=`, `==`...) | **Ufficiale**: `sudo dnf install fira-code-fonts` |
| **JetBrains Mono** | Monospace pensato per leggibilità del codice, molto diffuso (default in IDE JetBrains, VS Code) | Non ufficiale: solo COPR (es. `elxreno/jetbrains-mono-fonts`) |
| **Nerd Fonts** | Varianti "patchate" di font esistenti con icone aggiuntive — servono per prompt di shell decorati (Starship, Powerlevel10k) o barre di stato con icone (Waybar) | Nessun pacchetto Fedora ufficiale: download manuale da [nerdfonts.com](https://www.nerdfonts.com) in `~/.local/share/fonts/` |

### Per l'interfaccia grafica

| Font | A cosa serve | Disponibilità |
|---|---|---|
| **Google Noto Sans/Serif** | Copertura Unicode molto ampia (alfabeti non latini, emoji inclusi) | **Ufficiale**, e in buona parte già presente di default su Fedora 44 (`dnf list installed 'google-noto*'` per vedere cosa c'è già) |
| **Inter** | Sans-serif pensato per schermo, molto usato in interfacce moderne | ⚠️ Non ufficiale su Fedora (ho trovato solo COPR, es. `burhanverse/inter-fonts`; una richiesta di pacchettizzazione ufficiale risulta discussa nelle liste Fedora ma non confermata come conclusa — da riverificare) |

### Per documenti e typesetting

| Font | A cosa serve | Disponibilità |
|---|---|---|
| **Latin Modern / TeX Gyre** | Le famiglie "storiche" di LaTeX (Computer Modern e i suoi derivati ridisegnati) — utili per coerenza visiva tra documenti LaTeX e documenti non-LaTeX (LibreOffice, presentazioni) | **Ufficiale**: incluso nel pacchetto `texlive-collection-fontsrecommended` (se ho già un'installazione TeX Live) |

### Un comando utile per esplorare da solo

Prima di cercare su internet se un font è pacchettizzato, vale la pena provare direttamente:

```bash
dnf search font | grep -i "nome che cerco"
```

Mostra solo cosa è disponibile nei repository **già abilitati** sul mio sistema (quindi non COPR non ancora attivati) — ma è il primo posto dove guardare, prima di aprire il browser.

### Il criterio che mi do

Coerentemente con quanto scritto in `appendice-package-manager.md` a proposito dei repository in generale: preferisco i pacchetti **ufficiali Fedora** quando coprono il mio caso d'uso (tracciati, aggiornati, rimovibili puliti), accetto un **COPR** quando non c'è alternativa ma il maintainer è affidabile, e riservo la **copia manuale** ai casi in cui non esiste proprio un pacchetto — accettando consapevolmente che, come visto nella sezione 4, è la strada che lascia meno tracce di sé nel tempo.

---

## 7. Riepilogo operativo

- [ ] Sapere distinguere le tre cartelle: `/usr/share/fonts/` (sistema, dnf), `/usr/local/share/fonts/` (sistema, manuale), `~/.local/share/fonts/` (utente, manuale)
- [ ] Prima di copiare font a mano, controllare se esiste già un pacchetto Fedora o COPR equivalente (`dnf search font`)
- [ ] Dopo ogni copia manuale: `fc-cache -fv`, poi verificare con `fc-list` o `fc-match`
- [ ] Per sapere se un font visto in un'applicazione è un pacchetto o una copia manuale: `fc-list | grep -i <font>` per il percorso, poi `rpm -qf <percorso>`
- [ ] Per un controllo generale dei pacchetti font installati: `rpm -qa | grep -i font | sort`
- [ ] Se la provenienza non è chiara: incrociare `rpm -q`, `dnf history list`, `dnf copr list` e cronologia della shell
- [ ] Nominare le cartelle di font copiati a mano in modo che ricordino la provenienza, non genericamente
