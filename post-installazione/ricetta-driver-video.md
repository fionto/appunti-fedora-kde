# Impostare la scheda video

Questa sezione è autonoma: raccoglie i comandi che uso, nell'ordine in cui li eseguo, per tre casi. Il primo è quello del mio Zenbook (GPU AMD integrata). Gli altri due sono esempi NVIDIA **legacy**: una **Quadro K2000** (architettura Kepler) e una **GeForce GTX 1050** (architettura Pascal). Li tengo qui per confronto, perché mostrano bene quanto cambia la procedura a seconda del produttore e della generazione della scheda.

## Quale ricetta mi riguarda

Il primo passo è sempre lo stesso: leggere il **vendor ID** della GPU (sezione 4).

```bash
lspci -nnk 2>/dev/null | grep -A3 -E "VGA|3D|Display"
```

| Vendor ID | Produttore | Ricetta |
|---|---|---|
| `1002` | AMD | 11.1 |
| `10de` | NVIDIA | 11.2: prima identifico la generazione, poi scelgo il ramo del driver |
| `8086` | Intel | Non trattata qui |

Per NVIDIA, la generazione decide *quale pacchetto* installare:

| Architettura | Esempi | Ramo del driver (RPM Fusion) | Pacchetto `akmod` |
|---|---|---|---|
| Kepler | Quadro K2000, GeForce serie 600 e molte 700 | **470xx** (supporto terminato lato NVIDIA, pacchetto mantenuto da RPM Fusion "al meglio delle possibilità") | `akmod-nvidia-470xx` |
| Maxwell, Pascal, Volta | GeForce GTX 900 e serie 10 (compresa la **GTX 1050**) | **580xx** (ramo legacy mantenuto) | `akmod-nvidia-580xx` |
| Turing e successive | Serie RTX | ramo corrente | `akmod-nvidia` |

Due note su questa tabella. Le Quadro il cui nome inizia per **K** sono Kepler. E il pacchetto senza suffisso, `akmod-nvidia`, è il ramo **corrente**: su Fedora 44 non supporta più Maxwell/Pascal/Volta, quindi con una GTX 1050 **non va installato**.

⚠️ Prima di scegliere guardo cosa offrono davvero i repository abilitati:

```bash
dnf list --available 'akmod-nvidia*'
```

I rami legacy si trovano nei repository `rpmfusion-nonfree` e `rpmfusion-nonfree-updates` (non in `rpmfusion-nonfree-nvidia-driver`, secondo quanto discusso nei forum Fedora): devono essere **abilitati** entrambi, come nel controllo dei quattro repository di `rpm-fusion.md`.

---

## AMD: GPU integrata (il mio caso)

**Cosa serve davvero:** quasi niente. Il driver del kernel (`amdgpu`), il firmware e Mesa sono già nel sistema. L'unica aggiunta è il driver VA-API con i codec abilitati, perché quello di Fedora li ha disattivati (sezione 3.3).

```bash
# 1. Sistema aggiornato (se ci sono kernel nuovi, riavvio prima di proseguire)
sudo dnf upgrade --refresh

# 2. Strumenti di verifica
sudo dnf install pciutils libva-utils vulkan-tools

# 3. Il driver giusto è già in uso?   Atteso: "Kernel driver in use: amdgpu"
lspci -nnk 2>/dev/null | grep -A3 -E "VGA|3D|Display"

# 4. Firmware e Mesa presenti?
rpm -q amd-gpu-firmware mesa-dri-drivers mesa-vulkan-drivers

# 5. Stato "prima" del video accelerato
vainfo 2>&1 | grep -E "VAProfile" > ~/post-install-info/vainfo-prima.txt

# 6. Simulazione: nessuna rimozione di pacchetti Mesa
sudo dnf install mesa-va-drivers-freeworld --assumeno

# 7. Installazione (solo se il riepilogo è pulito)
sudo dnf install mesa-va-drivers-freeworld

# 8. Riavvio, poi stato "dopo" e confronto
vainfo 2>&1 | grep -E "VAProfile" > ~/post-install-info/vainfo-dopo.txt
diff ~/post-install-info/vainfo-prima.txt ~/post-install-info/vainfo-dopo.txt
```

Il comando `glxinfo` non compare nell'elenco perché il nome del pacchetto che lo fornisce varia tra versioni di Fedora: `dnf provides '*/glxinfo'` lo trova. Per tornare allo stato precedente: `sudo dnf remove mesa-va-drivers-freeworld`.

---

## NVIDIA legacy: esempi per confronto

Due differenze strutturali rispetto ad AMD, che spiegano quasi tutto il resto:

