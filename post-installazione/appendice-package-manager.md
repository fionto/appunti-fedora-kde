# Appendice — Orientarsi tra le sigle della gestione pacchetti

`dnf`, `yum`, `rpm`, `dpkg`, `apt`, `flatpak`, `COPR`, `EPEL`, `rpm-ostree`... Se vieni da altri sistemi operativi (o anche solo da altre distro Linux), il primo scoglio su Fedora non è tecnico, è **lessicale**: un sacco di sigle che sembrano sinonimi intercambiabili ma non lo sono, perché raccontano pezzi diversi della stessa storia, stratificati nel tempo. Questa appendice prova a mettere ordine in due passaggi: prima la struttura concettuale, valida per Linux in generale; poi la storia e la situazione concreta su Fedora nel 2026.

---

## Parte A — Come è fatto, concettualmente, un "package manager"

La confusione nasce soprattutto dal fatto che "package manager" non indica *una* cosa, ma un insieme di livelli distinti che lavorano insieme. Capire questi livelli una volta per tutte rende quasi tutte le sigle auto-esplicative.

### Livello 1 — Il formato del pacchetto (il "contenitore")

Un pacchetto è, nella sua essenza, un archivio compresso che contiene: i file del programma (eseguibili, librerie, icone, file di configurazione), più dei **metadati** — nome, versione, architettura (es. `x86_64`), e soprattutto la **lista delle dipendenze** dichiarate (di cos'altro ha bisogno per funzionare), oltre a eventuali script da eseguire prima/dopo l'installazione o la rimozione.

Ogni grande famiglia di distribuzioni ha il proprio formato:
- `.rpm` — famiglia Red Hat (Fedora, RHEL, openSUSE, Mageia...)
- `.deb` — famiglia Debian (Debian, Ubuntu, Linux Mint...)
- `.pkg.tar.zst` — Arch Linux

Il formato da solo non fa nulla: è solo un contenitore. Serve qualcosa che sappia leggerlo e agire di conseguenza.

### Livello 2 — Il tool di basso livello

Questo è il programma che sa effettivamente **installare, rimuovere o interrogare** un singolo file di pacchetto già presente sul disco. Fa il suo lavoro bene, ma con un limite enorme: **non conosce internet, non sa cosa esiste "fuori"**, e soprattutto non risolve le dipendenze da solo — se il pacchetto che stai installando richiede altri tre pacchetti che non hai, questo livello si limita a dirtelo e si ferma lì, lasciando a te il compito di procurarteli (manualmente, nell'ordine giusto — un incubo noto storicamente come **dependency hell**).

Esempi: `rpm` (famiglia Red Hat), `dpkg` (famiglia Debian).

### Livello 3 — Il risolutore di dipendenze / frontend di alto livello

Questo è il livello con cui interagisci davvero nel 99% dei casi. È un programma che sa parlare con i **repository** remoti (vedi sotto), conosce l'intero grafo delle dipendenze disponibili, e quando gli chiedi "installa X" calcola da solo tutta la catena di pacchetti necessari, li scarica e li passa al tool di basso livello corrispondente per l'installazione effettiva.

Esempi: `dnf`/`dnf5` (Fedora/RHEL), `apt`/`apt-get` (Debian/Ubuntu), `zypper` (openSUSE), `pacman` (Arch — caso particolare: su Arch il tool di basso e alto livello coincidono nello stesso binario).

Qui si trova gran parte delle sigle che generano confusione, perché negli anni **lo stesso ruolo** (risolutore di dipendenze per la famiglia Red Hat) è stato occupato da strumenti diversi in momenti diversi — è il filo conduttore della Parte B qui sotto.

### Livello 4 — I repository

Un repository è semplicemente un server che ospita una collezione di pacchetti, insieme a un **indice** (metadati) che elenca cosa contiene. Il tuo sistema tiene un elenco di repository "abilitati" (su Fedora, file `.repo` dentro `/etc/yum.repos.d/` — sì, quel nome risale ancora a YUM, vedi dopo), e il risolutore di dipendenze li interroga tutti insieme quando cerca un pacchetto o le sue dipendenze.

### Livello 5 — Firma e verifica

Ogni repository "serio" firma i propri pacchetti con una chiave **GPG**, e il risolutore di dipendenze verifica quella firma prima di installare qualsiasi cosa — è il meccanismo che garantisce che un pacchetto scaricato non sia stato manomesso lungo il tragitto. È il motivo per cui, abilitando un nuovo repository di terze parti, ti viene quasi sempre chiesto di accettare/importare una chiave GPG la prima volta.

### Livello 6 — Le interfacce, grafiche e non

Sopra tutto questo si appoggiano le interfacce che effettivamente usi: il terminale (`dnf5 install ...`), oppure un **software center** grafico — su KDE Plasma è **Discover**. Discover non parla direttamente con `dnf5`: si appoggia a **PackageKit**, un livello di astrazione pensato apposta per dare a un'unica interfaccia grafica la possibilità di gestire backend diversi (pacchetti nativi via `dnf5`, ma anche Flatpak) senza dover conoscere i dettagli di ciascuno.

---

## Parte B — La storia, ramo Red Hat/Fedora: da RPM a DNF5

Con i livelli concettuali chiari, la storia dietro le sigle diventa quasi aneddotica — ma è il tipo di aneddoto che aiuta a ricordarsele.

**Metà anni '90 — il problema di partenza.** Prima dei package manager, installare software su Linux significava scaricare il codice sorgente e compilarlo a mano (`./configure && make && make install`). Funzionava, ma non c'era alcun tracciamento di cosa fosse installato, né delle dipendenze: ogni programma andava compilato sapendo già, da soli, di cos'altro avesse bisogno.

**1995 — nasce RPM.** Red Hat introduce RPM come formato di pacchetto binario precompilato più il tool per gestirlo — risolvendo il problema della compilazione manuale, ma non ancora quello delle dipendenze: RPM resta un tool di "Livello 2" nella classificazione sopra, capace di installare un pacchetto locale ma non di andare a scaricare da solo ciò che gli manca. Il nome originale, **Red Hat Package Manager**, è stato poi reinterpretato come acronimo ricorsivo — **RPM Package Manager** — quando il formato è stato adottato anche da distribuzioni diverse da Red Hat (openSUSE, Mageia...), per i quali il riferimento esplicito a "Red Hat" nel nome non aveva più senso.

**Fine anni '90/inizio 2000 — nasce YUM.** Il problema del dependency hell resta irrisolto per anni, finché il progetto **Yellow Dog Linux** (una distribuzione pensata per i Mac basati su processori PowerPC, anch'essa costruita su RPM) sviluppa uno strumento per gestire dipendenze e repository sopra RPM. Viene successivamente ripreso e modificato (da un gruppo alla Duke University), da cui il nome che è rimasto: **YUM — Yellowdog Updater, Modified**. Fedora e RHEL lo adottano come risolutore di dipendenze ufficiale, e resta lo standard per oltre un decennio. È anche l'origine del nome della cartella `/etc/yum.repos.d/`, sopravvissuto fino a oggi anche se YUM in sé non esiste più come progetto a sé stante.

**2013 — nasce DNF.** Con repository sempre più grandi, i limiti di performance e la complessità crescente del codice di YUM (scritto in Python, con un risolutore di dipendenze diventato via via difficile da mantenere) spingono un gruppo di sviluppatori Fedora a scrivere da zero un sostituto, con un nuovo motore di risoluzione dipendenze (`libsolv`). Il nome è un gioco di parole scherzoso: **DNF = Dandified YUM** ("YUM, ma elegante"). Debutta sperimentalmente in Fedora 18 (2013) e diventa il default in **Fedora 22 (2015)**; RHEL lo adotta a sua volta da RHEL 8 (2019).

**2020 in poi — nasce DNF5.** Anche DNF, pur più veloce di YUM, nel tempo accumula gli stessi sintomi: scritto in Python, comincia a mostrare limiti di performance su macchine e repository grandi, e nel frattempo erano spuntati diversi tool paralleli con comportamenti leggermente diversi (`dnf`, `yum` tenuto come alias per compatibilità, `microdnf` per gli ambienti containerizzati minimali, `dnf-automatic` per gli aggiornamenti schedulati). Nasce così **DNF5**: una riscrittura completa in **C++** (libreria `libdnf5`), pensata per essere nettamente più veloce e per **unificare** tutti quei tool paralleli in un solo binario. Disponibile sperimentalmente da Fedora 38, diventa il **default assoluto da Fedora 41 (2024)** — ed è lo stato in cui ti trovi oggi: su Fedora 44, `/usr/bin/dnf` e `/usr/bin/yum` sono semplicemente due collegamenti simbolici (`symlink`) che puntano a `dnf5`. Scrivere `yum install` o `dnf install` funziona ancora, per compatibilità con decenni di guide e abitudini, ma dietro le quinte gira sempre e solo DNF5.

Vista così, la sequenza RPM → YUM → DNF → DNF5 non è una serie di "sostituzioni casuali", ma ogni passaggio risolve un problema concreto lasciato aperto dal precedente: RPM risolve "come installo un pacchetto binario", YUM risolve "come gestisco le dipendenze e i repository", DNF risolve "come lo faccio più velocemente e in modo più mantenibile", DNF5 risolve "come consolido tutto in un solo strumento coerente e performante".

---

## Parte C — La mappa di Fedora, nel 2026

Con la storia alle spalle, ecco cosa effettivamente convive oggi su un'installazione Fedora 44 KDE Plasma come la tua, e a cosa serve ciascun pezzo.

### Il nucleo: DNF5 e dintorni
- **`dnf5`** — il risolutore di dipendenze/frontend, come visto sopra. È quello che usi nel 95% dei casi.
- **`dnf5-plugins`** — pacchetto a parte che aggiunge sottocomandi extra a `dnf5` (ad esempio `config-manager`, che hai già usato implicitamente parlando di repository) — non sempre installato di default su un'immagine minimale.
- **PackageKit** — il livello di astrazione usato dalle interfacce grafiche (vedi sotto), capace di parlare sia con `dnf5` sia con Flatpak dietro un'unica API.

### Software grafico
- **Discover** — il software center di KDE Plasma. Si appoggia a PackageKit, quindi da lì puoi gestire sia pacchetti RPM nativi sia applicazioni Flatpak nella stessa interfaccia.

### Pacchetti "universali" (cross-distro)
- **Flatpak** — formato di pacchetto pensato per funzionare identico su qualsiasi distribuzione, non solo su quelle basate su RPM. Le applicazioni girano in una forma di sandbox (isolate dal resto del sistema, con permessi granulari su cosa possono vedere/toccare) e portano con sé i propri "runtime" condivisi, invece di dipendere dalle librerie di sistema. Fedora include di base il proprio repository Flatpak (`fedora`), attivo fin dall'installazione; **Flathub**, il catalogo comunitario più grande e conosciuto, viene proposto come opzione da abilitare già durante l'installazione (o dalle impostazioni di Discover in un secondo momento) — non è detto sia già attivo di default a seconda di come hai risposto in fase di setup, vale la pena controllarlo.
- **Snap** — l'equivalente sviluppato da Canonical (l'azienda dietro Ubuntu). Tecnicamente installabile anche su Fedora, ma non fa parte dell'ecosistema nativo della distro e qui si incontra raramente.
- **AppImage** — concettualmente il più semplice di tutti: un singolo file eseguibile che contiene l'intera applicazione e le sue dipendenze. Non richiede installazione né un package manager: si scarica, lo si rende eseguibile (`chmod +x`), si lancia. In cambio non ha né risoluzione automatica delle dipendenze di sistema né un meccanismo di aggiornamento integrato (a meno di usare tool esterni pensati apposta, come AppImageLauncher).

