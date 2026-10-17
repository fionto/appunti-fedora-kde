# Ricetta: RPM Fusion e codec multimediali su Fedora 44

Sequenza unica di comandi, ciascuno con una spiegazione breve e con **cosa aspettarsi** dall'output. Riunisce i capitoli su RPM Fusion e sui codec, già corretti.

> ⚠️ **Stato di verifica.** I comandi non sono stati eseguiti da me su questo sistema. Gli "attesi" sono ricavati dalle fonti consultate (thread di Fedora Discussion di aprile–maggio 2026 e febbraio 2026, repository RPM Fusion) e dalla logica dei comandi. Se un output differisce, vale quello che vedo sul sistema, non questa pagina. La pagina ufficiale [rpmfusion.org/Configuration](https://rpmfusion.org/Configuration) va riletta dal browser prima di partire.

**Convenzioni:** 🟢 passo normale · 🛑 punto di arresto: non proseguo se l'atteso non è confermato · 🔀 passo condizionale.

---

## Fase 0 — Preparazione e archivio delle fonti

### 0.1 🟢 Cartella di archivio e file delle fonti

```bash
mkdir -p ~/post-install-info

cat > ~/post-install-info/fonti-rpmfusion-$(date +%Y-%m-%d).txt <<EOF
Fonti consultate — $(date '+%d/%m/%Y')
  https://rpmfusion.org/Configuration
  https://rpmfusion.org/Howto/Multimedia
  https://github.com/rpmfusion/rpmfusion-free-release
  https://github.com/rpmfusion/rpmfusion-nonfree-release
  https://discussion.fedoraproject.org/t/fedora-44-and-rpmfusion/189911
  https://discussion.fedoraproject.org/t/how-do-i-enable-h265-hevc-playback-in-fedora-43-or-44-2026/147870
  https://fedoraproject.org/wiki/Hardware_Video_Acceleration
EOF
```

Il file registra da dove vengono i comandi e quando li ho consultati. `<<EOF` **senza** virgolette fa sì che bash espanda i `$(date ...)` nel testo.

**Atteso:** nessun output. `ls ~/post-install-info/` mostra il nuovo file.

Dal browser, salvo in PDF (`Ctrl+P` → *Salva come PDF*) le pagine `rpmfusion.org/Configuration` e `rpmfusion.org/Howto/Multimedia` dentro la stessa cartella. Sono protette da un controllo anti-bot: il browser lo supera, `curl`/`wget` probabilmente no.

### 0.2 🟢 Fotografia dei repository prima di cambiare

```bash
dnf repo list --all > ~/post-install-info/repo-prima-rpmfusion-$(date +%Y-%m-%d).txt
```

Serve per un confronto con `diff` alla fine.

**Atteso:** nessun output. Il file contiene l'elenco dei repo di Fedora (ID e nome), senza voci `rpmfusion*`.

---

## Fase 1 — Repository RPM Fusion

### 1.1 🟢 Installare i pacchetti release (Free e Nonfree)

```bash
sudo dnf install \
  https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
  https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```

`dnf install` accetta anche URL di file `.rpm`. `$(rpm -E %fedora)` diventa `44` (e `45` in futuro). Installo solo i due pacchetti *release* (file `.repo` e chiavi GPG), non RPM Fusion in sé. In questo momento mi fido di HTTPS e del dominio ufficiale, perché le chiavi non sono ancora nel sistema.

**Atteso:** DNF mostra due pacchetti `rpmfusion-free-release` e `rpmfusion-nonfree-release`, chiede conferma e importa le chiavi GPG. Posso trovare un messaggio "già installato" se c'erano già: in quel caso vedi il passo 1.3.

### 1.2 🛑 Diagnosi: i repository sono quelli giusti?

```bash
rpm -qa 'rpmfusion*release*' | sort
dnf repo list --all | grep -i rpmfusion
```

Le virgolette attorno a `'rpmfusion*release*'` impediscono alla shell di espandere gli asterischi.

| Cosa guardo | ✅ Atteso | ❌ Segnale d'allarme |
|---|---|---|
| Versione dei pacchetti release | `44-N` **senza** `0.` (nel thread: `44-3` o successiva) | `44-0.2`, `44-0.3` (versioni di branch, pre-rilascio) |
| Repo abilitati | `rpmfusion-free`, `rpmfusion-free-updates`, `rpmfusion-nonfree`, `rpmfusion-nonfree-updates` | Abilitati solo i `*-rawhide`, oppure i quattro risultano disabilitati |
| Repo `*-rawhide` | Assenti o disabilitati | Abilitati |

**Se tutto è ✅, salto direttamente alla Fase 2.** Se no, 1.3.

*Perché conta:* DNF, tra due repo che offrono lo stesso pacchetto, sceglie la versione più recente. Un repo `rawhide` abilitato vince su quello giusto e porta pacchetti per Fedora 45, con errori come `nothing provides libm.so.6(GLIBC_2.44)`.

### 1.3 🔀 Correzione (solo se la diagnosi è ❌)

**Caso A — versione `44-0.x`:** aggiorno i release da URL e riallineo i pacchetti.

```bash
sudo dnf update \
  https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
  https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

sudo dnf clean metadata
sudo dnf distro-sync
```

`update` confronta le versioni (`install` direbbe solo "già installato"). `clean metadata` scarta il catalogo in cache. `distro-sync` porta i pacchetti alle versioni dei repo abilitati, **anche con downgrade**: leggo il riepilogo prima di confermare (se `defaultyes=True`, Invio equivale a "sì").

**Atteso:** i release salgono a `44-N`; il riepilogo di `distro-sync` è vuoto o contiene solo pacchetti che riconosco. Se la versione resta `44-0.x`, l'URL ha servito di nuovo il file vecchio (ipotesi del thread: mirror non aggiornato). Alternativa storica da provare: stessi percorsi con `download1.rpmfusion.org` al posto di `mirrors.rpmfusion.org` (da verificare che risponda).

**Caso B — versione giusta, ma repo sbagliati:**

```bash
sudo dnf config-manager setopt rpmfusion-free-rawhide.enabled=0
sudo dnf config-manager setopt rpmfusion-nonfree-rawhide.enabled=0
sudo dnf config-manager setopt rpmfusion-free.enabled=1
sudo dnf config-manager setopt rpmfusion-free-updates.enabled=1
sudo dnf config-manager setopt rpmfusion-nonfree.enabled=1
sudo dnf config-manager setopt rpmfusion-nonfree-updates.enabled=1

ls /etc/dnf/repos.override.d/ 2>/dev/null
```

`setopt <id>.<opzione>=<valore>` imposta l'opzione di un repo. Per quanto ricordo dalla documentazione DNF5, scrive un **override** separato invece di modificare il `.repo`: l'`ls` lo verifica, e lo annoto perché resta anche dopo futuri aggiornamenti dei release. Se un ID non esiste (ad esempio i `-rawhide`), DNF lo segnala e quella riga non serviva.

**Atteso:** nessun errore per i quattro ID principali; `ls` mostra un file di override se `setopt` lo ha creato.

Dopo A o B, **ripeto la diagnosi del passo 1.2** finché non è tutta ✅.

### 1.4 🟢 Dati appstream per Discover

```bash
sudo dnf install rpmfusion-free-appstream-data rpmfusion-nonfree-appstream-data
```

Metadati che permettono a Discover di mostrare le app di RPM Fusion. Vanno installati **dopo** la diagnosi: nel thread, chi aveva rawhide attivo si ritrovava un `rpmfusion-free-appstream-data` della `45-1.fc45`.

**Atteso:** due pacchetti dai repo `rpmfusion-free`/`rpmfusion-nonfree`, versione `fc44`, non `fc45`. Se Discover non mostra subito le app: lo chiudo e lo riapro; in alternativa `pkcon refresh force` (tentativo ragionevole, non verificato).

### 1.5 🟢 Checkpoint dei repository

```bash
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
  echo "--- Pacchetti rpmfusion installati ---"
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
  echo "--- Appstream data ---"
  rpm -q rpmfusion-free-appstream-data rpmfusion-nonfree-appstream-data
  echo
  echo "--- Simulazione: distro-sync ---"
  sudo dnf distro-sync --assumeno 2>&1 | tail -n 20
  echo
  echo "--- Simulazione: risoluzione di ffmpeg completo (nessuna installazione) ---"
  sudo dnf install ffmpeg --allowerasing --assumeno 2>&1 | tail -n 30
  echo
} > "$CHECK"

echo "Checkpoint salvato in: $CHECK"
```

`sudo -v` chiede la password **prima** del blocco ridirezionato, così il prompt non finisce nel file. `$(( ... ))` è l'espansione aritmetica: da `44` calcola `45` senza scriverlo a mano. `--assumeno` risponde "no" alla conferma finale: ottengo una simulazione, non modifiche. `|| echo ...` stampa una riga leggibile quando `grep` non trova nulla.

**Atteso, riga per riga:**

| Sezione del file | ✅ Esito positivo | ❌ Esito negativo |
|---|---|---|
| Pacchetti `rpmfusion*` | release `44-N`, appstream `fc44` | `44-0.x`, oppure appstream `fc45` |
| Repo RPM Fusion | i quattro abilitati | disabilitati, o presenti solo i `-rawhide` |
| Repo rawhide abilitati | `nessuno` | qualsiasi riga |
| Pacchetti `fc45` | `nessuno` | qualunque pacchetto elencato |
| Override | assente, oppure file coerente con il Caso B | file che non ricordo di aver creato |
| Chiavi GPG | file e chiavi `rpmfusion` presenti | nessuna chiave RPM Fusion |
| `distro-sync` simulato | nulla da fare | lista di downgrade o sincronizzazioni |
| Risoluzione `ffmpeg` | riepilogo con pacchetti `fc44` | errori `nothing provides ... glibc` o riferimenti a `fc45` |

L'ultima riga è il test "vero": è esattamente l'errore che produce il problema dei repo rawhide. 🛑 **Non passo alla Fase 2 finché questo checkpoint non è pulito.**

Confronto con lo stato iniziale:

```bash
diff ~/post-install-info/repo-prima-rpmfusion-*.txt <(dnf repo list --all)
```

`<( ... )` (process substitution) presenta l'output di un comando a `diff` come se fosse un file. **Atteso:** differenze = solo le righe `rpmfusion-*` aggiunte.

---

## Fase 2 — Codec multimediali

*In breve:* un codec comprime (encoder) e decomprime (decoder) audio e video; H.264, H.265 e AAC sono i formati più diffusi coperti da brevetti, e Fedora non li distribuisce, quindi `ffmpeg-free` ne è privo. Il contenitore (`.mp4`, `.mkv`) è un'altra cosa: tiene insieme i flussi codificati.

### 2.1 🟢 Fotografia dei pacchetti multimediali

```bash
rpm -qa 'ffmpeg*' 'libav*' 'libsw*' 'gstreamer1*' 'PackageKit*' | sort \
  > ~/post-install-info/multimedia-pacchetti-prima-$(date +%Y-%m-%d).txt
```

**Atteso:** nessun output; il file elenca i pacchetti prima della sostituzione (tipicamente le versioni `*-free` di Fedora).

### 2.2 🟢 Checkpoint codec "prima" (baseline)

Questo script si lancia **due volte**: ora con `ETICHETTA=prima`, e dopo lo swap con `ETICHETTA=dopo`. La baseline mi dice cosa non funzionava e se qualcosa era già cambiato.

```bash
ETICHETTA=prima      # cambia in "dopo" quando lo rilanci a swap eseguita

CHECK=~/post-install-info/multimedia-checkpoint-$ETICHETTA-$(date +%Y-%m-%d_%H-%M).txt
NEXT=$(( $(rpm -E %fedora) + 1 ))

test_codec() {   # $1 = encoder video, $2 = estensione del contenitore
  local out="/tmp/codec-test-$1.$2"

  if ! command -v ffmpeg >/dev/null; then
    echo "[--] comando ffmpeg non trovato"
    return
  fi

  if ffmpeg -hide_banner -loglevel error -y \
       -f lavfi -i testsrc=duration=2:size=320x240:rate=25 \
       -f lavfi -i sine=frequency=440:duration=2 \
       -c:v "$1" -c:a aac "$out"; then
    echo "[OK]   codifica $1 + aac"
    echo "       stream nel file: $(ffprobe -v error -show_entries stream=codec_name \
          -of csv=p=0 "$out" | paste -sd' ')"
    if ffmpeg -v error -i "$out" -f null - ; then
      echo "[OK]   decodifica del file prodotto"
    else
      echo "[FAIL] decodifica del file prodotto"
    fi
  else
    echo "[FAIL] codifica $1 + aac"
  fi

  rm -f "$out"
}

{
  echo "=== Checkpoint multimedia ($ETICHETTA) — $(date '+%A %d %B %Y, %H:%M:%S') ==="
  echo
  echo "--- Fedora e kernel ---"
  cat /etc/fedora-release
  echo "Kernel: $(uname -r)"
  echo
  echo "--- Repository: RPM Fusion abilitati / rawhide abilitati (atteso: nessuno) ---"
  dnf repo list | grep -i rpmfusion
  dnf repo list | grep -i rawhide || echo "nessun repo rawhide abilitato"
  echo
  echo "--- Pacchetti ffmpeg/libav con vendor ---"
  rpm -qa 'ffmpeg*' 'libav*' 'libsw*' \
    --qf '%{NAME}-%{VERSION}-%{RELEASE}  [%{VENDOR}]\n' | sort
  echo
  echo "--- Presenza di libavcodec-freeworld ---"
  rpm -q libavcodec-freeworld || true
  echo
  echo "--- PackageKit-gstreamer-plugin ---"
  rpm -q PackageKit-gstreamer-plugin || true
  echo
  echo "--- Pacchetti installati costruiti per fc$NEXT (atteso: nessuno) ---"
  rpm -qa | grep -F ".fc$NEXT" || echo "nessuno"
  echo
  echo "--- Versione di ffmpeg ---"
  ffmpeg -version 2>/dev/null | head -n 1 || echo "ffmpeg non trovato"
  echo
  echo "--- Encoder presenti (libx264, libx265, aac) ---"
  ffmpeg -hide_banner -encoders 2>/dev/null | grep -E ' (libx264|libx265|aac) ' \
    || echo "nessuno dei tre trovato"
  echo
  echo "--- Decoder presenti (h264, hevc, aac) ---"
  ffmpeg -hide_banner -decoders 2>/dev/null | grep -E ' (h264|hevc|aac) ' \
    || echo "nessuno dei tre trovato"
  echo
  echo "--- Test funzionale H.264 + AAC ---"
  test_codec libx264 mp4
  echo
  echo "--- Test funzionale H.265 (HEVC) + AAC ---"
  test_codec libx265 mkv
  echo
  echo "--- GStreamer: decoder H.264 disponibile? ---"
  if command -v gst-inspect-1.0 >/dev/null; then
    gst-inspect-1.0 avdec_h264 >/dev/null 2>&1 \
      && echo "[OK]   avdec_h264 presente" \
      || echo "[--]   avdec_h264 non presente"
  else
    echo "[--] gst-inspect-1.0 non installato"
  fi
  echo
} > "$CHECK" 2>&1

echo "Checkpoint salvato in: $CHECK"
```

Note di sintassi: `test_codec() { ... }` definisce una **funzione** (`$1`, `$2` sono i suoi argomenti, `local` limita una variabile alla funzione); `if comando; then` esegue il ramo `then` se il comando esce con codice 0; `-f null -` decodifica tutto e butta il risultato, per dimostrare che il file è leggibile; `lavfi` con `testsrc` e `sine` genera video e audio sintetici, senza file esterni.

**Atteso (prima):**
- pacchetti `ffmpeg*`/`libav*` con vendor **Fedora**;
- `libavcodec-freeworld`: non installato (se non l'ho già installato per conto mio);
- encoder `libx264`/`libx265`: assenti, quindi test `[FAIL]` (oppure `[--] comando ffmpeg non trovato` se il CLI non è installato);
- `fc45`: nessuno.

Se `libavcodec-freeworld` risulta **già installato** insieme a `ffmpeg-libs`, prima di proseguire simulo la rimozione e leggo il riepilogo:

```bash
sudo dnf remove libavcodec-freeworld --assumeno
```

### 2.3 🛑 Simulare lo swap

```bash
sudo dnf swap ffmpeg-free ffmpeg --allowerasing --assumeno
```

**Cosa sta per succedere:** un sistema non ha "un" ffmpeg, ma un programma (`ffmpeg`) e un insieme di librerie (`libavcodec` e altre) che anche browser e lettori video usano. `swap` rimuove i pacchetti Fedora e installa quelli di RPM Fusion **in un'unica transazione**. Funziona perché: i file appartengono ai pacchetti (`rpm -qf "$(command -v ffmpeg)"` lo mostra); i pacchetti dichiarano conflitti che `--allowerasing` permette di sciogliere; DNF calcola tutto prima (risolutore `libsolv`), scarica e verifica le firme GPG, e solo dopo passa la transazione a RPM. Le applicazioni non se ne accorgono perché cercano la libreria per **nome** (soname), non per pacchetto.

**Atteso nel riepilogo (nessuna modifica, per `--assumeno`):**
- **Installing:** `ffmpeg`, `ffmpeg-libs` e dipendenze, dai repo `rpmfusion-free`/`rpmfusion-free-updates`, versioni `fc44`;
- **Removing:** i pacchetti `*-free` di FFmpeg (`ffmpeg-free` e le librerie corrispondenti). L'elenco esatto lo decide il riepilogo, non questa pagina;
- **Nulla di inatteso** tra le rimozioni: applicazioni che non c'entrano, componenti di KDE.

Se compaiono errori `nothing provides ... glibc` o pacchetti `fc45`, torno alla Fase 1.

### 2.4 🟢 Eseguire lo swap

```bash
sudo dnf swap ffmpeg-free ffmpeg --allowerasing
```

Non eseguo anche `dnf swap libavcodec-free libavcodec-freeworld`: secondo kwizart (thread di febbraio 2026) le due strade sono **alternative**, e il team RPM Fusion raccomanda quella con `ffmpeg` completo.

**Atteso:** la stessa transazione della simulazione, ora con conferma; al termine, "Complete" senza errori.

### 2.5 🟢 Gruppo `multimedia`

```bash
dnf group info multimedia

sudo dnf group upgrade multimedia \
  --setopt="install_weak_deps=False" \
  --exclude=PackageKit-gstreamer-plugin
```

`group info` mostra cosa contiene il gruppo (un insieme predefinito di pacchetti definito nei metadati dei repo). `install_weak_deps=False` evita le dipendenze "consigliate ma non indispensabili" (vale solo per questa esecuzione). `--exclude=PackageKit-gstreamer-plugin` impedisce di installare il componente che propone di cercare i codec mancanti via PackageKit. Che sia *necessario* non lo posso affermare; è innocuo.

**Atteso:** `group info` elenca pacchetti del gruppo; l'upgrade installa/aggiorna plugin GStreamer. Può anche concludersi con "nulla da fare", se tutto è già al suo posto.

### 2.6 🟢 Riavvio

```bash
sudo reboot
```

I programmi aperti durante lo swap continuano a usare le librerie già caricate in memoria. Il riavvio (o almeno la chiusura e riapertura di browser e lettori video) garantisce che usino quelle nuove.

### 2.7 🛑 Checkpoint codec "dopo" e confronto

Rilancio lo script del passo 2.2 con `ETICHETTA=dopo` (cambio solo la prima riga), poi:

```bash
diff ~/post-install-info/multimedia-checkpoint-prima-*.txt \
     ~/post-install-info/multimedia-checkpoint-dopo-*.txt

diff ~/post-install-info/multimedia-pacchetti-prima-*.txt \
     <(rpm -qa 'ffmpeg*' 'libav*' 'libsw*' 'gstreamer1*' 'PackageKit*' | sort)
```

**Atteso, riga per riga:**

| Sezione | Prima | ✅ Dopo |
|---|---|---|
| Vendor `ffmpeg`/`ffmpeg-libs` | Fedora | **RPM Fusion** |
| `libavcodec-freeworld` | non installato | non installato |
| Repo rawhide / pacchetti `fc45` | nessuno | nessuno |
| Encoder `libx264`, `libx265`, `aac` | assenti | tutti e tre presenti |
| Decoder `h264`, `hevc`, `aac` | da verificare | presenti |
| Test H.264 + AAC | `[FAIL]` | `[OK]` codifica **e** decodifica; stream `h264 aac` |
| Test H.265 + AAC | `[FAIL]` | `[OK]` codifica **e** decodifica; stream `hevc aac` |
| GStreamer `avdec_h264` | dipende | `[OK]` se il gruppo ha portato il plugin `libav`; altrimenti è un'informazione, non un errore |

Il valore del test sta nella combinazione: encoder elencato **e** file prodotto **e** file decodificabile. Un `[FAIL]` dopo lo swap si diagnostica leggendo l'errore di `ffmpeg` (qui ridotto a `-loglevel error`): di solito nomina l'encoder o la libreria che manca.

Nei `diff` dei pacchetti mi aspetto: righe `-` per i `*-free` di Fedora e righe `+` per `ffmpeg`/`ffmpeg-libs` di RPM Fusion, più eventuali plugin GStreamer aggiunti dal gruppo.

---

## Fuori da questa ricetta

**Accelerazione hardware (decodifica sulla GPU).** `ffmpeg` completo decodifica H.264/H.265 sulla CPU, non sulla GPU. Per una GPU AMD la parte rilevante sarebbe `mesa-va-drivers-freeworld` di RPM Fusion, che esiste per Fedora 44, ma le guide sono discordanti su come installarlo (`swap` o installazione diretta) e non l'ho verificato: lo tratto a parte, con `vainfo` come verifica.

---

## Riepilogo dei file prodotti in `~/post-install-info/`

| File | Contenuto |
|---|---|
| `fonti-rpmfusion-<data>.txt` + PDF delle pagine | da dove vengono i comandi |
| `repo-prima-rpmfusion-<data>.txt` | repository prima di RPM Fusion |
| `rpmfusion-checkpoint-<data_ora>.txt` | stato dei repo dopo l'installazione |
| `multimedia-pacchetti-prima-<data>.txt` | pacchetti multimediali prima dello swap |
| `multimedia-checkpoint-prima-<data_ora>.txt` | baseline dei codec |
| `multimedia-checkpoint-dopo-<data_ora>.txt` | stato dei codec dopo swap e riavvio |
