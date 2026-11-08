# PyIDE — IDE Python tascabile per Android (Java, zero dipendenze esterne)

Codice di partenza per un IDE Python sviluppabile **direttamente da telefono con Code Assist**.
Solo framework Android: niente AppCompat, niente Gradle plugin, niente librerie Maven.

> **APK:** non posso compilarlo qui (nessun Android SDK, rete disabilitata). Nel progetto ci sono i file Gradle e un
> workflow GitHub Actions (`.github/workflows/build-apk.yml`) che lo compila nel cloud: vedi §3.
>
> **Stato onesto del codice.** `ErrorAnalyzer` e `WheelValidator` (Java puro) sono stati compilati e
> testati con tracebacks e wheel di prova, e le regex dell'evidenziatore sono state verificate.
> Il resto (UI, servizi, processi) l'ho scritto con attenzione ma **non l'ho compilato né eseguito su
> un dispositivo**: al primo build aspettati qualche piccola correzione.

---

## 1. Due cose da sapere PRIMA (cambiano l'architettura)

**Chaquopy non è adatto a questo progetto.** Dalla documentazione e dai riferimenti che ho controllato:

- Chaquopy si integra solo tramite il **plugin Gradle**; Code Assist, invece, compila "senza ospitare
  Gradle" (usa un proprio motore + aapt2/D8/apksigner). Il plugin quindi non gira lì.
- I pacchetti pip di Chaquopy sono incorporati **a build-time** (blocco `pip { install ... }`).
  L'installazione a runtime digitando `torch` o `pandas` nell'app **non è supportata** (niente `ensurepip`).

**Scelta adottata:** CPython come **eseguibile nativo lanciato come processo** (`ProcessBuilder`),
con `pip` vero (`python -m pip`). Vantaggi: l'IDE non si blocca mai, un crash nativo non abbatte l'app,
il tasto Stop è un `destroy()`, il terminale e pip usano lo stesso interprete.

**Termux come motore principale.** Dato che il runtime incorporato richiede una preparazione a parte (§5), l'app
funziona subito con **Termux**: PyIDE manda a Termux il comando (intent `RUN_COMMAND`) e legge l'output in tempo
reale da file nella cartella condivisa `/sdcard/PyIDE`. Così hai `pkg`, `pip`, wheel e compilatori di Termux.
Se il runtime incorporato non è presente, il backend predefinito è Termux.

