# Post-installazione — Firmware updates con `fwupd`

Gli aggiornamenti di `dnf` (vedi `primi-passi.md`) toccano il **software** del sistema operativo. Esiste però un altro strato che `dnf upgrade` non vede: il **firmware**, cioè il software che vive *dentro* i componenti hardware (UEFI/BIOS, SSD, controller USB4/Thunderbolt, dock, a volte touchpad e lettori di impronte). Su Fedora questo strato si aggiorna con `fwupd` e il suo client a riga di comando, `fwupdmgr`.

⚠️ **Stato di verifica.** Cosa `fwupd` offra per il mio Zenbook (UM3406KA) non l'ho ancora verificato sul sistema: dipende da cosa ASUS e i produttori dei singoli componenti pubblicano sul catalogo che `fwupd` consulta. Dove dico "mi aspetto", è una previsione da confermare con l'output reale.

---

## 1. Cos'è il firmware, e perché si aggiorna a parte

Un'analogia: il sistema operativo è il guidatore, il firmware è il **cervello interno di ogni componente**. Un SSD non è un blocchetto di memoria "stupido": ha un piccolo computer dentro che gestisce wear leveling, cache e controllo errori, e quel computer esegue un programma. Quel programma è il firmware, ed è un software come gli altri: ha bug e ha correzioni.

Il motivo per cui non passa da `dnf` è che **non vive sul disco di sistema**, ma in una memoria flash sul componente stesso (o sulla scheda madre, nel caso di UEFI/BIOS). Serve quindi qualcuno che sappia parlare con ciascun tipo di hardware, con il suo protocollo di aggiornamento. Quel qualcuno è `fwupd`.

Perché può valere la pena: i bug di firmware possono manifestarsi come problemi "misteriosi" lato Linux (sospensione che non riprende, dispositivi USB che si scollegano, SSD che rallenta), che si prestano a essere scambiati per bug del kernel o dei driver. Non è detto che il firmware sia *la* causa, ma è una delle cose da escludere presto, e controllarlo costa pochi minuti.

---

## 2. I pezzi in gioco: `fwupd`, `fwupdmgr`, LVFS

Come per i package manager (vedi `appendice-package-manager.md`), la confusione sulle sigle sparisce separando i ruoli:

| Pezzo | Cos'è | Analogia con DNF |
|---|---|---|
| **`fwupd`** | Il **demone** (servizio in background) che parla con l'hardware, scarica e applica i firmware. | Il motore di `dnf5` + RPM |
| **`fwupdmgr`** | Il **client a riga di comando**: gli dici cosa fare, lo passa al demone. | Il comando `dnf` che digiti |
| **LVFS** (*Linux Vendor Firmware Service*) | Il **catalogo online** dove i produttori caricano i firmware. È la fonte che `fwupd` consulta. | Un repository (vedi sotto: in `fwupd` si chiamano proprio *remotes*) |
| **Discover** | L'interfaccia grafica di KDE può mostrare anche gli aggiornamenti firmware. Non ho verificato se sul mio sistema sia attivo; qui uso il terminale. | Discover come front-end di PackageKit |

Due dettagli utili da fissare:

- `fwupd` non lavora in modo continuo: parte quando serve (di norma attivato da `fwupdmgr` o da Discover tramite D-Bus) e poi si ferma. Vederlo "inattivo" in `systemctl status fwupd` non è un problema.
- Il **LVFS** non è di `fwupd`: è un servizio separato, gestito nell'ambito della comunità Linux, dove ogni produttore decide *se* e *cosa* pubblicare. Il fatto che il mio portatile non compaia per il BIOS vuol dire che ASUS non lo pubblica lì (o non per questo modello), non che `fwupd` sia rotto.

Per vedere se `fwupd` è installato (su Fedora KDE mi aspetto di sì, ma lo controllo invece di fidarmi):

```bash
rpm -q fwupd
```

---

## 3. Come funziona, a grandi linee

Quello che succede quando lancio un aggiornamento:

