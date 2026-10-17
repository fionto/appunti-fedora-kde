# Post-installazione — Parte 3: codec multimediali

Con RPM Fusion abilitato (vedi `rpm-fusion.md`), il passo successivo è ripristinare il supporto completo ai formati audio e video più diffusi. In questo capitolo spiego cosa sono i codec, **cosa stiamo sostituendo** quando lancio `dnf swap`, come fa un comando DNF a cambiare librerie di sistema, e soprattutto come verifico che il lavoro sia stato fatto bene, con un file di archivio prima e dopo.

⚠️ **Prerequisito:** questo capitolo dà per buono che i repository RPM Fusion siano quelli giusti (release `44-N`, quattro repo abilitati, nessun `rawhide`). Se non l'ho verificato, comincio da lì: i pacchetti di questo capitolo arrivano proprio da quei repository.

---

## 1. A cosa serve un codec

Un video grezzo è enorme: pochi minuti di filmato non compresso occupano gigabyte. Per questo audio e video vengono quasi sempre **compressi**, e un **codec** (da *co*der-*dec*oder) è il programma, o più precisamente l'algoritmo con la sua implementazione, che fa due cose:

- **encoder**: prende il video/audio grezzo e lo comprime in un formato (quando registro, esporto o converto);
- **decoder**: fa il percorso inverso, trasformando i dati compressi in immagini e suoni riproducibili (quando guardo o ascolto).

Un'analogia: è come spedire un mobile in kit. L'**encoder** lo smonta e lo impacchetta nel modo più compatto possibile; il **decoder** ha le istruzioni per rimontarlo. Funziona solo se i due parlano *lo stesso* "linguaggio di imballaggio". Quel linguaggio è il formato, per esempio H.264, H.265 (HEVC), AAC, MP3.

Una confusione comune da chiarire subito: **codec e contenitore sono cose diverse**.

| Cosa | Esempi | Ruolo |
|---|---|---|
| **Contenitore** | `.mp4`, `.mkv`, `.webm` | La "scatola" che tiene insieme video, audio, sottotitoli |
| **Codec video** | H.264, H.265/HEVC, VP9, AV1 | Come è compresso il video dentro la scatola |
| **Codec audio** | AAC, MP3, Opus, FLAC | Come è compreso l'audio dentro la scatola |

Un file `.mp4` può quindi contenere codec diversi, ed è per questo che due `.mp4` apparentemente identici possono comportarsi in modo diverso sullo stesso computer: uno si apre, l'altro no.

### Perché Fedora non li include tutti

Molti codec molto diffusi (H.264, H.265, AAC, tra i più noti) sono coperti da **brevetti** o da licenze restrittive. Per scelta del progetto, Fedora non li distribuisce nei suoi repository ufficiali: per questo fornisce `ffmpeg-free`, una versione di FFmpeg (la libreria/programma di riferimento per manipolare audio e video) in cui questi codec sono lasciati fuori. I formati liberi da questo punto di vista, come Opus, Vorbis, FLAC, VP9, AV1, in genere funzionano già senza fare nulla.

