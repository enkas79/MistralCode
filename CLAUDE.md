# CLAUDE.md di progetto (da salvare nella root del repository)

Le regole su lingua, stile, uso degli strumenti e metadati autore sono nel CLAUDE.md globale: non vengono ripetute qui.

## 1. Stack
- **Python 3.9+**, type hints PEP 484 obbligatori, OOP.
- **GUI:** solo **PyQt6** o **PySide6** (vietati Tkinter, CustomTkinter, Flet e altri).
- **CI/CD:** GitHub Actions, build multi-piattaforma con PyInstaller / NSIS.
- **PHP, JavaScript, Java:** solo se richiesti esplicitamente per integrazioni esterne.

## 2. Interfaccia obbligatoria
Ogni finestra principale ha una `QMenuBar` con menu **Aiuto / Info**:
- **Informazioni:** `QMessageBox.about` con autore e versione letta dinamicamente da `version.txt`.
- **Controlla Aggiornamenti:** verifica e download manuale delle nuove versioni.
- **Guida:** finestra o dialogo con la guida all'uso.

Altre voci di menu vengono indicate di volta in volta.

## 3. Versioni, release e autoupdate
- `version.txt` nella root contiene la versione corrente (es. `1.0.0`).
- Ogni modifica funzionale o strutturale incrementa `version.txt` (patch o minor): questo fa partire `.github/workflows/build-installers.yml`.
- **Autoupdate:** all'avvio, verifica in background (API GitHub Releases) confrontando versione remota e locale; se c'è una release nuova, dialogo con changelog e conferma; download e sostituzione/riavvio senza bloccare la GUI.

## 4. Architettura Qt
- **Mai** I/O, rete, query pesanti o controllo aggiornamenti nel main thread: usa `QThread` o `QThreadPool`/`QRunnable` con segnali (`pyqtSignal` / `Signal`).
- Separa GUI (layout, widget, segnali), logica di business e modulo `updater`.
- **QSS:** fogli di stile centralizzati (`styles.qss` o modulo dedicato), spaziatura a multipli di 4/8 px, palette ad alto contrasto (WCAG AA) per tema chiaro e scuro, nessun widget di sistema non stilizzato.
- **Errori:** intercetta eccezioni di rete e I/O; silenziose in background, `QMessageBox.warning`/`critical` se l'azione è manuale. Nessun crash non gestito.

## 5. Comandi
- Avvio: `python src/main.py`
- Test: `pytest` (priorità a logica interna e modelli)
- Lint/format: `ruff check . --fix` (alternative: `black .`, `flake8 .`)
- Dipendenze: `pip freeze > requirements.txt`

## 6. Git
- Lavora su branch. A modifiche completate **e** con `pytest` e `ruff` verdi, esegui push e merge su `main` senza chiedere conferma.
- Se test o lint falliscono, non fare merge: correggi prima.

## 7. Plugin, skill e MCP
**Attivi:**
- **claude-mem:** salva bug risolti e decisioni architetturali (`mem-search`, `learn-codebase`).
- **superpowers:** `test-driven-development` per core e logica di business; `systematic-debugging` per bug e crash Qt; prima di modifiche strutturali, piano con file target, segnali/slot e thread worker.
- **impeccable / frontend-design:** solo per QSS e layout Qt. Vietati runtime o dipendenze web.
- **21st-ui:** solo come riferimento visivo (card, sidebar, tabelle); traduci i pattern in widget Qt e classi QSS.
- **Context7:** verifica firme di metodi, enum e differenze PyQt6/PySide6 prima di ipotizzare API deprecate o inesistenti.

**Da ignorare:** plugin finanza/contabilità (`variance-analysis`, `journal-entry`, `sox-testing`, ecc.), editing Office (`docx`, `xlsx`, `pptx`) e suite **ponytail** (non sacrificare modularità OOP e robustezza per il minimalismo).

**find-skills:** solo su richiesta esplicita. Prima di proporre un'installazione verifica install count (>1K) e fonte; mai `-y` o `-g` senza conferma.