1. **Il driver non è nel kernel.** Il modulo NVIDIA è **proprietario e fuori dal kernel**: RPM Fusion lo distribuisce come sorgente e un sistema chiamato **akmods** lo **compila sul mio computer** per il kernel in uso, e lo ricompila a ogni aggiornamento del kernel. Per questo, nei pacchetti, compare `akmod-` (il "compilatore automatico").
2. **Il nome del pacchetto contiene il ramo.** `-580xx` e `-470xx` sono rami di versione distinti, ciascuno con il proprio set di pacchetti. Un normale `dnf upgrade` resta dentro il ramo installato.

### Passo A: Secure Boot

Un modulo compilato localmente è **non firmato**. Se il Secure Boot è attivo, il kernel lo rifiuta finché non firmo il modulo con una mia chiave e non la registro nel firmware (*MOK*, Machine Owner Key).

```bash
mokutil --sb-state
```

Se risponde `SecureBoot disabled`, passo al punto B. Se risponde `enabled`, preparo la chiave **prima** di installare il driver:

```bash
sudo dnf install kmodtool akmods mokutil openssl
sudo kmodgenca -a
sudo mokutil --import /etc/pki/akmods/certs/public_key.der
```

- `kmodgenca -a` genera la chiave con cui `akmods` firmerà i moduli;
- `mokutil --import` mette in coda la sua registrazione e chiede di scegliere una **password temporanea**;
- al **prossimo riavvio** compare una schermata blu (*MOK Manager*): scelgo *Enroll MOK*, confermo, inserisco la password scelta. Ricordare la password serve solo per quel passaggio.

Questi tre comandi sono quelli della procedura Secure Boot di RPM Fusion per come li ricordo; non ho potuto rileggere oggi la pagina ufficiale (vedi limite nella sezione 1), quindi li confronto con la documentazione prima di usarli.

### Passo B: installazione

**Esempio 1: GeForce GTX 1050 (Pascal) → ramo 580xx**

```bash
sudo dnf install akmod-nvidia-580xx xorg-x11-drv-nvidia-580xx xorg-x11-drv-nvidia-580xx-cuda
```

**Esempio 2: Quadro K2000 (Kepler) → ramo 470xx**

```bash
sudo dnf install akmod-nvidia-470xx xorg-x11-drv-nvidia-470xx xorg-x11-drv-nvidia-470xx-cuda
```

I tre pacchetti, in ordine: il compilatore automatico del modulo, i componenti in user space (le librerie OpenGL/Vulkan di NVIDIA) e, facoltativo, il supporto CUDA, che nelle guide è indicato come necessario per avere il comando `nvidia-smi`. Se non mi servono né CUDA né `nvidia-smi`, posso ometterlo.

Se DNF risponde `No match for argument`, controllo prima di tutto che `rpmfusion-nonfree-updates` sia abilitato e riprovo con `--refresh`.

### Passo C: aspettare la compilazione (non riavviare subito)

Dopo l'installazione, `akmods` compila il modulo in background; può richiedere alcuni minuti. Se riavvio prima che finisca, rischio di ritrovarmi con il driver libero (nouveau). Controllo:

```bash
modinfo -F version nvidia
```

Se stampa un numero di versione, il modulo è pronto. Se risponde che il modulo non esiste, aspetto e ripeto. Se dopo parecchio tempo non c'è, guardo i log di compilazione nella cache di `akmods` (per questi rami la cartella è `/var/cache/akmods/nvidia-580xx/` oppure `/var/cache/akmods/nvidia-470xx/`, a seconda del ramo):

```bash
ls /var/cache/akmods/nvidia-580xx/    # oppure nvidia-470xx
```

I file con `failed` nel nome contengono l'errore: per il ramo 470xx, che non è più sviluppato da NVIDIA, un kernel molto recente può rompere la compilazione ed è il caso peggiore da mettere in conto.

### Passo D: Kepler e sessione X11

Questo passo vale **solo** per il ramo 470xx. A quanto leggo nelle guide, quel driver non supporta Wayland, quindi la Quadro K2000 richiede una sessione **X11**. Su Fedora KDE la sessione X11 non è installata di default:

```bash
sudo dnf install plasma-workspace-x11
```

Dopo il riavvio scelgo **Plasma (X11)** dal selettore di sessione nella schermata di login (con Plasma Login Manager non ho verificato dove si trovi il selettore). Il pacchetto, secondo la sua descrizione, è mantenuto da singoli packager e **non** dal KDE SIG: è un fatto da tenere presente, perché una scheda Kepler dipende da una sessione non supportata ufficialmente. Come si comporterà con le versioni future di Plasma non lo so: lo ricontrollo a ogni passaggio di versione di Fedora.

