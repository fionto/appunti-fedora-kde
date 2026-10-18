# Post-installazione — Parte 4: driver grafici e accelerazione video

Dopo RPM Fusion (`rpm-fusion.md`) e i codec (`codec-multimediali.md`), il pezzo che manca per avere una riproduzione video "davvero" a posto è la **grafica**: capire quali driver usa il sistema, perché su Linux non si scarica "il driver dal sito del produttore" come su Windows, e come far sì che la decodifica H.264/H.265 la faccia la **GPU** invece della CPU.

Questo capitolo parte da una guida in inglese che avevo seguito. Il suo scheletro era giusto, ma su Fedora 44 alcuni comandi sono **superati** (sezione 5) e la parte di verifica si fermava a un solo comando, `vainfo`, che da solo non dimostra quello che sembra dimostrare (sezioni 6 e 7). Il capitolo è quindi più lungo dell'originale soprattutto nelle parti di verifica.

⚠️ **Prerequisiti:** repository RPM Fusion verificati (release `44-N`, nessun `rawhide`) e `ffmpeg` completo già installato come descritto in `codec-multimediali.md`. Alcuni test di questo capitolo usano proprio quell'`ffmpeg`.

---

## 1. Da dove arrivano queste informazioni (consultato il 18 ottobre 2026)