### Repository di terze parti
- **RPM Fusion** — il repository "di complemento" più importante per Fedora: ospita tutto ciò che il progetto Fedora non può includere direttamente nei propri repository ufficiali per ragioni di licenza o brevetti (codec multimediali non liberi, driver proprietari NVIDIA, e altro ancora), diviso in una parte *free* e una *nonfree*. Va abilitato manualmente — non è attivo di default.
- **COPR** — "Cool Other Package Repo": una piattaforma di build comunitaria dove chiunque può pubblicare pacchetti non ufficiali, concettualmente simile ai PPA di Ubuntu. Utile per software recente o di nicchia non ancora nei repository ufficiali, ma senza le stesse garanzie di qualità/supporto dei repository curati da Fedora.
- **EPEL** ("Extra Packages for Enterprise Linux") — non riguarda Fedora direttamente, ma vale la pena conoscerlo perché compare spessissimo in guide che trattano l'intera famiglia Red Hat: è un repository di pacchetti aggiuntivi pensato per **RHEL** e le sue derivate (AlmaLinux, Rocky Linux), che hanno un ciclo di rilascio molto più conservativo di Fedora e quindi repository ufficiali più "scarni".

### Le varianti "immagine-based" (solo per contesto — non è il tuo caso)

Vale la pena sapere che esiste un intero ramo alternativo di Fedora, concettualmente molto diverso da quello che stai usando: le **Fedora Atomic Desktops** (nome ombrello introdotto nel 2024, che raggruppa quelle che prima venivano chiamate genericamente "immutable desktops" — **Silverblue** per GNOME, **Kinoite** per KDE, e altre varianti minori). Qui il motore di gestione pacchetti non è DNF5 ma **`rpm-ostree`**: invece di installare/rimuovere pacchetti singolarmente "in place" come fa DNF, costruisce **livelli** sopra un filesystem di base immutabile, con aggiornamenti e rollback **atomici** dell'intero sistema in blocco (o va tutto a buon fine, o non cambia nulla). Spesso si affianca a **Toolbox** o **Distrobox**, container leggeri che permettono di avere un ambiente con un package manager "normale" dentro, isolato dal sistema di base immutabile. La tua installazione (Fedora KDE Plasma "classica", con DNF5) non usa nulla di tutto questo — lo riporto solo perché è facile imbattersi in guide o discussioni online che danno per scontato quale dei due mondi si stia usando, senza specificarlo.

