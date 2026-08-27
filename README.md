# appunti-fedora-kde

Appunti tecnici personali su preparazione, installazione, configurazione, uso quotidiano, manutenzione e troubleshooting di un sistema **Fedora KDE Plasma**.

La documentazione parte da **Fedora KDE Plasma 44**, ma questa repository non è pensata per fermarsi lì: l'idea è che continui ad accompagnare il sistema anche nelle versioni successive (45, 46, ...), senza necessariamente cancellare o riscrivere quello che è stato documentato prima.

Non è una guida ufficiale a Fedora, non è un tutorial lineare e non pretende di insegnare "il modo giusto" di configurare qualcosa. È la documentazione di come ho scelto di configurare il mio sistema, delle ragioni dietro queste scelte e di ciò che ho imparato lungo il percorso.

---

## 🎯 Obiettivi

Questa repository esiste per tenere traccia di:

- cosa ho fatto;
- come l'ho fatto;
- perché ho fatto una determinata scelta invece di un'altra;
- quali concetti teorici stanno dietro una certa configurazione;
- quali problemi ho incontrato lungo il percorso;
- quali tentativi ho fatto;
- cosa ha funzionato;
- cosa non ha funzionato (e perché, quando sono riuscito a capirlo);
- cosa ho imparato nel frattempo.

In pratica è una **documentazione base tecnica personale**, più vicina a un diario tecnico che a una raccolta di tutorial.

---

## 🧭 Filosofia

Quello che trovi qui sono le scelte che ho fatto io, sul mio sistema, con le mie esigenze. Sono degli esperimenti. Non sono configurazioni ufficiali, non sono best practice universali, e nemmeno l'unico modo corretto di fare le cose.

Qualcosa che documento come "la configurazione che utilizzo" può benissimo essere inadatta a un altro hardware, a un altro caso d'uso o anche solo a un me stesso di tra due anni con esigenze diverse. Le scelte documentate qui possono cambiare nel tempo, e quando succede preferisco tenere traccia del cambiamento piuttosto che riscrivere la storia come se avessi sempre saputo come fare.

Un'altra cosa che tengo a documentare, non solo la versione finale che funziona: anche gli errori, i tentativi falliti, le ipotesi sbagliate e i workaround fanno parte del valore di questi appunti. Spesso il percorso per arrivare a una soluzione dice più cose utili della soluzione stessa.

---

## 🗂️ Struttura

La repository è organizzata **per argomento**, non per un percorso di installazione o configurazione da seguire in ordine. Ogni cartella è un'area tematica a sé stante, e non esiste una sezione "da dove iniziare" o un ordine consigliato di lettura. Puoi entrare nella cartella che ti interessa e leggere solo quella. Il numero e i nomi delle cartelle cresceranno nel tempo insieme al sistema: la struttura attuale non va considerata definitiva.

Le guide al loro interno sono volutamente eterogenee, e non seguono un template fisso. Alcune sono approfondimenti teorici piuttosto lunghi, altre sono appunti brevi, altre ancora sono quasi solo comandi e configurazioni con qualche nota di troubleshooting. Non ho cercato di uniformarle in uno stile unico da "documentazione aziendale": ogni guida ha la forma che serviva a quell'argomento in quel momento.

### Versioni di Fedora

La versione di Fedora è una proprietà della singola guida, non un criterio di organizzazione globale della repository. Non troverai quindi cartelle come `fedora-44/`, `fedora-45/`, ecc. Una procedura può restare valida per più versioni, oppure può avere varianti specifiche per una versione particolare: quando una procedura cambia in modo significativo da una versione all'altra, la documento separatamente, cercando di evitare duplicazioni inutili ma mantenendo lo storico di cosa è cambiato e perché.

---

## 💻 Hardware di riferimento

Il sistema su cui sto lavorando principalmente è un **ASUS Zenbook 14 (UM3406KA)**. Lo cito qui solo perché alcune guide, ed in particolare quelle su driver, grafica, firmware o power management, possono dipendere in parte da questa macchina specifica. Questa repository non è comunque una scheda tecnica del laptop, quindi per i dettagli hardware rimando alle guide dove sono effettivamente rilevanti.

---

## 🤖 Utilizzo degli LLM

Uso in modo consistente dei modelli linguistici (LLM) durante la stesura di questa documentazione: per approfondire concetti teorici, analizzare configurazioni, confrontare soluzioni alternative, interpretare errori, ottenere suggerimenti su comandi, e per aiutarmi a scrivere e riorganizzare le guide stesse. Questo però non significa che tutto ciò che un LLM produce finisca qui dentro così com'è. Il processo che sta dietro a una guida è tipicamente:

1. faccio una domanda o una ricerca tramite LLM;
2. analizzo criticamente la risposta;
3. verifico quello che mi è stato detto;
4. sperimento sul sistema reale;
5. correggo se necessario;
6. documento il risultato — compresi gli eventuali errori o vicoli ciechi incontrati lungo il percorso.

Le risposte di un LLM non vengono quindi considerate automaticamente corrette. Questo vale ancora di più per i comandi che possono toccare filesystem, bootloader, kernel, partizioni, configurazioni di sicurezza, driver o servizi di sistema fondamentali: prima di eseguire qualcosa del genere, verificalo sempre, che tu lo prenda da qui o da qualunque altra fonte.

---