La GTX 1050 con il ramo 580xx non ha questo vincolo (in linea di principio funziona su Wayland); lo verifico con il controllo di sessione qui sotto.

### Passo E: riavvio e verifica

```bash
# 1. il kernel usa il driver NVIDIA e non nouveau?  Atteso: "Kernel driver in use: nvidia"
lspci -nnk 2>/dev/null | grep -A3 -i nvidia

# 2. moduli caricati
lsmod | grep nvidia

# 3. versione del driver in uso (utile per confrontarla con il ramo scelto)
cat /proc/driver/nvidia/version

# 4. stato della GPU (solo se ho installato il pacchetto -cuda)
nvidia-smi

# 5. tipo di sessione attiva: x11 oppure wayland
echo $XDG_SESSION_TYPE
```

Atteso: `Kernel driver in use: nvidia`, un numero di versione coerente con il ramo (580.x oppure 470.x) e, per il Kepler, sessione `x11`. Se `Kernel driver in use` mostra ancora `nouveau`, il modulo NVIDIA non si è caricato: torno al passo C (compilazione) e, con Secure Boot attivo, al passo A (chiave non registrata).

### Tornare indietro

Rimuovo solo i pacchetti del ramo installato, dopo una simulazione. Evito i caratteri jolly generici come `*nvidia*`, che potrebbero includere pacchetti Fedora legati al driver libero:

```bash
sudo dnf remove akmod-nvidia-580xx 'kmod-nvidia*' 'xorg-x11-drv-nvidia-580xx*' --assumeno
```

(per il ramo 470xx cambio `580xx` in `470xx`). Se il riepilogo è quello atteso, ripeto senza `--assumeno` e riavvio.

---

## Le differenze, in una tabella

| | AMD (integrata, il mio Zenbook) | NVIDIA Pascal (GTX 1050) | NVIDIA Kepler (Quadro K2000) |
|---|---|---|---|
| Driver del kernel | `amdgpu`, **dentro il kernel** | Modulo proprietario, **compilato da `akmods`** | Come Pascal, ma ramo non più sviluppato da NVIDIA |
| Cosa installo | Un solo pacchetto (`mesa-va-drivers-freeworld`), solo per i codec video | `akmod-nvidia-580xx` + librerie | `akmod-nvidia-470xx` + librerie |
| Repository necessari | RPM Fusion Free | RPM Fusion Nonfree (+ updates) | RPM Fusion Nonfree (+ updates) |
| Secure Boot | Nessun problema (modulo firmato dal kernel) | Serve registrare una chiave (MOK) | Serve registrare una chiave (MOK) |
| Attesa dopo l'installazione | Nessuna | Compilazione del modulo | Compilazione del modulo |
| Dopo un aggiornamento del kernel | Niente da fare | Ricompilazione automatica | Ricompilazione automatica, con rischio di rottura su kernel recenti |
| Sessione grafica | Wayland | Wayland (da verificare) | **X11** (`plasma-workspace-x11`) |
| Pacchetto sbagliato da evitare | `mesa-va-drivers` come pacchetto separato (non esiste più) | `akmod-nvidia` senza suffisso (ramo corrente, non supporta Pascal) | `akmod-nvidia` senza suffisso |
| Video accelerato | VA-API nativo via Mesa (sezioni 5-7) | Via NVDEC, con un livello di traduzione per VA-API (`libva-nvidia-driver`): non verificato | Non verificato, probabilmente non utilizzabile con un driver così vecchio |

In sintesi: con AMD il driver "c'è già" e si tratta solo di sbloccare i codec; con NVIDIA legacy devo **scegliere il ramo giusto**, **farmi compilare il modulo** e occuparmi di Secure Boot e sessione grafica. Il ramo scelto è una decisione legata alla generazione della scheda: se l'hardware cambia, la ricetta cambia.

⚠️ **Limiti di questa sezione.** Non ho hardware NVIDIA per provarla e non sono riuscito a leggere le pagine ufficiali di RPM Fusion (vedi sezione 1). Mi appoggio a guide e discussioni su Fedora 44 consultate il 18 ottobre 2026, alcune di mesi fa: nomi dei pacchetti, rami supportati e disponibilità nei repository sono il tipo di informazione che cambia in fretta. Il caso Kepler/470xx poggia su meno fonti ed è quello più fragile. In ogni caso, `dnf list --available 'akmod-nvidia*'` e una simulazione con `--assumeno` valgono più di questa sezione.