---

## Parte D — Glossario rapido

| Sigla / termine | Cos'è, in una riga |
|---|---|
| **RPM** | Formato di pacchetto binario + tool di basso livello (installa un singolo file `.rpm`, non risolve dipendenze da solo). Nato nel 1995 con Red Hat. |
| **YUM** | Ex risolutore di dipendenze di Fedora/RHEL (2003 circa – 2015), oggi mantenuto solo come alias verso DNF5. "Yellowdog Updater, Modified". |
| **DNF** | Successore di YUM, default Fedora dal 2015 al 2024. "Dandified YUM". Scritto in Python. |
| **DNF5** | Riscrittura in C++ di DNF, default assoluto da Fedora 41 (2024) in poi — quello che usi oggi. |
| **libdnf5** | La libreria che implementa la logica di DNF5, usata anche da altri strumenti (es. PackageKit). |
| **dnf5-plugins** | Pacchetto con sottocomandi extra per `dnf5` (es. `config-manager`). |
| **DPKG** | Equivalente di RPM nel mondo Debian/Ubuntu (tool di basso livello, formato `.deb`). |
| **APT** | Equivalente di DNF nel mondo Debian/Ubuntu (risolutore di dipendenze). |
| **Pacman** | Il package manager di Arch Linux — basso e alto livello nello stesso tool. |
| **Zypper** | Il risolutore di dipendenze di openSUSE (anch'esso basato su pacchetti `.rpm`, ma strumenti diversi da DNF). |
| **PackageKit** | Livello di astrazione usato dalle interfacce grafiche per parlare con backend diversi (DNF5, Flatpak...) senza conoscerne i dettagli. |
| **Discover** | Il software center di KDE Plasma, costruito sopra PackageKit. |
| **Repository / repo** | Server che ospita pacchetti + il loro indice (metadati). File di configurazione su Fedora: `/etc/yum.repos.d/*.repo`. |
| **GPG key** | Chiave crittografica usata per firmare e verificare l'autenticità dei pacchetti di un repository. |
| **RPM Fusion** | Repository di terze parti con tutto ciò che Fedora non può includere per licenza/brevetti (codec, driver proprietari). Non attivo di default. |
| **COPR** | Piattaforma di build comunitaria per pacchetti non ufficiali — simile ai PPA di Ubuntu. |
| **EPEL** | Repository di pacchetti extra per RHEL e derivate (AlmaLinux, Rocky Linux) — non per Fedora stessa. |
| **Flatpak** | Formato di pacchetto universale, sandboxed, cross-distro. Fedora ha un proprio repo Flatpak attivo di default; Flathub va abilitato. |
| **Flathub** | Il catalogo comunitario Flatpak più grande, non abilitato di default su ogni setup. |
| **Snap** | Formato universale concorrente di Flatpak, sviluppato da Canonical/Ubuntu — poco usato su Fedora. |
| **AppImage** | Singolo file eseguibile portatile, nessuna installazione né package manager coinvolti. |
| **rpm-ostree** | Motore di gestione pacchetti "a immagine", usato dalle Fedora Atomic Desktops (Silverblue, Kinoite...) invece di DNF5 — non riguarda la tua installazione attuale. |
| **Fedora Atomic Desktops** | Nome ombrello (dal 2024) per le varianti Fedora basate su `rpm-ostree` — Silverblue (GNOME), Kinoite (KDE), e altre. |
| **Toolbox / Distrobox** | Container leggeri per avere un ambiente con un altro package manager isolato dal sistema — utili soprattutto sulle varianti Atomic. |

---

Un'ultima osservazione, utile a livello di metodo più che di contenuto: quando una guida online parla di package manager dando per scontate sigle come queste, conviene sempre chiedersi **a che livello si riferisce** (formato? tool di basso livello? risolutore di dipendenze? repository di terze parti?) e **per quale famiglia di distribuzioni** — la stragrande maggioranza della confusione sparisce da sola una volta fatte queste due domande.