Una nota di precisione: nel repository Fedora esiste anche `fedora-cisco-openh264` (lo vedo nell'elenco dei repo di altri utenti sui forum), che distribuisce il codec H.264 di Cisco. È una strada separata, pensata soprattutto per browser e videochiamate, e non sostituisce quello che faccio qui. Non lo uso in questo capitolo.

---

## 2. Fonti e stato della questione (consultato il 17 ottobre 2026)

| Fonte | Cosa mi dà |
|---|---|
| [rpmfusion.org/Howto/Multimedia](https://rpmfusion.org/Howto/Multimedia) | La pagina del progetto sui multimedia: il riferimento per i comandi. |
| [Fedora Discussion — "How do I enable h265 HEVC playback in Fedora 43 or 44?"](https://discussion.fedoraproject.org/t/how-do-i-enable-h265-hevc-playback-in-fedora-43-or-44-2026/147870) | Un messaggio di febbraio 2026 di chi gestisce i file di RPM Fusion che chiarisce la differenza tra le due strade (vedi sotto). |
| [Fedora Wiki — Hardware Video Acceleration](https://fedoraproject.org/wiki/Hardware_Video_Acceleration) | Spiega cosa serve per l'accelerazione hardware su AMD. Non fa parte di questo capitolo (vedi sezione 6). |

**Il punto che conta.** Nel thread sopra, kwizart (nel thread risulta tra chi cura i file di RPM Fusion) scrive che ci sono **due strade alternative**, non complementari:

1. `sudo dnf swap ffmpeg-free ffmpeg --allowerasing`: si passa a FFmpeg completo di RPM Fusion;
2. installare `libavcodec-freeworld`, che **completa** `ffmpeg-free` di Fedora tenendolo al suo posto.

Il team RPM Fusion, sempre secondo quel messaggio, raccomanda con decisione la prima, per evitare i disallineamenti di versione che si creano ogni tanto quando la versione Fedora resta indietro. Anche altre guide concordano che, dopo lo swap a `ffmpeg` completo, `libavcodec-freeworld` non serve più.

Questo mi interessa perché la guida che avevo seguito eseguiva **entrambe** le strade. Nella sezione 4 spiego come la gestisco.

---

## 3. Prima di cambiare qualcosa: la fotografia

Come per il resto della post-installazione, prima di toccare qualcosa fotografo lo stato. Qui mi servono due cose:

```bash
mkdir -p ~/post-install-info

rpm -qa 'ffmpeg*' 'libav*' 'libsw*' 'gstreamer1*' 'PackageKit*' | sort \
  > ~/post-install-info/multimedia-pacchetti-prima-$(date +%Y-%m-%d).txt
```

Le virgolette attorno ai pattern (`'ffmpeg*'`) impediscono alla shell di espandere l'asterisco prima che arrivi a `rpm`, come già visto per DNF. Il file mi dice **quali pacchetti multimediali c'erano** prima della sostituzione.

La seconda cosa è lo **script di checkpoint** della sezione 7, che lancio ora con `ETICHETTA=prima`. Sembra un controsenso testare qualcosa che so già che non funziona, ma ha due vantaggi: ottengo una *baseline* con cui confrontare il dopo, e scopro se qualcosa funzionava già (o se la guida che avevo seguito aveva già cambiato delle cose).

---

## 4. Cosa sostituisco: `dnf swap`

### 4.1 I due comandi, e come li gestisco

La guida originale proponeva:

```bash
sudo dnf swap ffmpeg-free ffmpeg --allowerasing
sudo dnf swap libavcodec-free libavcodec-freeworld --allowerasing
```

Per quello che ho trovato, la **seconda riga non va aggiunta** quando si fa la prima: sono le due strade alternative descritte sopra. Mi limito quindi a:

```bash
sudo dnf swap ffmpeg-free ffmpeg --allowerasing
```

Se invece ho **già eseguito** anche la seconda, non la annullo "a occhio". Guardo prima cosa c'è:

```bash
rpm -q ffmpeg ffmpeg-libs ffmpeg-free libavcodec-free libavcodec-freeworld
```

Se `libavcodec-freeworld` risulta installato insieme a `ffmpeg-libs`, uso una simulazione per vedere cosa succederebbe a toglierlo, **senza toccare nulla**:

```bash
sudo dnf remove libavcodec-freeworld --assumeno
```

Leggo il riepilogo (cosa verrebbe rimosso a cascata) e decido dopo. `--assumeno` fa rispondere "no" alla conferma finale: ottengo la simulazione, nessuna modifica.

### 4.2 Che cosa stiamo "sostituendo"

Un sistema Linux non ha "un" ffmpeg: ha un insieme di **librerie** (`libavcodec`, `libavformat`, `libswscale` e altre) che contengono il codice dei codec, più un **programma a riga di comando** (`ffmpeg`) che le usa. Anche molte applicazioni grafiche, come lettori video e browser, usano quelle librerie di sistema invece di portarsi dietro le proprie: per questo la sostituzione ha effetto su tutto il sistema, non solo sul comando `ffmpeg`.

Schema semplificato:

| | Versione Fedora | Versione RPM Fusion |
|---|---|---|
| Programma | `ffmpeg-free` | `ffmpeg` |
| Librerie | `libavcodec-free`, `libavformat-free`, ... | `ffmpeg-libs` (e dipendenze) |
| Codec | Quelli senza vincoli di brevetto | Anche H.264, H.265, AAC ecc. |

Non do per scontato l'elenco preciso dei pacchetti coinvolti: **è il riepilogo della transazione a dirmelo**. Per questo prima simulo:

```bash
sudo dnf swap ffmpeg-free ffmpeg --allowerasing --assumeno
```

Nel riepilogo cerco tre cose: i pacchetti **Installing** (devono arrivare da `rpmfusion-free`/`rpmfusion-free-updates`), i pacchetti **Removing** (quali `*-free` spariscono) e, soprattutto, che tra le rimozioni non compaia nulla di inatteso: applicazioni che non c'entrano, pezzi di KDE.

### 4.3 Come fa un comando DNF a sostituire file di sistema

È la domanda giusta, perché `dnf swap` sembra magia: scrivo una riga e delle librerie di sistema cambiano. In realtà non cambia nessun file "a mano": cambiano dei **pacchetti**, e i file seguono. Il meccanismo ha quattro livelli.

**1. I file appartengono ai pacchetti.** Il database di RPM sa, per ogni file di sistema, a quale pacchetto appartiene. Posso chiederglielo:

```bash
rpm -qf "$(command -v ffmpeg)"   # a quale pacchetto appartiene il comando ffmpeg
rpm -ql ffmpeg-free              # quali file contiene un pacchetto (se installato)
```

(`command -v ffmpeg` stampa il percorso del comando; racchiuso in `$( )` diventa l'argomento di `rpm -qf`.)

Dunque "sostituire una libreria" significa **togliere il pacchetto che la possiede e installarne un altro che fornisce quella stessa funzione**.

**2. I pacchetti dichiarano cosa forniscono e cosa non tollerano.** Nei metadati di ogni pacchetto ci sono dipendenze, ma anche **conflitti**: un pacchetto può dichiarare "non posso coesistere con X", tipicamente perché i due installerebbero gli stessi file o fornirebbero lo stesso componente. Se provo un semplice `dnf install ffmpeg` mentre c'è `ffmpeg-free`, DNF si ferma e mi segnala il conflitto.

**3. `--allowerasing` e `swap` sciolgono il conflitto, in modo esplicito.** `dnf swap A B` equivale a dire: "in **un'unica transazione** rimuovi A e installa B". Il flag `--allowerasing` dà a DNF il permesso di rimuovere i pacchetti in conflitto. L'avviso della guida originale, "non spaventarti", è giusto a metà: il flag non è pericoloso in sé, ma permette a DNF di cancellare cose, ed è per questo che ho voluto prima la simulazione.

**4. La transazione viene calcolata prima e poi eseguita.** DNF non procede passo per passo "sperando che vada bene":
- il risolutore di dipendenze (`libsolv`, già incontrato nell'appendice) calcola **prima** l'intero insieme di pacchetti da togliere e mettere;
- DNF scarica tutto e **verifica le firme GPG**;
- solo a quel punto passa la transazione a RPM, che controlla i conflitti tra i file e scrive il nuovo stato.

Per le applicazioni la sostituzione è trasparente per una ragione precisa: i programmi non cercano "il pacchetto `libavcodec-free`", cercano una **libreria con un certo nome** (per esempio `libavcodec.so.NN`, il cosiddetto *soname*). Se dopo la swap quel nome è fornito dal pacchetto RPM Fusion, il programma lo trova e non si accorge della differenza, se non perché ora sa decodificare più formati.

⚠️ **Programmi già in esecuzione.** Un programma aperto durante la swap continua a usare le librerie vecchie, già caricate in memoria. Per vedere l'effetto serve **riavviare le applicazioni** (browser, lettori video) o, più semplicemente, riavviare il sistema.

---

## 5. Il gruppo `multimedia` e le opzioni

La guida originale proseguiva con:

```bash
sudo dnf update @multimedia --setopt="install_weak_deps=False" --exclude=PackageKit-gstreamer-plugin
```

Nella forma esplicita di DNF5 (vedi `nuovo-in-fedora-44.md`) la scrivo così:

```bash
sudo dnf group upgrade multimedia \
  --setopt="install_weak_deps=False" \
  --exclude=PackageKit-gstreamer-plugin
```

Prima di eseguirla guardo, come per qualunque gruppo, cosa contiene:

```bash
dnf group info multimedia
```

E poi riguardo cosa fa ciascun pezzo, dato che alcune spiegazioni della guida originale non mi tornavano del tutto:

- **`group upgrade multimedia`**: opera sul **gruppo** `multimedia`, un insieme predefinito di pacchetti definito nei metadati dei repository (non un gruppo "creato" dal sistema). Con RPM Fusion abilitato, a quel gruppo si aggiungono anche pacchetti del repository di terze parti, ed è questo che lo rende utile qui.
- **`--setopt="install_weak_deps=False"`**: le *dipendenze deboli* sono pacchetti "consigliati" ma non indispensabili. Con questa opzione DNF **non** li installa: ottengo il gruppo senza il contorno di extra. L'opzione vale solo per questa esecuzione, non modifica `dnf.conf`.
- **`--exclude=PackageKit-gstreamer-plugin`**: qui la guida originale dava una spiegazione che non riesco a confermare (diceva che serve a impedire a PackageKit di "ripristinare i plugin restrittivi"). Per quello che ricostruisco, `PackageKit-gstreamer-plugin` è il componente che, quando un'applicazione GStreamer non trova un codec, propone di **cercarlo e installarlo** tramite PackageKit. Escluderlo significa **non averlo**, e quindi non ricevere quelle proposte. Controllo io stesso la descrizione del pacchetto:

```bash
dnf info PackageKit-gstreamer-plugin
```

Non posso affermare che sia necessario, ma è un'opzione che le guide ufficiali di RPM Fusion hanno proposto a lungo, ed è innocua.

---

## 6. Cosa questo capitolo non fa: l'accelerazione hardware

La guida originale parlava di "hardware acceleration", ma **nessuno dei comandi di sopra la abilita**. `ffmpeg` completo permette al *software* di decodificare H.264/H.265 sulla CPU; l'accelerazione hardware è un'altra cosa: la decodifica fatta dalla GPU, che consuma meno energia e CPU, tramite un'interfaccia chiamata VA-API.

Per una GPU AMD come quella del mio Zenbook, a quanto leggo la parte rilevante è il pacchetto `mesa-va-drivers-freeworld` di RPM Fusion (esiste per Fedora 44 nel repository free). Ho però trovato indicazioni **discordanti** su come installarlo su Fedora 44 (alcune guide parlano di `swap`, altre di installazione diretta). Non voglio scriverlo senza averlo verificato sul sistema, quindi lo lascio fuori da questo capitolo: se serve, gli dedico un capitolo a parte, con `vainfo` per la verifica.

---

## 7. Verifica e checkpoint

### 7.1 Verifica rapida

La verifica della guida originale era:

```bash
ffmpeg -encoders 2>/dev/null | grep -E 'libx264|libx265|aac'
```

È un buon primo controllo, ma ha due limiti:
- `aac` è una stringa troppo generica: compare anche in altre voci e potrebbe trovare qualcosa anche senza i codec che mi interessano;
- mi dice che l'encoder è **elencato**, non che **funziona** né che i file prodotti sono leggibili.

La versione più precisa, che cerca le due voci esatte:

```bash
ffmpeg -hide_banner -encoders 2>/dev/null | grep -E ' (libx264|libx265|aac) '
```

Gli spazi attorno al gruppo evitano i falsi positivi su nomi simili. Ma il controllo che fa fede è un altro.

### 7.2 Il test funzionale: produrre e rileggere un file

Invece di fidarmi dell'elenco, genero un breve filmato di prova con un **video sintetico** (`testsrc`) e un **tono audio** (`sine`), lo codifico con H.264 + AAC e poi con H.265 + AAC, e controllo due cose: che la codifica riesca, e che il file si **decodifichi** senza errori. Nessun file esterno richiesto.

### 7.3 Lo script di checkpoint

Lo uso due volte: con `ETICHETTA=prima` prima della swap, con `ETICHETTA=dopo` dopo. Alcune note sulla sintassi:

- `test_codec() { ... }` definisce una **funzione** bash: un blocco di comandi con un nome, richiamabile più volte. `$1` e `$2` sono i suoi argomenti.
- `local` dichiara una variabile visibile solo dentro la funzione.
- `if comando; then ... else ... fi` esegue il ramo `then` se il comando termina con **codice di uscita 0** (cioè "successo"). È così che capisco se `ffmpeg` ha funzionato, senza dover leggere l'output.
- `-f null -` dice a `ffmpeg` di decodificare tutto e buttare via il risultato: serve solo a dimostrare che il file è decodificabile.

```bash
ETICHETTA=prima      # cambia in "dopo" quando rilanci a swap eseguita

mkdir -p ~/post-install-info
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

  echo "--- Presenza di libavcodec-freeworld (in alternativa a ffmpeg completo) ---"
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

### 7.4 Come leggere il risultato

| Sezione | Prima della swap (atteso) | Dopo la swap (atteso) |
|---|---|---|
| Vendor dei pacchetti | Fedora | `ffmpeg`, `ffmpeg-libs` con vendor **RPM Fusion** |
| `libavcodec-freeworld` | non installato | non installato (se ho seguito solo la strada 1) |
| `fc$NEXT` | nessuno | nessuno |
| Encoder `libx264`/`libx265` | assenti | presenti |
| Test H.264 + AAC | `[FAIL]` (o `ffmpeg` assente) | `[OK]` con codifica e decodifica |
| Test H.265 + AAC | `[FAIL]` (o `ffmpeg` assente) | `[OK]` con codifica e decodifica |
| GStreamer `avdec_h264` | dipende | `[OK]` se il gruppo `multimedia` ha portato il plugin `libav`; altrimenti è un'informazione, non un errore |

Il valore del test funzionale sta nella combinazione: encoder elencato **e** file prodotto **e** file decodificabile. Se un `[FAIL]` compare dopo la swap, leggo il messaggio di errore di `ffmpeg` (che qui ho ridotto a `-loglevel error`): di solito dice esplicitamente quale encoder o libreria manca.

Per confrontare i due stati:

```bash
diff ~/post-install-info/multimedia-checkpoint-prima-*.txt \
     ~/post-install-info/multimedia-checkpoint-dopo-*.txt
```

E per confrontare l'elenco dei pacchetti con quello della sezione 3:

```bash
diff ~/post-install-info/multimedia-pacchetti-prima-*.txt \
     <(rpm -qa 'ffmpeg*' 'libav*' 'libsw*' 'gstreamer1*' 'PackageKit*' | sort)
```

`<( ... )` è la *process substitution* già vista: bash esegue il comando e lo passa a `diff` come se fosse un file.

---

## 8. Riepilogo operativo

- [ ] Verificare i repository RPM Fusion (`rpm-fusion.md`): release `44-N`, nessun `rawhide`
- [ ] Salvare la pagina `rpmfusion.org/Howto/Multimedia` in PDF in `~/post-install-info/`
- [ ] Salvare l'elenco dei pacchetti multimediali (sezione 3)
- [ ] Lanciare lo script di checkpoint con `ETICHETTA=prima`
- [ ] Simulare: `sudo dnf swap ffmpeg-free ffmpeg --allowerasing --assumeno` e leggere il riepilogo
- [ ] Eseguire lo swap vero (senza `libavcodec-freeworld`)
- [ ] Controllare `dnf group info multimedia`, poi `dnf group upgrade multimedia` con le opzioni della sezione 5
- [ ] Riavviare le applicazioni o il sistema
- [ ] Rilanciare lo script con `ETICHETTA=dopo` e confrontare i due file con `diff`
- [ ] Se serve la decodifica via GPU: capitolo a parte (sezione 6)