| Fonte | A cosa mi serve |
|---|---|
| [Fedora Wiki — Hardware Video Acceleration](https://fedoraproject.org/wiki/Hardware_Video_Acceleration) | Riferimento della distribuzione per l'accelerazione video (già citato in `codec-multimediali.md`). |
| [Fedora Wiki — Firefox Hardware acceleration](https://fedoraproject.org/wiki/Firefox_Hardware_acceleration) | Come controllare lo stato della decodifica hardware in Firefox su Fedora (`about:support`, riga `HARDWARE_VIDEO_DECODING`). |
| [LinuxCapable — How to Install AMD Radeon Drivers on Fedora 44](https://linuxcapable.com/how-to-install-amd-radeon-drivers-on-fedora-linux/) | Procedura aggiornata per Fedora 44: installazione diretta di `mesa-va-drivers-freeworld`, niente `swap`. |
| [Fedora Packages — `mesa-dri-drivers` per Fedora 44](https://packages.fedoraproject.org/pkgs/mesa/mesa-dri-drivers/fedora-44-updates.html) | Elenco dei "Provides" del pacchetto: contiene `mesa-va-drivers` (vedi sezione 5). |
| Changelog di `mesa-vulkan-drivers-freeworld` nel repository RPM Fusion (voce del 16/10/2025) | La nota "Follow Fedora and drop VDPAU support". |
| [Phoronix — Firefox 136 Beta](https://www.phoronix.com/news/Firefox-136-Beta) | Dal 2025 Firefox abilita di default la decodifica hardware su GPU AMD in Linux. |

⚠️ **Limite di queste verifiche.** Quello che scrivo sulla procedura per Fedora 44 si appoggia quindi su guide di terzi e sui metadati dei pacchetti, **non** sulla pagina ufficiale. Prima di eseguire qualcosa vale la pena riconfrontarla a mano, e per questo la procedura della sezione 5 passa sempre da una **simulazione**.

---

## 2. Come Linux (e Fedora) gestiscono la grafica, rispetto a Windows

### 2.1 Su Windows: un pacchetto del produttore

Su Windows, il "driver della scheda video" è di solito **un unico pacchetto** scaricato dal produttore (o da Windows Update): AMD, NVIDIA o Intel distribuiscono un installer che porta con sé tutto: il componente che gira nel kernel, la libreria che implementa DirectX/OpenGL/Vulkan, i blocchi per la decodifica video, il pannello di controllo. È un prodotto **chiuso e proprietario**, aggiornato con il calendario del produttore, indipendente dalla versione di Windows.

### 2.2 Su Linux: uno stack a strati, quasi tutto già nel sistema

Su Linux il quadro è diverso, perché la grafica è un **insieme di livelli** che hanno proprietari e ritmi di aggiornamento diversi. Un'analogia: su Windows il driver è un'auto venduta con motore, cambio e carrozzeria come blocco unico dal costruttore; su Linux motore, cambio e carrozzeria sono componenti standardizzati, ognuno con il suo manutentore, e la distribuzione li assembla e li tiene allineati.

| Livello | Cosa fa | Chi lo fornisce, nel mio caso (AMD) |
|---|---|---|
| **Firmware della GPU** | Piccoli programmi caricati sulla GPU all'avvio (la GPU non funziona senza) | Pacchetto Fedora (`amd-gpu-firmware`, derivato da `linux-firmware`) |
| **Driver del kernel** | Parla con l'hardware, gestisce memoria video, risoluzione, scheda in sé. Si chiama **DRM** (Direct Rendering Manager) e, per l'impostazione dello schermo, **KMS** (Kernel Mode Setting). Per AMD il modulo è **`amdgpu`** | È **dentro il kernel Linux** (upstream), quindi arriva con il kernel di Fedora |
| **Driver in user space: Mesa** | Implementa le API che i programmi usano: OpenGL, Vulkan, VA-API (video)… | Pacchetti Fedora (`mesa-*`) |
| **Compositor / display server** | Disegna le finestre (su Plasma 6: KWin, tipicamente su Wayland) | KDE, via Fedora |
| **Applicazioni** | Chiedono a Mesa di fare cose | Qualsiasi programma |

Tre conseguenze pratiche, tutte diverse da Windows:

1. **Per le GPU AMD e Intel non si scarica niente dal sito del produttore.** Il driver del kernel e Mesa sono già nell'installazione e si aggiornano con `dnf upgrade`, insieme al resto del sistema. L'unico caso in cui serve qualcosa "in più" per l'accelerazione video riguarda i **codec con brevetti** (vedi sezione 3).
2. **Versioni legate a quelle della distribuzione.** Non scelgo io "la versione del driver": Fedora aggiorna Mesa e il kernel a ogni ciclo. Se un giorno un bug è risolto in una versione più recente di Mesa, la ricevo con gli aggiornamenti di Fedora, non da un installer.
3. **NVIDIA è il caso a parte.** Il suo driver proprietario non è nel kernel né in Mesa: va aggiunto (da RPM Fusion) e compila moduli per il kernel in uso. È il motivo per cui in `primi-passi.md` consiglio di riavviare sul kernel nuovo *prima* di compilare qualcosa. Sul mio Zenbook, con sola GPU AMD integrata, **questa parte non mi riguarda**.

### 2.3 Dove entra in gioco il video

Per il video c'è un livello in più da distinguere. Le GPU moderne hanno, accanto alle unità che disegnano la grafica 3D, un **blocco dedicato alla decodifica/codifica video** (su AMD si chiama VCN, *Video Core Next*). È hardware fatto apposta per quel lavoro: consuma molto meno di una CPU che decodifica via software.

Per usarlo, le applicazioni non parlano direttamente col blocco hardware, ma attraverso un'**interfaccia standard**:

- **VA-API** (*Video Acceleration API*): l'interfaccia che si usa oggi su Linux per AMD e Intel. È quella di questo capitolo.
- **VDPAU**: un'interfaccia più vecchia, nata per NVIDIA, oggi quasi superata da VA-API. Il motivo per cui la nomino è che la guida originale la includeva: su Fedora 44 non serve più (sezione 5).

La catena completa, per un video H.264 in Firefox o in un lettore, è quindi:

```text
Applicazione → libreria VA-API (libva) → driver VA-API dentro Mesa → driver kernel amdgpu → blocco VCN
```

Se **uno qualsiasi** di questi anelli manca, l'applicazione ripiega silenziosamente sulla decodifica software (CPU). Per questo le verifiche sono importanti: il video si vede comunque, ma sto usando la CPU senza accorgermene.

---

## 3. Mesa: cos'è, e in cosa è diverso dal "driver Windows"

### 3.1 Cos'è

**Mesa** è una raccolta open source di driver grafici in user space. Un modo per vederlo: la GPU parla un suo dialetto interno, i programmi parlano linguaggi standard (OpenGL, Vulkan, VA-API); Mesa è l'**interprete** che traduce dagli uni all'altro. Un'unica libreria contiene più "interpreti", uno per famiglia di hardware.

Per una GPU AMD moderna, i pezzi di Mesa che mi interessano sono questi:

| Componente | A cosa serve | Pacchetto Fedora |
|---|---|---|
| **radeonsi** | Driver **OpenGL** per AMD (e, nello stesso modulo, la parte **video**) | `mesa-dri-drivers` |
| **RADV** | Driver **Vulkan** per AMD | `mesa-vulkan-drivers` |
| **Driver VA-API** (`radeonsi_drv_video.so`) | La parte che espone la decodifica video a VA-API | Dentro `mesa-dri-drivers` (vedi sotto) |

### 3.2 Differenze con il driver Windows

| | Windows | Linux (AMD) |
|---|---|---|
| Distribuzione | Installer del produttore | Pacchetti della distribuzione (`dnf`) |
| Aggiornamenti | Calendario del produttore | Ciclo di Fedora (kernel + Mesa) |
| Licenza | Proprietario | Open source (kernel `amdgpu` + Mesa) |
| Componenti | Un unico pacchetto | Firmware + modulo del kernel + Mesa, pacchetti separati |
| Codec con brevetti | Inclusi (licenze gestite dal produttore/sistema) | Dipende dalla distribuzione: Fedora li **disattiva in Mesa**, RPM Fusion li fornisce |
| Pannello di controllo | App del produttore | Impostazioni di KDE; niente "pannello AMD" |

Un'ultima cosa che sembra un dettaglio ma non lo è: **non c'è un "driver AMD" da aggiornare separatamente dal sistema**. Se leggendo guide generiche trovi riferimenti a driver proprietari AMD da installare a parte, per l'uso che ne faccio qui non servono. La guida originale lo diceva a ragione ("non servono driver proprietari né moduli kernel esterni").

### 3.3 Perché Fedora toglie H.264/H.265 da Mesa

Mesa upstream supporta la decodifica H.264/H.265 via VA-API. Fedora, per la stessa ragione di brevetti/licenze per cui fornisce `ffmpeg-free` (vedi `codec-multimediali.md`), **compila Mesa con quei codec disattivati**. Il risultato è curioso: la GPU potrebbe decodificare H.264 perfettamente, ma il driver nel pacchetto Fedora non espone quella funzione a VA-API.

RPM Fusion distribuisce una variante dello stesso driver con quei codec attivi, chiamata **`mesa-va-drivers-freeworld`** ("freeworld" = per chi non ha i vincoli legali di Fedora). Il pacchetto non sostituisce Mesa: aggiunge un driver VA-API "completo" accanto a quello di Fedora.

I formati liberi (VP9, AV1) in linea di principio non hanno bisogno di niente di questo; che siano effettivamente decodificabili in hardware dipende però dalla **generazione del blocco video** della GPU, ed è `vainfo` (sezione 6) a dirmelo, non la teoria.

---

## 4. Prima di toccare qualcosa: quale GPU ho, e come leggo i risultati

La guida originale proponeva solo:

```bash
lspci | grep -E "VGA|3D"
```

Funziona, ma non spiega come leggere il risultato e ha un limite: non intercetta la classe `Display`, che alcune GPU (soprattutto integrate o ibride) usano al posto di `VGA`. Meglio una forma più completa:

```bash
lspci | grep -E "VGA|3D|Display"
```

oppure

```bash
lspci -nnk 2>/dev/null | grep -A3 -E "VGA|3D|Display"
```

(`2>/dev/null` scarta i messaggi di errore di `lspci`: serve per un avviso innocuo spiegato nella sezione 4.2.)

### 4.1 Cosa fanno i flag

- `lspci` elenca i dispositivi collegati al bus PCI (la GPU, anche se integrata, appare come dispositivo PCI). Viene dal pacchetto `pciutils`.
- `-nn` aggiunge, oltre ai nomi, i **codici numerici** in parentesi quadre: `[classe]` e `[vendor:device]`.
- `-k` aggiunge, per ogni dispositivo, il **driver del kernel** in uso e i moduli che potrebbero gestirlo.
- `grep -A3 -E "..."` filtra le righe che contengono una delle tre parole e stampa anche le **3 righe successive** (`-A3` = *after*), che sono proprio quelle con driver e moduli.

### 4.2 Come leggere il risultato

Questo è l'output reale sul mio Zenbook (ma *dopo* il processo di installazione del driver):

```text
pcilib: Error reading /sys/bus/pci/devices/0000:00:08.3/label: Operation not permitted
63:00.0 Display controller [0380]: Advanced Micro Devices, Inc. [AMD/ATI] Krackan [Radeon 840M / 860M Graphics] [1002:1114] (rev c2)
        Subsystem: ASUSTeK Computer Inc. Device [1043:1194]
        Kernel driver in use: amdgpu
        Kernel modules: amdgpu
```

**La riga `pcilib: Error reading ...`** non fa parte del risultato e non riguarda la GPU. Viene da `lspci` e finisce nel flusso degli errori (*stderr*), che il `grep` non filtra: per questo compare comunque a schermo. Il dispositivo citato, `00:08.3`, ha un indirizzo diverso da quello della GPU (`63:00.0`). A quanto capisco, `lspci` prova a leggere un attributo `label` del dispositivo e il kernel non lo concede a un utente normale; non ho verificato la causa esatta. L'avviso non cambia le righe che mi interessano, quindi si può ignorare o silenziare con `2>/dev/null`. Lo svantaggio di `2>/dev/null` è che nasconderebbe anche eventuali errori veri di `lspci`: se l'output della GPU manca del tutto, lo rilancio senza.

**Lettura riga per riga:**

- **`Display controller [0380]`**: la GPU compare come classe `Display`, non `VGA`. È proprio il caso per cui ho aggiunto `Display` al filtro: con il comando della guida originale (`VGA|3D`) questa riga non sarebbe comparsa.
- **`Krackan [Radeon 840M / 860M Graphics]`**: `Krackan` è il nome in codice del chip. Il nome commerciale è doppio perché lo stesso chip è venduto in due configurazioni; `lspci` non dice quale sia la mia. La guida originale parlava di Radeon 860M e per il mio processore è plausibile, ma lo confermo con `glxinfo -B` (sezione 4.3), non da qui.
- **`[1002:1114]`**: vendor `1002` = AMD/ATI, device `1114`.
- **`Subsystem: ASUSTeK Computer Inc. [1043:1194]`**: il produttore del portatile (ASUS = `1043`), non della GPU.
- **`Kernel driver in use: amdgpu`**: la riga che conta di più. La GPU è gestita dal driver giusto.

Esito: una sola GPU AMD, driver `amdgpu` in uso, nessuna GPU dedicata. Posso passare ai passaggi successivi.

Si legge così:

| Pezzo | Significato |
|---|---|
| `VGA compatible controller [0300]` | La **classe** del dispositivo. `0300` = VGA, `0302` = 3D controller, `0380` = Display controller. |
| `[AMD/ATI]` / `Intel Corporation` / `NVIDIA Corporation` | Il produttore. |
| `[1002:XXXX]` | `vendor:device`. Il vendor è fisso: **`1002` = AMD/ATI**, **`10de` = NVIDIA**, **`8086` = Intel**. Il secondo numero identifica il chip specifico, utile per cercare bug o guide. |
| `Kernel driver in use: amdgpu` | **È la riga che conta di più.** Dice quale modulo del kernel sta gestendo la GPU *adesso*. |

Differenza tra `VGA` e `3D`: su un portatile con GPU integrata + GPU dedicata (non è il mio caso) si vedrebbero tipicamente **due** voci: la GPU integrata come `VGA compatible controller` e quella dedicata come `3D controller`. Ritrovarsi due voci non è un errore, è la configurazione "ibrida". Sul mio Zenbook mi aspetto **una sola voce AMD**.

Se invece `Kernel driver in use` mostra `nouveau` (NVIDIA libero) o `simpledrm`/`vesa`, la GPU non sta usando il driver giusto: è un segnale da indagare *prima* di passare ai codec.

### 4.3 Altre verifiche utili

Nessuna di queste serve da sola, ma insieme completano il quadro. Le ordino dalla più "di base" alla più "di dettaglio":

```bash
# 1. il modulo del kernel è caricato?
lsmod | grep amdgpu

# 2. il kernel ha trovato la GPU e caricato il firmware? (cerco eventuali errori)
journalctl -k -b | grep -i amdgpu | head -n 30

# 3. i "nodi" DRM esposti al sistema (i punti d'accesso che le applicazioni aprono)
ls -l /dev/dri/

# 4. cosa dice Mesa, lato OpenGL (serve il pacchetto che fornisce glxinfo)
glxinfo -B

# 5. e lato Vulkan (pacchetto vulkan-tools)
vulkaninfo --summary
```

Come leggerli:

- **`lsmod`**: se compare `amdgpu` il driver del kernel è attivo (di per sé non mi dice che funzioni bene).
- **`journalctl -k -b`**: `-k` = solo messaggi del kernel, `-b` = solo dall'ultimo avvio. Cerco in particolare righe con `error`, `failed`, o richieste di firmware non trovato.
- **`/dev/dri/`**: contiene `card0` (o simile) e `renderD128`. Il secondo è il **render node**, quello che le applicazioni usano per fare calcoli/decodifica senza passare dallo schermo; è anche quello che passerò a `ffmpeg` nei test. Se ci sono più GPU compaiono più render node (`renderD128`, `renderD129`…).
- **`glxinfo -B`** (`-B` = riepilogo breve): le righe `OpenGL renderer string` e `OpenGL version string` indicano quale driver sta disegnando. Per AMD con Mesa mi aspetto di leggere un nome che comincia con `AMD Radeon ...` e, nella versione, la parola `Mesa`. Se compare `llvmpipe`, sto usando il **rendering software**: nessuna accelerazione grafica attiva.
- **`vulkaninfo --summary`**: lo stesso controllo per Vulkan; compare il nome della GPU e, in genere, il driver `radv`.
- **Interfaccia grafica**: in KDE, anche *Impostazioni di sistema → Informazioni su questo sistema → Visualizza altre informazioni* (o l'app **Info Center**) mostra la GPU in uso.

Se un comando non esiste, DNF di solito suggerisce il pacchetto da installare; in alternativa `dnf provides '*/glxinfo'` lo trova.

---

## 5. Installazione dei driver VA-API: cosa è cambiato rispetto alla guida

### 5.1 I comandi della guida originale

```bash
sudo dnf swap mesa-va-drivers mesa-va-drivers-freeworld --allowerasing
sudo dnf swap mesa-vdpau-drivers mesa-vdpau-drivers-freeworld --allowerasing
```

Erano la procedura standard fino a Fedora 42/43. Su Fedora 44 **non li uso**, per due ragioni indipendenti:

**1. `mesa-va-drivers` non è più un pacchetto separato.** Dalla pagina di Fedora Packages per Fedora 44, il pacchetto `mesa-dri-drivers` elenca tra i suoi *Provides* anche `mesa-va-drivers`: il vecchio pacchetto è stato **incorporato** in `mesa-dri-drivers`. Questo vuol dire che il vecchio `swap` non ha più un pacchetto da "togliere". Non so con certezza cosa farebbe DNF 5: nella peggiore ipotesi proverebbe a rimuovere `mesa-dri-drivers` (cioè il driver OpenGL del sistema) per soddisfare il conflitto. Non voglio scoprirlo sul mio sistema. La guida di LinuxCapable per Fedora 44 dice esplicitamente di **non** usare più lo `swap`, perché il pacchetto RPM Fusion installa i suoi file in una directory separata (`dri-freeworld`) *accanto* ai file di Fedora, e quindi la forma corretta è l'installazione diretta.

**2. VDPAU non esiste più.** Dalle note di rilascio dei pacchetti RPM Fusion: nell'ottobre 2025 il manutentore scrive "Follow Fedora and drop VDPAU support" (nel changelog di `mesa-vulkan-drivers-freeworld`). Il secondo `swap` della guida riguarda quindi un componente che non fa più parte né di Fedora né di RPM Fusion nelle versioni recenti. Per l'uso che faccio (Firefox, mpv, ffmpeg su AMD), VA-API è l'unica interfaccia che mi serve.

Questo è il tipo di errore tipico delle guide copiate da una versione all'altra di Fedora (ne parlo anche in `nuovo-in-fedora-44.md`): il comando era corretto quando è stato scritto.

⚠️ Ho verificato questo cambiamento su due guide di terzi e sui dati dei pacchetti, non sulla pagina ufficiale di RPM Fusion (vedi limite nella sezione 1). La simulazione qui sotto serve proprio a coprire il rischio.

### 5.2 Stato di partenza

Prima di installare, fotografo lo stato (come per RPM Fusion e codec). Se non ho ancora creato il file di checkpoint della sezione 7, lo lancio ora con `ETICHETTA=prima`. In più:

```bash
dnf list --installed 'mesa*' 'libva*'
rpm -q mesa-va-drivers-freeworld
```

Quest'ultimo, a questo punto, dovrebbe rispondere `package mesa-va-drivers-freeworld is not installed`.

### 5.3 Simulazione, poi installazione

```bash
sudo dnf install mesa-va-drivers-freeworld --assumeno
```

Nel riepilogo cerco tre cose:

- il pacchetto `mesa-va-drivers-freeworld` arriva da **`rpmfusion-free`** o **`rpmfusion-free-updates`**;
- **nessuna rimozione** (`Removing`) di pacchetti Mesa: se compare `mesa-dri-drivers` tra i pacchetti da rimuovere, mi fermo;
- la versione di Mesa del pacchetto corrisponde a quella di `mesa-dri-drivers` installato (vedi nota sotto).

Se è tutto in ordine:

```bash
sudo dnf install mesa-va-drivers-freeworld
```

(Come per il resto, con `defaultyes=True` in `dnf.conf` il riepilogo va comunque letto: la conferma è automatica.)

**Una nota sulle versioni.** Il pacchetto RPM Fusion richiede la **stessa versione di Mesa** di quella che Fedora ha installato (per la precisione, quella di `mesa-filesystem`). Può capitare, per qualche giorno dopo un aggiornamento di Mesa in Fedora, che RPM Fusion sia indietro e l'aggiornamento fallisca o resti bloccato. È un fenomeno documentato da anni nel bugtracker di RPM Fusion e non è un mio errore. Se succede, il rimedio indicato dalla guida LinuxCapable è rimuovere il pacchetto freeworld, aggiornare, e reinstallarlo:

```bash
sudo dnf remove mesa-va-drivers-freeworld
sudo dnf upgrade --refresh
sudo dnf install mesa-va-drivers-freeworld
```

Per **tornare indietro** (e riavere il solo driver VA-API di Fedora), basta rimuovere il pacchetto freeworld:

```bash
sudo dnf remove mesa-va-drivers-freeworld
```

### 5.4 E il Vulkan "freeworld"?

Una delle fonti che ho letto aggiunge un passaggio facoltativo: `sudo dnf swap mesa-vulkan-drivers mesa-vulkan-drivers-freeworld`, per la **decodifica video via Vulkan** (*Vulkan Video*), usata da versioni recenti di alcuni lettori. Non l'ho verificato e non rientra nell'obiettivo di questo capitolo (VA-API per browser e lettori comuni). Lo lascio fuori: se un giorno mi serve, gli dedico una nota a parte.

### 5.5 Dopo l'installazione

I programmi già in esecuzione hanno ancora il vecchio driver in memoria (stesso discorso visto per le librerie in `codec-multimediali.md`). Chiudo e riapro browser e lettori video, oppure riavvio.

---

## 6. Verifica con `vainfo`, spiegata bene

La guida originale si limitava a dire: "se compare una tabella con `VAProfileH264...` e `VAProfileHEVC...`, funziona". È un buon primo segnale, ma va letto per intero.

`vainfo` viene dal pacchetto `libva-utils`:

```bash
sudo dnf install libva-utils
vainfo
```

### 6.1 Le righe iniziali

All'inizio, prima della tabella, `vainfo` stampa informazioni sul driver caricato. Quelle da guardare:

| Riga | Cosa mi dice |
|---|---|
| `libva info: VA-API version X.Y.Z` | Versione dell'interfaccia VA-API. Di solito non è un problema. |
| `libva info: Trying to open /usr/lib64/dri…/radeonsi_drv_video.so` | **Quale file di driver sta caricando.** Con il pacchetto freeworld mi aspetto di vedere il percorso nella directory **`dri-freeworld`** (secondo la guida LinuxCapable); se vedo la directory `dri` normale, sto ancora usando il driver di Fedora. Da controllare sul mio sistema. |
| `vainfo: Driver version: Mesa Gallium driver ... for AMD ...` | Versione di Mesa e nome della GPU. Dovrebbe coincidere con quella di `rpm -q mesa-dri-drivers`. |

Se invece compare un errore come `libva error: ... failed` o `vaInitialize failed`, la catena si è interrotta in qualche punto: vedi sezione 9.

### 6.2 La tabella dei profili

Ogni riga ha la forma `VAProfile<Codec>... : VAEntrypoint<Tipo>`.

**Profilo** = il codec (e il suo "livello" di funzionalità):

| Profilo | Codec |
|---|---|
| `VAProfileH264Main`, `H264High`, `H264ConstrainedBaseline` | H.264 / AVC |
| `VAProfileHEVCMain`, `HEVCMain10` | H.265 / HEVC (8 e 10 bit) |
| `VAProfileVP9Profile0`, `VP9Profile2` | VP9 (8 e 10 bit) |
| `VAProfileAV1Profile0` | AV1 |
| `VAProfileJPEGBaseline` | JPEG |

**Entrypoint** = che operazione può fare:

| Entrypoint | Significato |
|---|---|
| `VAEntrypointVLD` | **Decodifica** (*Variable Length Decoding*): quella che mi interessa per guardare video. |
| `VAEntrypointEncSlice` | **Codifica**. |
| `VAEntrypointEncSliceLP` | Codifica in modalità a basso consumo (*Low Power*). |
| `VAEntrypointVideoProc` | Elaborazione video (ridimensionamento, conversione colore). |

Con Mesa su AMD mi aspetto principalmente `VLD`; la codifica hardware può essere presente per alcuni codec e dipende da generazione di GPU e versione di Mesa. Non do per scontato quali.

### 6.3 La verifica "prima/dopo", quella che dimostra qualcosa

L'ho già usata per RPM Fusion e codec e qui è ancora più utile, perché la differenza è proprio **tra due stati dello stesso comando**:

```bash
vainfo 2>&1 | grep -E "VAProfile" > ~/post-install-info/vainfo-prima.txt
# ... installo mesa-va-drivers-freeworld e riavvio ...
vainfo 2>&1 | grep -E "VAProfile" > ~/post-install-info/vainfo-dopo.txt
diff ~/post-install-info/vainfo-prima.txt ~/post-install-info/vainfo-dopo.txt
```

(`2>&1` serve perché `vainfo` scrive parte dell'output su *stderr*; il reindirizzamento lo porta nello stesso flusso sul quale agisce `grep`.)

Il risultato che mi aspetto, in base alla guida originale, è che le righe `H264` e `HEVC` **compaiano solo nel "dopo"**. Se compaiono già "prima", significa che il driver di Fedora le espone già (la guida originale potrebbe essere fuori data anche su questo) oppure che avevo già installato il pacchetto freeworld senza ricordarlo. Se "dopo" non mostra niente di nuovo, il pacchetto freeworld non sta venendo caricato: passo alla sezione 9.

### 6.4 I limiti di `vainfo`

Questo è il punto che la guida originale non diceva, e che vale la pena fissare:

> `vainfo` dimostra che **il driver è in grado** di decodificare un certo formato. **Non** dimostra che le applicazioni lo stiano effettivamente usando.

È come controllare che in cucina ci sia una lavastoviglie: sapere che c'è non dimostra che i piatti si stiano lavando lì invece che a mano nel lavello. Per capire se lavora davvero servono test che osservano **un'applicazione in esecuzione**, ed è la sezione successiva.

---

## 7. Test reali: sta davvero lavorando la GPU?

Cinque test, dal più controllato al più "di tutti i giorni". Per ognuno annoto cosa mi aspetto e cosa vuol dire se non va.

### 7.1 `ffmpeg` con decodifica VA-API (il più controllato)

Costruisco prima un video di prova, senza scaricare niente, con l'`ffmpeg` completo di `codec-multimediali.md`:

```bash
mkdir -p ~/post-install-info/prove
cd ~/post-install-info/prove

# 30 secondi di video sintetico 1080p, codificato in H.264
ffmpeg -hide_banner -loglevel error -y \
  -f lavfi -i testsrc2=duration=30:size=1920x1080:rate=30 \
  -c:v libx264 -pix_fmt yuv420p prova-h264.mp4

# lo stesso in H.265 (la codifica è più lenta: sono 10 secondi)
ffmpeg -hide_banner -loglevel error -y \
  -f lavfi -i testsrc2=duration=10:size=1920x1080:rate=30 \
  -c:v libx265 -pix_fmt yuv420p prova-h265.mkv
```

Poi decodifico lo stesso file in due modi e confronto il **tempo di CPU** che `ffmpeg` dichiara di aver usato (`-benchmark` stampa, in fondo, le righe `bench: utime=... rtime=...`):

```bash
# decodifica in software (CPU)
ffmpeg -benchmark -hide_banner -i prova-h264.mp4 -f null -

# decodifica con VA-API (GPU)
ffmpeg -benchmark -hide_banner \
  -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 -hwaccel_output_format vaapi \
  -i prova-h264.mp4 -f null -
```

Cosa significano i flag:

- `-hwaccel vaapi` chiede a `ffmpeg` di usare VA-API per decodificare.
- `-hwaccel_device /dev/dri/renderD128` indica **quale GPU** (il render node visto in 4.3). Con una sola GPU è quasi certamente `renderD128`; lo verifico con `ls /dev/dri`.
- `-hwaccel_output_format vaapi` tiene i fotogrammi decodificati **nella memoria della GPU** invece di riportarli alla CPU. Lo aggiungo anche per un motivo pratico: senza questa opzione, se la decodifica hardware fallisce `ffmpeg` può ripiegare in silenzio sul software e il test sembra riuscito. Con l'opzione, un problema produce un **errore esplicito**. (È una mia deduzione dall'uso comune di questa opzione, non l'ho trovata scritta come garanzia: se il comando termina senza errori e `utime` crolla, ho comunque una conferma doppia.)
- `-f null -` decodifica tutto e butta il risultato: misuro solo la decodifica.

**Come leggo il risultato.** Confronto `utime` (secondi di CPU "utente" consumati) tra le due esecuzioni:

| Osservazione | Interpretazione |
|---|---|
| Comando VA-API termina senza errori, `utime` **molto più basso** | La GPU sta decodificando. Esito atteso. |
| Errore (`Failed setup for format vaapi`, `No usable decoder`, `hwaccel initialisation returned error`) | La catena non funziona per quel codec: guardo `vainfo` e la sezione 9. |
| Funziona, ma `utime` quasi uguale al software | Sospetto: il file è troppo "leggero" (un video sintetico a basso carico può dare differenze piccole) oppure la decodifica è finita in CPU. Riprovo con un video reale (7.2). |

Stesso test per H.265, con `prova-h265.mkv`.

### 7.2 Un video reale con `mpv`

`mpv` è un lettore che dice chiaramente se sta usando l'accelerazione hardware:

```bash
sudo dnf install mpv
mpv --hwdec=vaapi prova-h264.mp4
```

- `--hwdec=vaapi` chiede esplicitamente la decodifica VA-API.
- Nel terminale, tra i messaggi iniziali, cerco una riga come `Using hardware decoding (vaapi)`. Se compare un messaggio di fallimento e poi una riga che parla di decodifica software, sta ripiegando sulla CPU.
- Durante la riproduzione, `Shift+I` mostra le statistiche a schermo: tra queste, la voce relativa al *hardware decoding* dice quale metodo è attivo. Va letta a video, perché i nomi delle voci cambiano tra versioni di `mpv`.

Per un test più realistico, serve un video non sintetico (animazione o ripresa vera). Per esempio i file di prova distribuiti pubblicamente con il film *Big Buck Bunny* in vari formati, o i file di test pubblici in diversi codec e bitrate (ricordo siti come `test-videos.co.uk` e `jell.yfish.us`; **non ho verificato che siano raggiungibili oggi**, quindi vanno cercati o sostituiti con un qualsiasi video proprio). Se ho un file mio in H.265, è il modo migliore di provare HEVC, perché i browser non sono un buon banco di prova per quel codec.

### 7.3 Firefox

Il browser è il caso d'uso reale, ed è anche un bersaglio che si muove: dal 2025 Firefox abilita la decodifica hardware su GPU AMD in Linux di default (era la novità di Firefox 136), quindi sul mio sistema non dovrei dover toccare `about:config`. Verifico comunque.

**Controllo 1: `about:support`.** Nella pagina `about:support`, sezione *Graphics*, cerco nel registro delle decisioni (*Decision Log*) la riga **`HARDWARE_VIDEO_DECODING`**. Secondo la pagina Fedora dedicata a Firefox, la dicitura `available by default` significa che il browser sta decodificando in hardware per impostazione predefinita. Nelle versioni recenti esiste anche una sezione *Media* con le informazioni sui codec supportati in software e hardware. Dipende dalla versione: se non la trovo, non è un errore.

**Controllo 2: il log di Firefox.** Il più concreto, perché registra cosa succede *durante* una riproduzione. Chiudo del tutto Firefox, poi lo avvio dal terminale con il logging attivato:

```bash
MOZ_LOG="FFmpegVideo:5" firefox 2>&1 | grep -iE "vaapi|va-api|hardware"
```

- `MOZ_LOG="FFmpegVideo:5"` è una **variabile d'ambiente** impostata solo per quel comando (si scrive davanti al comando, senza `export`) che attiva il livello di log 5 per il modulo di decodifica video di Firefox.
- `2>&1` porta i messaggi di log, che Firefox scrive su stderr, nel flusso filtrato da `grep`.

Poi apro un video (anche un file H.264 in locale trascinato nella finestra) e guardo se compaiono righe che parlano di VA-API. Non ho verificato quali stringhe esatte compaiano sul mio sistema: la prima volta le leggo senza filtro e annoto qui sotto cosa ho visto.

*(Spazio per le mie annotazioni dopo la prova.)*

⚠️ **Se Firefox è un Flatpak** (installato da Discover/Flathub) **e non il pacchetto RPM di Fedora**, tutto questo capitolo ha effetto diverso: le applicazioni Flatpak non usano il Mesa e i codec del sistema, ma quelli dei **runtime** Flatpak. Non ho verificato come si comporti la decodifica hardware in quel caso, quindi prima controllo quale Firefox ho (`rpm -q firefox`; se risponde "not installed" è molto probabile che sia Flatpak) e tratto il caso a parte.

### 7.4 YouTube e le "statistiche per nerd"

Un test online semplice, ma da interpretare con attenzione. Su YouTube, clic destro sul video → **Statistiche per nerd**. Mi interessano due righe:

- **Codecs**: indica il codec **che YouTube sta servendo** in quel momento, per esempio `avc1` (H.264), `vp09` (VP9), `av01` (AV1).
- **Dropped frames**: i fotogrammi persi. Un numero che cresce rapidamente indica un sistema che non riesce a tenere il passo.

Questa schermata **non dice se la decodifica è hardware o software**. Serve invece a incrociare un'informazione con `vainfo`: se YouTube mi serve `av01` e `vainfo` non elenca AV1 tra i profili con `VAEntrypointVLD`, quel video lo sta decodificando la CPU, anche se tutto il resto è configurato bene. Allo stesso modo `vp09` richiede un profilo VP9 in `vainfo`. YouTube sceglie il codec in base a cosa il browser dichiara di saper decodificare, quindi può cambiare da un video all'altro e da un browser all'altro.

### 7.5 Osservare il carico e il consumo mentre il video gira

Le prove precedenti dicono "la decodifica è hardware". Questa dice "e infatti la CPU non fa fatica e il portatile consuma meno": è la prova che interessa davvero a chi usa il portatile in batteria.

**Carico della CPU.** Mentre il video gira in `mpv` (con e senza `--hwdec=vaapi`, o con `--hwdec=no` per la versione software):

```bash
top
# oppure, se installato: htop
```

Confronto la percentuale di CPU del processo del lettore nei due casi. Su un 1080p la differenza dovrebbe essere netta. Anche il *Monitor di sistema* di KDE va bene.

**Carico della GPU: attenzione al falso negativo.** Strumenti come `radeontop` mostrano soprattutto il carico della parte 3D/grafica della GPU, **non** quello del blocco video dedicato. Un `radeontop` quasi fermo durante la riproduzione non smentisce l'accelerazione, e non la dimostra nemmeno. Per vedere il blocco video esistono strumenti come **`nvtop`** (nonostante il nome supporta anche AMD) e **`amdgpu_top`**: a quanto leggo mostrano l'attività di decodifica/codifica. Non li ho ancora provati su questo sistema né verificato se sono nei repository che ho abilitati; li annoto come prova da fare.

**Consumo.** Il valore più significativo per un portatile. Funziona con il portatile **a batteria** (non in carica):

```bash
upower -e                                   # elenca i dispositivi; cerco quello della batteria
upower -i /org/freedesktop/UPower/devices/battery_BAT0 | grep -E "energy-rate|state"
```

(il percorso `battery_BAT0` va sostituito con quello che compare in `upower -e`.) `energy-rate` è la potenza istantanea in Watt. Misuro, a luminosità e profilo energetico identici, il consumo durante la riproduzione in hardware e in software, lasciando passare qualche decina di secondi prima di leggere il valore. Meglio fare più letture e ragionare sull'ordine di grandezza.

Questo test ha un motivo specifico, non solo pratico. La guida originale affermava che lo swap lascia un consumo basso. Non ho trovato conferme su Fedora 44, e anzi un utente (Fedora 38, un portatile AMD diverso dal mio, quindi un dato da prendere con le pinze) aveva misurato **più** consumo con i driver freeworld che con quelli standard, senza riuscire a spiegarlo. Quindi: la mia affermazione sul consumo è un'ipotesi finché non la misuro io.

### 7.6 Riepilogo: come interpreto gli esiti

| Situazione | Probabile significato | Dove guardare |
|---|---|---|
| `vainfo` senza H.264/HEVC dopo l'installazione | Il driver freeworld non è caricato | Sezione 9 |
| `vainfo` OK, `ffmpeg` VA-API con errore | Problema di `ffmpeg`/render node/versioni | Sezione 9 |
| `ffmpeg` OK, `mpv` ripiega sul software | Problema nel lettore o nella sua configurazione | Messaggi di `mpv` nel terminale |
| `mpv` OK, Firefox no | Problema nel browser (Flatpak? impostazioni? profilo?) | `about:support`, log di Firefox |
| Tutto "OK" ma un certo video di YouTube pesa sulla CPU | Codec servito non supportato dalla GPU (tipicamente AV1 o VP9) | Statistiche per nerd + `vainfo` |

---

## 8. Checkpoint automatico (prima/dopo)

Come per RPM Fusion e codec, raccolgo tutto in un file con data e `ETICHETTA`, da lanciare **prima** dell'installazione di `mesa-va-drivers-freeworld` e **dopo**.

Note sulla sintassi, per quanto non sia la prima volta che compaiono:

- `RENDER=/dev/dri/renderD128` è una variabile con il render node, in testa allo script per cambiarlo in un solo posto se serve.
- `test_hw()` è una funzione bash (come `test_codec` in `codec-multimediali.md`): genera un file di prova e prova a decodificarlo con VA-API, annotando `[OK]` o `[FAIL]` in base al **codice di uscita** di `ffmpeg`.
- `|| echo "..."` stampa una riga leggibile quando il comando a sinistra fallisce (cioè `grep` non trova niente).

```bash
ETICHETTA=prima      # cambia in "dopo" quando rilanci a installazione fatta
RENDER=/dev/dri/renderD128   # verifica con: ls /dev/dri

mkdir -p ~/post-install-info
CHECK=~/post-install-info/grafica-checkpoint-$ETICHETTA-$(date +%Y-%m-%d_%H-%M).txt
NEXT=$(( $(rpm -E %fedora) + 1 ))

test_hw() {   # $1 = encoder usato per generare il file, $2 = estensione
  local src="/tmp/hwtest-$1.$2"

  if ! command -v ffmpeg >/dev/null; then
    echo "[--] ffmpeg non trovato"
    return
  fi

  if ! ffmpeg -hide_banner -loglevel error -y \
       -f lavfi -i testsrc2=duration=10:size=1920x1080:rate=30 \
       -c:v "$1" -pix_fmt yuv420p "$src"; then
    echo "[--] impossibile generare il file di prova con $1"
    return
  fi

  if ffmpeg -hide_banner -loglevel error \
       -hwaccel vaapi -hwaccel_device "$RENDER" -hwaccel_output_format vaapi \
       -i "$src" -f null - ; then
    echo "[OK]   decodifica VA-API del file generato con $1"
  else
    echo "[FAIL] decodifica VA-API del file generato con $1"
  fi

  rm -f "$src"
}

{
  echo "=== Checkpoint grafica ($ETICHETTA) — $(date '+%A %d %B %Y, %H:%M:%S') ==="
  echo

  echo "--- Fedora e kernel ---"
  cat /etc/fedora-release
  echo "Kernel: $(uname -r)"
  echo

  echo "--- GPU e driver del kernel in uso ---"
  lspci -nnk 2>/dev/null | grep -A3 -E "VGA|3D|Display"
  echo

  echo "--- Nodi DRM ---"
  ls -l /dev/dri/
  echo

  echo "--- Pacchetti Mesa, libva, firmware (con vendor) ---"
  rpm -qa 'mesa*' 'libva*' 'amd-gpu-firmware*' \
    --qf '%{NAME}-%{VERSION}-%{RELEASE}  [%{VENDOR}]\n' | sort
  echo

  echo "--- Pacchetti freeworld installati ---"
  rpm -qa '*freeworld*' || echo "nessuno"
  echo

  echo "--- Pacchetti installati costruiti per fc$NEXT (atteso: nessuno) ---"
  rpm -qa | grep -F ".fc$NEXT" || echo "nessuno"
  echo

  echo "--- OpenGL (glxinfo) ---"
  if command -v glxinfo >/dev/null; then
    glxinfo -B 2>&1 | grep -E "renderer string|OpenGL version string"
  else
    echo "[--] glxinfo non installato"
  fi
  echo

  echo "--- vainfo: driver caricato ---"
  if command -v vainfo >/dev/null; then
    vainfo 2>&1 | grep -iE "libva info|Trying to open|Driver version|error|failed"
  else
    echo "[--] vainfo non installato (pacchetto libva-utils)"
  fi
  echo

  echo "--- vainfo: profili ed entrypoint ---"
  if command -v vainfo >/dev/null; then
    vainfo 2>&1 | grep -E "VAProfile" || echo "nessun profilo elencato"
  fi
  echo

  echo "--- Test funzionale ffmpeg + VA-API (H.264) ---"
  test_hw libx264 mp4
  echo

  echo "--- Test funzionale ffmpeg + VA-API (H.265) ---"
  test_hw libx265 mkv
  echo

} > "$CHECK" 2>&1

echo "Checkpoint salvato in: $CHECK"
```

### Come leggere il risultato

| Sezione del file | Prima (atteso) | Dopo (atteso) |
|---|---|---|
| Driver in uso | `amdgpu` | `amdgpu` (non deve cambiare) |
| Pacchetti freeworld | `nessuno` | `mesa-va-drivers-freeworld`, vendor **RPM Fusion** |
| Pacchetti `fc$NEXT` | `nessuno` | `nessuno` |
| `glxinfo` | renderer AMD/Mesa, non `llvmpipe` | invariato |
| `vainfo` (percorso del driver) | directory `dri` di Fedora | probabilmente `dri-freeworld` (da confermare) |
| Profili `H264`/`HEVC` | assenti (secondo la guida originale) | presenti con `VAEntrypointVLD` |
| Test H.264/H.265 | `[FAIL]` | `[OK]` |

Per confrontare i due stati:

```bash
diff ~/post-install-info/grafica-checkpoint-prima-*.txt \
     ~/post-install-info/grafica-checkpoint-dopo-*.txt
```

I due file hanno timestamp diversi nel titolo; è normale che compaia come differenza. Se ho più di un file `prima` o `dopo` (per esempio due tentativi), il carattere `*` si espande in più nomi e `diff` si lamenta: in quel caso indico io il file esatto.

---

## 9. Quando qualcosa non va

**`vainfo` dà errore all'avvio** (`libva error: ... failed`, `vaInitialize failed`).
Controllo, in quest'ordine: (1) che esista un render node (`ls /dev/dri`); (2) che `Kernel driver in use` sia `amdgpu` (sezione 4); (3) che `rpm -q mesa-va-drivers-freeworld` risponda con un pacchetto e non con `not installed`. Se sono su un'altra GPU o ho più GPU, posso indicare il dispositivo esplicitamente con `vainfo --display drm --device /dev/dri/renderD128`.

**`vainfo` funziona ma non vedo H.264/HEVC dopo l'installazione.**
Vuol dire che il driver freeworld non è quello caricato. Rivedo il percorso nella riga `Trying to open` (sezione 6.1) e controllo che le versioni coincidano:

```bash
rpm -q mesa-dri-drivers mesa-filesystem mesa-va-drivers-freeworld
```

Se la versione di Mesa dei pacchetti Fedora e quella del pacchetto freeworld sono diverse, torno alla nota sulle versioni nella sezione 5.3.

**Un aggiornamento di Mesa si blocca o chiede di toccare `mesa-va-drivers`.**
È il disallineamento tra Fedora e RPM Fusion descritto nella sezione 5.3. Non forzo con `--allowerasing` senza aver letto il riepilogo.

**`ffmpeg` con VA-API funziona ma il browser no.**
Il browser potrebbe non usare `ffmpeg` di sistema (Flatpak, sezione 7.3) o avere la decodifica hardware disattivata nel profilo (`about:config`, chiave `media.hardware-video-decoding.enabled`, citata nella pagina Fedora su Firefox). Non cambio nulla prima di aver letto `about:support`.

**Forzare il driver con una variabile d'ambiente.**
Su `vainfo`/applicazioni, la variabile `LIBVA_DRIVER_NAME` (per esempio `LIBVA_DRIVER_NAME=radeonsi`) forza la scelta del driver VA-API. Una guida la cita per alcune combinazioni con più GPU. La uso solo come **test temporaneo** (`LIBVA_DRIVER_NAME=radeonsi vainfo`), non la metto in modo permanente nel profilo senza capire perché serve: se serve, c'è un problema a monte.

---

## 10. Riepilogo operativo

- [ ] Verificare i repository RPM Fusion (`rpm-fusion.md`) e che `ffmpeg` completo sia installato (`codec-multimediali.md`)
- [ ] Identificare la GPU: `lspci -nnk | grep -A3 -E "VGA|3D|Display"` e controllare `Kernel driver in use: amdgpu`
- [ ] Controllare il livello grafico: `glxinfo -B` (niente `llvmpipe`), `ls /dev/dri`
- [ ] Installare `libva-utils` e salvare lo stato **prima** (`vainfo-prima.txt` e checkpoint con `ETICHETTA=prima`)
- [ ] Simulare: `sudo dnf install mesa-va-drivers-freeworld --assumeno` e leggere il riepilogo (nessuna rimozione di pacchetti Mesa)
- [ ] Installare `mesa-va-drivers-freeworld` (niente `swap`, niente VDPAU)
- [ ] Riavviare le applicazioni o il sistema
- [ ] Rilanciare `vainfo` e il checkpoint con `ETICHETTA=dopo`, poi `diff`
- [ ] Test `ffmpeg` (software vs VA-API) con `-benchmark`
- [ ] Test `mpv --hwdec=vaapi` su un video reale
- [ ] Controllare Firefox (`about:support`, riga `HARDWARE_VIDEO_DECODING`; capire se è RPM o Flatpak)
- [ ] YouTube: incrociare "Statistiche per nerd" (codec servito) con i profili di `vainfo`
- [ ] Misurare CPU e consumo con e senza decodifica hardware, a batteria
- [ ] Annotare i risultati effettivi (percorso del driver in `vainfo`, righe del log di Firefox, consumi) in questo capitolo
