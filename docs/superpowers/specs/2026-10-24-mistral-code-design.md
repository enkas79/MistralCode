# Mistral Code: design

Data: 2026-10-24. Dimensione del lavoro: progetto (più fasi). Mockup approvato: variante A (https://claude.ai/artifact/QXdGeE7cXWM7vkTYLba2yb).

## Intento e perché
Un'app desktop con l'esperienza d'uso di Claude Code (conversazione a flusso, chiamate a strumenti visibili, diff, richiesta di permesso) ma con un agente basato sui modelli Mistral. Serve all'autore, per lavorare con Mistral con la stessa ergonomia di Claude Code. Funzione chiave: estendere l'agente con "potenziamenti" presi da repository GitHub, compreso codice eseguibile.

## Obiettivi
- Finestra PySide6, una colonna, stile terminale (variante A), temi chiaro/scuro.
- Chat in streaming con l'API Mistral (SDK `mistralai`), senza mai bloccare la GUI.
- Blocchi per le chiamate a strumenti, diff colorati, richiesta di permesso (Consenti / Sempre / Rifiuta).
- Potenziamenti da GitHub (`owner/repo`, opzionale sottocartella) con installazione solo dopo conferma.
- Menu obbligatori dal CLAUDE.md: Aiuto/Info (Informazioni, Controlla Aggiornamenti, Guida), `version.txt`, autoupdate da GitHub Releases.

## Non obiettivi
- Pannelli laterali (sessioni, piano, file): variante B, rimandata.
- Un marketplace o una ricerca di plugin: si aggiunge solo per URL/`owner/repo`.
- Sandbox a livello di sistema operativo per i plugin (vedi rischi).

## Anti-obiettivi (sarebbe un fallimento anche se funziona)
- Un plugin scaricato che gira senza che l'utente l'abbia confermato.
- Un plugin che blocca o fa crashare la GUI.
- Qualsiasi I/O, rete o chiamata API nel thread principale.

## Vincoli
Python 3.9+, type hints, OOP, solo PySide6, QThread/QRunnable con segnali, QSS centralizzato (multipli di 4/8 px, WCAG AA), errori di rete silenziosi in background e `QMessageBox` se l'azione è manuale. Test con `pytest` (priorità a logica e modelli), `ruff`. Il resto è nel CLAUDE.md di progetto.

## Approcci considerati (plugin con codice)
1. **Sottoprocesso con manifest, stile MCP (scelto).** Isolamento dalla GUI e ogni chiamata passa dai permessi. Costo: un piccolo client del protocollo.
2. Import in-process: più semplice, ma un repo scaricato ha i permessi dell'app e può far crashare la GUI. Scartato.
3. Solo testo/comandi, codice solo via server MCP già installati: più sicuro ma non copre il caso d'uso. Scartato.

## Architettura (decisa in conversazione)
- `src/gui/`: finestra principale, widget del flusso chat, dialoghi, `styles.qss`. Solo layout e segnali.
- `src/core/`: modelli (messaggio, chiamata a strumento, sessione), loop agente, registro permessi. Logica pura, testabile.
- `src/mistral/`: client sull'SDK `mistralai`, streaming e function calling, eseguito in `QThread` con segnali.
- `src/plugins/`: gestore potenziamenti: clone/aggiornamento da GitHub, lettura del manifest, avvio e comunicazione con il sottoprocesso.
- `src/updater/`: controllo release, download, sostituzione e riavvio.
- Strumenti integrati dell'agente: lettura file, modifica file con diff, ricerca, comando shell. Tutti dietro i permessi.

### Potenziamenti
- Input: `owner/repo[/sottocartella]`, risolto a un commit fissato (pin). L'aggiornamento è un'azione esplicita con nuova conferma.
- Manifest nella radice del repo (formato esatto da definire nel piano): nome, versione, comandi `/`, strumenti, file di istruzioni, comando di avvio.
- Installazione: finestra con repo, commit e comando che verrà eseguito; il clone avviene solo dopo "Installa". Cartella dedicata sotto la directory dati dell'utente.
- Esecuzione: sottoprocesso con comunicazione su stdio; ogni chiamata a uno strumento del plugin richiede lo stesso Consenti/Sempre/Rifiuta degli strumenti integrati.
- I file di istruzioni dei repo sono trattati come input non fidato e mostrati in anteprima prima dell'attivazione.
- Menu "Potenziamenti": aggiungi, aggiorna, attiva/disattiva, rimuovi.

## Decisioni mie (da verificare)
- Chiave API Mistral in una variabile d'ambiente o nel portachiavi di sistema, mai nel codice o nel repo.
- Modello predefinito configurabile; l'elenco modelli viene dall'API.
- Il protocollo stdio segue MCP, così i server MCP esistenti funzionano come plugin.
- Cartella dei plugin e configurazione nella directory dati dell'utente, non nel repo.
- Prima fase: finestra, chat in streaming, strumenti integrati, permessi. Plugin e autoupdate nelle fasi successive.

## Rischi noti
Un sottoprocesso non è una sandbox: un plugin approvato può comunque fare ciò che l'utente stesso può fare. La protezione è la conferma all'installazione, il pin del commit e i permessi per chiamata. Una sandbox vera è un lavoro successivo.

## Lasciato al realizzatore
Formato preciso del manifest, schema dei permessi persistenti ("Sempre"), strategia di troncamento del contesto, struttura dei test, tema QSS.