1. **Metadati.** `fwupd` scarica dal LVFS un indice (*metadati*) che dice: "esiste il firmware versione X per il dispositivo con questo identificativo hardware". È lo stesso concetto di "catalogo" visto per la cache di DNF in `primi-passi.md`: non contiene i firmware, solo l'elenco.
2. **Confronto.** `fwupd` interroga l'hardware presente ("che versione di firmware hai?") e la confronta con l'indice. Se c'è una versione più nuova *per quel preciso dispositivo*, la segnala.
3. **Download e verifica.** Il file di firmware (un archivio `.cab`) viene scaricato e ne viene controllata l'integrità e la firma, prima di toccare qualsiasi cosa.
4. **Applicazione.** Dipende dal tipo di componente. Per molti dispositivi l'aggiornamento è immediato. Per **UEFI/BIOS** il firmware viene *messo in coda* e applicato **al riavvio successivo**, tramite il meccanismo UEFI *capsule*: `fwupd` copia il file nella partizione EFI (ESP) e dice al firmware di installarlo all'avvio seguente. È per questo che dopo un aggiornamento di BIOS il computer si riavvia in una schermata dedicata prima di arrivare al sistema.

---

## 4. Prima di aggiornare: avvertenze

Aggiornare firmware è diverso da aggiornare un pacchetto, perché **un'interruzione a metà può rendere inutilizzabile il componente** (nel caso peggiore, la scheda madre). Le buone pratiche che mi segno:

- **Portatile in carica**, con batteria sufficiente. `fwupd` in genere rifiuta di applicare aggiornamenti con batteria troppo bassa, ma non faccio affidamento solo su quello.
- **Non spegnere né interrompere** durante la schermata di aggiornamento dopo il riavvio. Può durare qualche minuto e sembrare "bloccata": non è un buon momento per forzare lo spegnimento.
- **Leggere cosa sta per fare.** Come per le transazioni DNF, vale la pena leggere il riepilogo: nome del dispositivo, versione attuale, versione nuova e note di rilascio.
- **Fare un backup** dei dati importanti prima di un aggiornamento di SSD. Di norma va tutto bene, ma l'aggiornamento di un disco è l'operazione in cui un errore costa di più.
- Se `Secure Boot` è attivo, nulla di particolare da fare: `fwupd` su Fedora usa un binario EFI firmato. Se un aggiornamento fallisce, è uno dei punti da guardare.

---

## 5. I comandi della procedura, uno per uno

```bash
sudo fwupdmgr refresh --force
sudo fwupdmgr get-updates
sudo fwupdmgr update
```

### 5.1 `fwupdmgr refresh --force`

Scarica i metadati aggiornati dal LVFS. Senza `--force`, `fwupd` può decidere di non riscaricare se considera quelli che ha ancora "freschi" (stesso discorso della cache di DNF con `--refresh` in `primi-passi.md`). Con `--force` ignoro quel controllo. Subito dopo l'installazione è quello che voglio: la lista più recente possibile.

**Mi aspetto:** un messaggio di aggiornamento dei metadati per il remote `lvfs` (o un avviso che sono già aggiornati). La prima volta `fwupdmgr` può chiedere se abilitare il remote LVFS: va accettato, perché senza non ha nulla da consultare.

### 5.2 `fwupdmgr get-updates`

È una **simulazione**: elenca cosa si può aggiornare, senza toccare nulla. È l'equivalente di guardare il riepilogo prima di premere Invio. Per ogni dispositivo mostra versione installata, versione disponibile, gravità e note di rilascio.

Se non c'è niente da aggiornare, il comando lo dice e, per quanto ricordo, termina con un **codice di uscita diverso da 0**: non è un errore, è un modo per gli script di distinguere "niente da fare". Non l'ho ancora verificato.

### 5.3 `fwupdmgr update`

Scarica e applica gli aggiornamenti trovati. Per ciascuno chiede conferma e, se serve un riavvio, lo propone alla fine. Non c'è `-y` per ora: scelgo di leggere ogni richiesta.

Gli aggiornamenti UEFI/BIOS vengono **programmati** per il riavvio successivo (sezione 3, punto 4): il sistema non è ancora aggiornato finché non riparto.

---

## 6. Il resto di `fwupdmgr`: i sottocomandi che uso più spesso