**PyTorch e TensorFlow:** per quanto ho potuto verificare **non esistono wheel Android ufficiali su PyPI**,
quindi `pip install torch` fallirà con "No matching distribution" (l'app lo spiega con un messaggio chiaro).
Chaquopy ha pubblicato alcune wheel pesanti (es. TensorFlow) ma legate alle sue versioni di Python/ABI:
verifica sul loro sito prima di contarci. NumPy/Pandas/SciPy sono invece realistici **se trovi wheel
con tag `android_*_arm64_v8a` e `cp313`** (vedi §5). Alternative per il machine learning su telefono:
ONNX Runtime / TFLite / ExecuTorch come librerie Java, oppure esecuzione remota.

**Con Termux le possibilità aumentano, ma nessuna garanzia:** `pkg install python-numpy` funziona di solito;
OpenCV è disponibile come pacchetto Termux (cerca con `pkg search opencv`). `python-torch` esiste nei repository
Termux ma ci sono segnalazioni di errori durante l'installazione; per TensorFlow controlla con
`pkg search tensorflow` dopo aver abilitato i repository extra (`tur-repo`, `x11-repo`; pulsante "Repo extra" nel
Gestore PIP). Termux ha anche un indice pip con pacchetti precompilati:
`https://termux-user-repository.github.io/pypi/` (aggiunto automaticamente dal pulsante "pip in Termux").
Conta che questi pacchetti sono grandi (torch: gigabyte) e spesso in ritardo rispetto a PyPI.

---

## 2. Architettura

```
app/src/main/
├─ AndroidManifest.xml        extractNativeLibs=true, foreground service
├─ res/layout/activity_main.xml   toolbar, find bar, editor, tasti rapidi, log, FAB Play, sidebar
├─ res/drawable/fab_bg.xml
└─ java/dev/pyide/
   ├─ MainActivity.java       collega tutto; mai I/O sul thread UI
   ├─ CodeEditText.java       editor: undo/redo, vai a linea, cerca/sostituisci, numeri di riga,
   │                          barre rosse/arancio sulle righe di errore, auto-indentazione
   ├─ SyntaxHighlighter.java  Python, JSON, HTML/XML, CSS, JS (regex, con debounce)
   ├─ AutoSaver.java          salvataggio asincrono, coalescente, atomico (tmp + rename)
   ├─ ProjectManager.java     progetti multi-file + entry point (.pyide.json)
   ├─ FileOps.java            crea / rinomina / sposta / elimina, vincolati alla cartella progetto
   ├─ FileTreeAdapter.java    albero file della sidebar
   ├─ PythonEnv.java          posizione runtime, ambiente, estrazione stdlib al primo avvio
   ├─ ScriptRunner.java       interfaccia dei backend di esecuzione
   ├─ PythonRunner.java       backend incorporato: processo separato + stdin + Stop (SIGTERM → SIGKILL)
   ├─ TermuxRunner.java       backend Termux: usa venv ~/.pyide-venv se presente
   ├─ TermuxBridge.java       intent RUN_COMMAND, controlli installazione/permesso, comandi di setup
   ├─ TermuxJob.java          comando in Termux + log in tempo reale (file out/err/pid/exit) + Stop
   ├─ ErrorAnalyzer.java      traceback → tipo, riga, cause collegate   (Java puro, testato)
   ├─ PipManager.java         pip automatico + import manuale .whl + rimozione
   ├─ WheelValidator.java     validazione .whl (nome, ZIP/CRC, dist-info, tag)  (Java puro, testato)
   ├─ TerminalSession.java    shell con funzioni python / pip
   ├─ ToolDialogs.java        dialoghi Terminale, Gestore PIP, Impostazioni
   ├─ RunnerService.java      foreground service: tiene vivo il processo in background
   ├─ Prefs.java / Ui.java
```

| Requisito | Dove |
|---|---|
| Undo/Redo, Vai alla linea, Cerca/Sostituisci | `CodeEditText` + toolbar/find bar in `MainActivity` |
| Autosalvataggio (on/off) + salvataggio manuale | `AutoSaver`, `Prefs.autosave()`, pulsante 💾 |
| File manager (crea, rinomina, sposta, elimina) | `FileTreeAdapter`, `FileOps`, menu a tocco lungo |
| Progetti multi-file + entry point | `ProjectManager`; Play avvia **sempre** `getEntry()` |
| Terminale | `TerminalSession` + `ToolDialogs.terminal` |
| PIP automatico / manuale con validazione | `PipManager`, `WheelValidator`, `ToolDialogs.PipDialog` |
| Log + errori + cause collegate | `ErrorAnalyzer`, `MainActivity.analyzeError`, marcature in `CodeEditText` |
| Background, no ANR, Stop per bot | `PythonRunner`, `RunnerService`, executor in ogni gestore |

### Come funziona l'analisi errori
- **SyntaxError / IndentationError / TabError** → "Errore di sintassi" (non ha traceback: si usa il blocco `File "...", line N`).
- Altri → "Errore di runtime". Riga rossa = ultimo frame nel **tuo** progetto (non dentro NumPy).
- Arancio = catena di chiamate nel progetto + eccezioni concatenate (`direct cause` / `During handling`).
- Giallo = **causa probabile**: ultima assegnazione/definizione delle variabili usate nella riga che fallisce.
  È un'euristica (non un debugger): va letta come suggerimento.
- Le righe nel log sono cliccabili e aprono il file alla riga giusta.

---

## 3. Come ottenere l'APK

**Opzione A — GitHub Actions (consigliata, tutto da telefono):**
1. Crea un repository su GitHub e carica il contenuto di questo progetto (dal sito GitHub in modalità desktop).
2. Scheda *Actions* → *Build APK* → *Run workflow* (parte anche a ogni push).
3. A fine build scarica l'artefatto `PyIDE-debug-apk`, scompattalo e installa l'APK (debug, firmato con chiave di debug).
4. Se la build fallisce, copiami il log dell'errore: il codice Android non è stato compilato prima d'ora.

**Opzione B — Code Assist:** nuovo progetto *Java / Empty Activity*, package `dev.pyide`, minSdk 26; copia
`AndroidManifest.xml`, `res/` e i `.java`. Nessuna dipendenza da aggiungere (il file `app/build.gradle` è solo di riferimento).

---

## 4. Configurare Termux (una volta)

1. Installa **Termux da F-Droid o GitHub** (la versione del Play Store è obsoleta).
2. In Termux incolla (la stessa procedura appare in PyIDE, pulsante *Copia comandi*):
```
mkdir -p ~/.termux && (grep -q '^allow-external-apps' ~/.termux/termux.properties 2>/dev/null || echo 'allow-external-apps=true' >> ~/.termux/termux.properties)
termux-setup-storage
pkg update -y && pkg install -y python python-pip
```
   poi chiudi e riapri Termux.
3. In PyIDE → Impostazioni → *Termux*; concedi **"Esegui comandi in Termux"** e **"Accesso a tutti i file"**
   (i progetti vanno in `/sdcard/PyIDE/projects`, l'unica cartella leggibile da entrambe le app).
4. Disattiva l'ottimizzazione batteria per Termux se vuoi bot di lunga durata.
5. Gestore PIP → *pkg in Termux* (`numpy`, `opencv-python`, …) o *pip in Termux* (usa il venv `~/.pyide-venv`
   con `--system-site-packages`, quindi vede i pacchetti installati con `pkg`).

Limiti con Termux: nessun `input()` interattivo (stdin non disponibile); l'output arriva con ~0,2 s di ritardo; i job
continuano a girare anche se chiudi PyIDE (Stop li termina).

---

## 5. Runtime incorporato (facoltativo)

Servono questi file nell'APK:

```
src/main/jniLibs/arm64-v8a/libpython_exec.so   ← l'eseguibile `python3.13` rinominato
src/main/jniLibs/arm64-v8a/libpython3.13.so    ← + tutte le .so da cui dipende (libssl, libsqlite3, libz...)
src/main/assets/python-stdlib.zip              ← contiene lib/python3.13/ (stdlib, lib-dynload, ensurepip con la wheel di pip)
```

- Il file va in `jniLibs` perché da Android 10 non si può eseguire un binario copiato in `filesDir`;
  i file in `nativeLibraryDir` sì (per questo `extractNativeLibs="true"`).
- Come ottenerlo: CPython ha supporto ufficiale ad Android dalla 3.13 (script `Android/android.py` nel
  repository, da lanciare su Linux/macOS o in GitHub Actions). **Controlla le licenze** e la coerenza dei percorsi
  (`PYTHONHOME` = `filesDir/python`). Un telefono da solo non basta per compilare CPython.
- Se usi un'altra versione cambia `PythonEnv.PY_TAG` (es. `"312"`): la validazione delle wheel lo usa.
- Test: apri il Terminale integrato e lancia `python -c "import sys; print(sys.version)"` e `pip --version`.

---

## 6. Linee guida per librerie pesanti senza bloccare l'app

0. **Per i pacchetti davvero pesanti usa Termux** (§4): l'installazione gira fuori da PyIDE e il log arriva in tempo reale.
1. **Mai in-process.** pip, script e terminale sono processi figli; la UI riceve solo testo via callback sul main thread.
2. **Foreground service** (`RunnerService`) durante installazioni e script: senza, Android sospende il processo a schermo spento.
   Android 13+ chiede il permesso notifiche (già richiesto all'avvio).
3. **Spazio**: `PipManager` rifiuta l'installazione sotto i 300 MB liberi. Pacchetti come NumPy+Pandas+SciPy occupano centinaia di MB.
4. **Solo wheel**: l'installazione usa `--prefer-binary`; sul telefono non c'è un compilatore C/Rust, quindi i pacchetti
   che si compilano da sorgente falliscono (l'app lo segnala).
5. **Tag giusti**: servono `cp313` (o `abi3`/`py3`) e `android_*_<abi>`. Le wheel `manylinux`/`win`/`macosx` sono rifiutate con spiegazione.
   Per wheel Android da un repository dedicato imposta l'**indice pip aggiuntivo** in Impostazioni (solo `https://`).
6. **Memoria**: il processo Python ha il suo budget; con modelli grandi Android può terminarlo (low-memory killer).
   Salva spesso e tieni ridotto il log (limite predefinito 150.000 caratteri).
7. **Rete**: scarica su Wi-Fi; `--progress-bar off` evita log enormi.

---

## 7. Funzionalità extra incluse
Tasti rapidi (Tab, parentesi, ecc.) sopra la tastiera · auto-indentazione Python · input per `input()` dal pannello log ·
copia log · limite di memoria del log · salvataggio atomico · rimozione pacchetti · schermo sempre acceso opzionale ·
dimensione carattere · ripristino ultimo progetto · protezione da path traversal nelle operazioni sui file.

## 8. Limiti noti
- Il terminale usa pipe, non un PTY: niente vim/htop e niente Ctrl-C reale (c'è "Interrompi").
- Lo Stop termina il processo principale; eventuali sotto-processi lanciati dallo script potrebbero restare.
- Chiudere l'Activity ferma gli script. Per bot 24/7 sposta `PythonRunner` dentro `RunnerService`.
- Evidenziazione a regex (non un parser): file oltre 250.000 caratteri restano senza colori.
- Non ci sono autocompletamento, debugger, Git.
- Il workflow CI e il codice Termux sono **non testati** su dispositivo; l'intent RUN_COMMAND è quello documentato da Termux.
- Con `targetSdk` alti Android può restringere ulteriormente l'esecuzione di binari: testa su più versioni.
