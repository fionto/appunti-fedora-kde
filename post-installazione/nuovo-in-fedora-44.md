# Prima di modificare il sistema: cosa cambia in Fedora 44

Prima di iniziare qualsiasi attività di post-installazione, conviene fermarsi un momento e capire **che cosa è cambiato in Fedora 44** rispetto alle versioni precedenti. Molte guide che si trovano online sono ancora perfettamente valide dal punto di vista concettuale, ma possono utilizzare comandi, nomi di servizi o procedure che appartengono a versioni precedenti di Fedora. Questo è particolarmente importante per una distribuzione come Fedora, dove alcuni componenti fondamentali del sistema evolvono rapidamente.

Nel caso di Fedora KDE Plasma 44 ci sono almeno due cambiamenti che è utile avere presenti fin dall'inizio:

1. **DNF 5 è ormai il riferimento per la gestione dei pacchetti.**
2. **Fedora KDE Plasma 44 introduce Plasma Login Manager e Plasma Setup.**

---

## 1. DNF 5: attenzione alle vecchie guide

### 1.1 Non è semplicemente "il vecchio DNF con un numero diverso"

Fedora utilizza DNF 5 come gestore dei pacchetti predefinito già a partire da Fedora 41. Con Fedora 44, quindi, le procedure di amministrazione dei pacchetti devono essere pensate principalmente in termini di **DNF 5**. Per l'utente questo significa che molti comandi quotidiani rimangono familiari:

```bash
sudo dnf install <pacchetto>
sudo dnf remove <pacchetto>
sudo dnf upgrade
```

ma alcune funzionalità meno comuni, alias e sintassi utilizzate nelle vecchie guide non sono più equivalenti. Il problema è particolarmente evidente quando si cercano online procedure scritte per Fedora 38, 39, 40 o versioni ancora precedenti. Una guida può quindi essere **corretta nel concetto ma non necessariamente corretta nella sintassi per Fedora 44**.

---

## 1.2 I gruppi di pacchetti

Un esempio importante riguarda i package group. I **package group** sono insiemi logici di pacchetti RPM che Fedora mantiene raggruppati perché svolgono una funzione comune o perché sono pensati per essere installati insieme. Un gruppo può, per esempio, raccogliere i pacchetti necessari a un determinato ambiente desktop, a una categoria di software o a una particolare funzionalità del sistema.

È importante però distinguere i package group dai normali pacchetti installati sul sistema. **Fedora non raggruppa automaticamente tutti i pacchetti installati in gruppi in base al loro utilizzo**: i gruppi sono metadati definiti nei repository e rappresentano insiemi predefiniti di pacchetti. L'appartenenza a un gruppo non significa quindi che ogni pacchetto installato faccia necessariamente parte di un gruppo, né che il sistema crei dinamicamente nuovi gruppi in base a ciò che viene installato.

Un gruppo può inoltre contenere pacchetti già installati, pacchetti non ancora installati e pacchetti considerati opzionali o predefiniti all'interno del gruppo. Per questo motivo l'operazione di installazione di un gruppo non equivale semplicemente a installare "tutti i pacchetti Fedora di una certa categoria".

È possibile esplorare i gruppi disponibili con:

```bash
dnf group list
```

e ottenere maggiori informazioni su un gruppo specifico con:

```bash
dnf group info <nome-gruppo>
```

Questo è particolarmente utile quando si segue una guida che suggerisce di installare un package group: prima di eseguire il comando, è possibile verificare **quali pacchetti appartengono effettivamente al gruppo** e capire quindi cosa verrà aggiunto al sistema.

Nelle vecchie guide Fedora è facile incontrare comandi come:

```bash
sudo dnf groupupdate @multimedia
```

Questa sintassi mescola due elementi della vecchia interfaccia di DNF:

* l'alias `groupupdate`;
* la notazione `@` utilizzata per identificare un gruppo.

Con DNF 5 è preferibile utilizzare esplicitamente i nuovi sottocomandi:

```bash
sudo dnf group list
sudo dnf group info <gruppo>
sudo dnf group install <gruppo>
sudo dnf group remove <gruppo>
sudo dnf group upgrade <gruppo>
```

DNF 5 ha infatti rimosso diversi alias storici, tra cui `groupinstall`, `groupinfo`, `grouplist`, `groupremove` e `groupupdate`. Le operazioni equivalenti vengono effettuate tramite il comando `group` e il relativo sottocomando. Per esempio:

```bash
sudo dnf group upgrade multimedia
```

è la forma esplicita corrispondente all'operazione che nelle vecchie guide poteva essere indicata come:

```bash
sudo dnf groupupdate @multimedia
```

### Una precisazione sulla sintassi `@`

È importante non trasformare questa osservazione in una regola assoluta del tipo "`@gruppo` non funziona più". DNF 5 supporta ancora la specifica `@<group-spec>` in determinati contesti. La documentazione ufficiale, per esempio, indica che i package group possono essere specificati con `@` quando vengono passati a comandi che accettano sia package spec sia group spec. La differenza importante è quindi soprattutto **la nuova interfaccia esplicita per le operazioni sui gruppi**, non semplicemente la presenza o l'assenza del carattere `@`.

---

# 2. Fedora KDE Plasma 44: una nuova esperienza di primo avvio

Il secondo cambiamento importante riguarda specificamente Fedora KDE Plasma. Fedora KDE Plasma 44 introduce una nuova esperienza di installazione e primo avvio basata su due componenti:

* **Plasma Setup**
* **Plasma Login Manager (PLM)**

Fedora descrive questa modifica come parte di una nuova esperienza KDE più integrata, con l'obiettivo di spostare alcune attività che in precedenza venivano gestite durante l'installazione verso il primo avvio del sistema.

---

## 2.1 Plasma Setup

Nelle nuove installazioni di Fedora KDE Plasma 44, il processo di installazione è stato semplificato. Anaconda non deve più necessariamente occuparsi di tutte le fasi di configurazione iniziale: alcune impostazioni vengono completate tramite **Plasma Setup al primo avvio**. Questo significa che, dopo aver terminato l'installazione e avviato per la prima volta il sistema, non bisogna necessariamente aspettarsi la stessa sequenza di schermate che si sarebbe incontrata seguendo una guida scritta per Fedora KDE precedente. Il primo avvio fa quindi parte integrante del processo di configurazione. La nuova procedura consente, tra le altre cose, di completare la configurazione dell'utente e del sistema attraverso un'interfaccia integrata nell'ambiente Plasma.

---

## 2.2 Plasma Login Manager

Fedora KDE Plasma 44 introduce inoltre **Plasma Login Manager (PLM)** come display/login manager predefinito nelle nuove installazioni KDE, sostituendo SDDM. Questo è un cambiamento importante perché il display manager viene utilizzato prima ancora che la sessione KDE venga avviata. In pratica, è il componente responsabile della schermata di login e dell'avvio della sessione grafica. Per questo motivo una vecchia guida che contiene istruzioni come:

```bash
sudo systemctl enable sddm
```

o che presuppone la presenza di file di configurazione specifici di SDDM **non deve essere applicata automaticamente a una nuova installazione Fedora KDE 44**. Prima bisogna capire quale display manager è effettivamente in uso.

Ad esempio:

```bash
systemctl status display-manager
```

può essere utilizzato per verificare quale servizio è associato al display manager.

Il pacchetto Fedora relativo a Plasma Login Manager è `plasma-login-manager`, mentre il componente KDE dispone anche di un modulo KCM per la relativa configurazione.