Oltre ai tre della procedura, `fwupdmgr` ha tanti sottocomandi. Per sapere cosa c'è nella versione installata:

```bash
fwupdmgr --help
```

Quelli che mi servono davvero:

| Comando | A cosa serve |
|---|---|
| `fwupdmgr get-devices` | Elenca i dispositivi che `fwupd` riconosce, con versione di firmware, produttore e **flag** (ad esempio se sono aggiornabili). È il comando per capire *cosa vede* sul mio hardware. |
| `fwupdmgr get-updates` | Elenca gli aggiornamenti disponibili, senza applicarli. |
| `fwupdmgr update` | Applica gli aggiornamenti. |
| `fwupdmgr get-history` | Cronologia degli aggiornamenti già eseguiti, con esito. Utile come "registro" del sistema. |
| `fwupdmgr get-releases` | Elenca *tutte* le versioni note per un dispositivo (non solo quella più recente). Serve se voglio tornare indietro. |
| `fwupdmgr downgrade` | Installa una versione **precedente**, se il produttore la rende disponibile. |
| `fwupdmgr get-remotes` | Elenca i *remotes*, cioè le fonti di firmware (come `dnf repo list` per i repository). |
| `fwupdmgr security` | Riepilogo di sicurezza dell'hardware (il progetto lo chiama **HSI**, *Host Security ID*): controlla cose come Secure Boot, protezioni del firmware e altro, e assegna un livello. Mi interessa come fotografia, non come verdetto. |

### Come si legge `get-devices`

Un pezzo di output tipico ha questa forma (esempio generico, non il mio):

```text
Nome del dispositivo
│
├─ Device ID:          <identificativo>
├─ Summary:            ...
├─ Current version:    1.2.3
├─ Vendor:             ...
├─ Update Error:       ...      (solo se c'è un problema)
└─ Flags:              updatable | supported | ...
```

Le righe che guardo: **Current version** (cosa ho oggi) e **Flags**. Se un dispositivo non ha il flag di "aggiornabile" non vuol dire che sia difettoso: significa solo che `fwupd` lo conosce ma non può aggiornarlo, o che il produttore non pubblica nulla per quel componente.

### I *remotes*: `lvfs` e `lvfs-testing`

```bash
fwupdmgr get-remotes
```

Mi aspetto almeno `lvfs` (abilitato) e, di norma, un remote `lvfs-testing` **disabilitato**. Quest'ultimo contiene firmware in fase di prova, prima della pubblicazione ufficiale. Per l'uso quotidiano lo lascio spento: sul firmware preferisco non essere io il collaudatore.

---

## 7. Condivisione dei report

Dopo un aggiornamento, `fwupdmgr` può chiedere se vuoi **inviare un report anonimo** al LVFS con l'esito (riuscito o fallito) e informazioni sul dispositivo. Serve ai produttori e alla comunità per capire se un firmware dà problemi. È un'opzione: rispondo secondo come mi sento riguardo alla condivisione di dati sull'hardware. Se ho dubbi sul contenuto, il sottocomando `fwupdmgr report-history` può mostrare cosa verrebbe inviato; vale la pena leggerlo prima, in particolare la prima volta.

---

## 8. Sul mio Zenbook: cosa mi aspetto, cosa non so

Cose che **non posso affermare** senza prima averle viste sul sistema:

- se il **BIOS/UEFI ASUS** è distribuito via LVFS per l'UM3406KA;
- quali altri componenti (SSD, controller USB4, ecc.) risultano aggiornabili;
- se ci sono già aggiornamenti in attesa.

Se `get-devices` mostra il BIOS ma `get-updates` non propone nulla, è un'informazione valida: significa che al momento non c'è niente di più recente sul LVFS, non necessariamente che sia l'ultima versione assoluta esistente. Per confrontare la versione di BIOS che ho con quella pubblicata sul sito del produttore:

```bash
cat /sys/class/dmi/id/bios_version
cat /sys/class/dmi/id/bios_date
```

(Sono due file "virtuali" del kernel che espongono i dati DMI/SMBIOS della macchina; si leggono senza `sudo`.)

Se ASUS non pubblica il BIOS su LVFS, l'aggiornamento passerà dai suoi canali (sito del produttore, o strumenti integrati nel BIOS stesso). Non ho verificato quali opzioni ci siano per questo modello, quindi per ora lascio la questione aperta: la affronterò a parte, se serve, con la fonte ufficiale alla mano.

---

## 9. Fotografia prima e dopo

Come per il resto della post-installazione, salvo lo stato **prima** e **dopo**, per poi confrontarlo con `diff`.

```bash
mkdir -p ~/post-install-info
ETICHETTA=prima      # cambia in "dopo" quando rilanci a riavvio fatto

CHECK=~/post-install-info/firmware-checkpoint-$ETICHETTA-$(date +%Y-%m-%d_%H-%M).txt

{
  echo "=== Checkpoint firmware ($ETICHETTA) — $(date '+%A %d %B %Y, %H:%M:%S') ==="
  echo

  echo "--- Versione di fwupd ---"
  rpm -q fwupd
  echo

  echo "--- BIOS/UEFI (dal kernel) ---"
  echo "Versione : $(cat /sys/class/dmi/id/bios_version)"
  echo "Data     : $(cat /sys/class/dmi/id/bios_date)"
  echo

  echo "--- Remotes ---"
  fwupdmgr get-remotes 2>&1
  echo

  echo "--- Dispositivi riconosciuti ---"
  fwupdmgr get-devices 2>&1
  echo

  echo "--- Aggiornamenti disponibili ---"
  fwupdmgr get-updates 2>&1 || true
  echo

  echo "--- Cronologia ---"
  fwupdmgr get-history 2>&1 || true
  echo
} > "$CHECK"

echo "Checkpoint salvato in: $CHECK"
```

Note sulla sintassi, per chi non l'ha ancora incontrata:

- `2>&1` porta anche i messaggi di errore (*stderr*) nello stesso flusso dell'output normale (*stdout*), così finiscono nel file invece che sul terminale.
- `|| true` evita che un comando che esce con codice diverso da 0 (come `get-updates` quando non c'è nulla da fare) venga letto come un fallimento: serve se in futuro volessi inserire lo script in un contesto che si ferma al primo errore.
- Qui non uso `sudo` per i comandi in sola lettura. Se un comando chiede autorizzazioni, `fwupd` passa da *polkit* (la finestra o il prompt della password): non l'ho verificato su questo sistema, quindi se un'informazione manca dal file lo rilancio con `sudo`.

Per confrontare i due stati:

```bash
diff ~/post-install-info/firmware-checkpoint-prima-*.txt \
     ~/post-install-info/firmware-checkpoint-dopo-*.txt
```

**Atteso dopo un aggiornamento riuscito:** una versione diversa nel dispositivo aggiornato e una nuova voce in `get-history`, con esito positivo. Se compare un errore, lo copio per intero prima di cercare altro.

---

## 10. Periodicità

Una volta subito dopo l'installazione e poi di tanto in tanto: non ho bisogno di una cadenza rigida. Alcune distribuzioni aggiornano i metadati in automatico con un timer di `systemd`; se lo fa anche la mia, lo verifico invece di darlo per scontato:

```bash
systemctl list-timers | grep -i fwupd
```

Anche in quel caso, il timer **aggiorna l'indice**, non applica aggiornamenti: quelli restano un'azione che faccio io.

---

## 11. Riepilogo operativo

- [ ] Verificare che `fwupd` sia installato: `rpm -q fwupd`
- [ ] Collegare il portatile all'alimentazione, con backup dei dati importanti
- [ ] Salvare il checkpoint con `ETICHETTA=prima`
- [ ] `sudo fwupdmgr refresh --force`
- [ ] `fwupdmgr get-devices`: cosa vede `fwupd` sul mio hardware?
- [ ] `sudo fwupdmgr get-updates` e **leggere** cosa viene proposto
- [ ] `sudo fwupdmgr update`, leggendo ogni richiesta di conferma
- [ ] Riavviare quando richiesto e **non interrompere** la schermata di aggiornamento
- [ ] Rilanciare il checkpoint con `ETICHETTA=dopo` e confrontare con `diff`
- [ ] Annotare qui cosa `fwupd` ha effettivamente visto e aggiornato sul mio Zenbook

