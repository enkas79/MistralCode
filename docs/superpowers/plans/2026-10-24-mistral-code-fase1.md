# Mistral Code, Fase 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Finestra PySide6 in stile terminale (variante A) con chat in streaming sull'API Mistral, strumenti integrati e permessi per chiamata.

**Architecture:** Logica pura in `src/core` (modelli, strumenti, permessi, loop agente), adattatore SDK in `src/mistral`, GUI in `src/gui`. Il loop agente gira in un `QThread` (`AgentWorker`) e parla con la GUI solo via segnali; la richiesta di permesso blocca il worker, non la GUI.

**Tech Stack:** Python 3.9+, PySide6, `mistralai`, pytest + pytest-qt, ruff.

**Spec:** `docs/superpowers/specs/2026-10-24-mistral-code-design.md`. Fuori da questo piano: plugin da GitHub, autoupdate, variante B.

## Global Constraints
- Python 3.9+: type hints su tutto, usare `Optional`/`List`/`Dict` di `typing` (niente `X | Y`).
- Solo PySide6 per la GUI. Mai I/O, rete o chiamate API nel thread principale.
- QSS centralizzato in `src/gui/styles.qss`, spaziature multipli di 4/8 px, contrasto WCAG AA, tema chiaro e scuro.
- Errori di rete e I/O sempre intercettati; `QMessageBox.warning/critical` per azioni manuali.
- Menu `Aiuto` con `Informazioni` (`QMessageBox.about`, autore e versione da `version.txt`), `Controlla Aggiornamenti`, `Guida`.
- Chiave API solo da `MISTRAL_API_KEY`, mai nel codice o nel repo.
- Avvio `python src/main.py`; test `pytest`; lint `ruff check . --fix`. Test GUI con `QT_QPA_PLATFORM=offscreen`.
- Ogni commit termina con le righe `Co-Authored-By` e `Claude-Session` del contesto di sessione.

## Review Focus
- Chiave API assente: la finestra si apre e mostra un avviso, non crasha.
- Rete caduta o errore API a metà streaming: messaggio d'errore nella chat, il worker termina, l'input torna usabile.
- Argomenti dello strumento non validi (JSON malformato, campo mancante): risultato d'errore restituito al modello, nessuna eccezione.
- Percorso fuori dalla cartella di lavoro (`../../etc/passwd`, link simbolici): rifiutato.
- Modifica file con testo da sostituire assente o ripetuto: errore chiaro, file intatto.
- Comando shell che non termina: interrotto dopo il timeout, output troncato.
- Permesso negato: il modello riceve un risultato "rifiutato dall'utente" e il loop prosegue o si ferma senza bloccarsi.
- Annullamento durante lo streaming: il worker si ferma senza lasciare messaggi incompleti nella sessione.

## File Structure
- `version.txt`, `requirements.txt`, `pyproject.toml`: metadati, dipendenze, config ruff/pytest.
- `src/main.py`: avvio. `src/core/{models,tools,permissions,agent,version}.py`: logica. `src/mistral/client.py`: adattatore SDK.
- `src/gui/{worker,chat_view,main_window}.py`, `src/gui/styles.qss`: interfaccia.
- `tests/` specchia `src/`.

---

### Task 1: Scheletro e modelli

**Files:** Create `version.txt` (`0.1.0`), `requirements.txt`, `pyproject.toml`, `src/core/models.py`, `src/core/version.py`, `tests/core/test_models.py`, `tests/core/test_version.py`.

**Interfaces:**
- Produces: `ToolCall(id: str, name: str, arguments: Dict[str, Any])`; `Message(role: str, content: str = "", tool_calls: List[ToolCall] = [], tool_call_id: Optional[str] = None)` con `to_api() -> Dict[str, Any]`; `Session` con `messages: List[Message]`, `add(msg: Message) -> None`, `to_api() -> List[Dict[str, Any]]`; `read_version(path: Path) -> str` (stringa vuota se il file manca).

- [ ] **Step 1:** Test `test_message_to_api_includes_tool_calls` (un messaggio assistant con una `ToolCall` produce `tool_calls[0]["function"]["name"]` e `arguments` come stringa JSON), `test_tool_message_to_api_has_tool_call_id`, `test_read_version_missing_file_returns_empty`.
- [ ] **Step 2:** `pytest tests/core -v` → FAIL (moduli assenti).
- [ ] **Step 3:** Implementare i dataclass e `to_api` nel formato chat-completions di Mistral. Pin in `requirements.txt`: `PySide6`, `mistralai`, `pytest`, `pytest-qt`, `ruff`. In `pyproject.toml` configurare `pythonpath = ["src"]` per pytest.
- [ ] **Step 4:** `pytest tests/core -v` → PASS.
- [ ] **Step 5:** Commit `feat: scheletro progetto e modelli`.

### Task 2: Permessi e strumenti integrati

**Files:** Create `src/core/permissions.py`, `src/core/tools.py`, `tests/core/test_permissions.py`, `tests/core/test_tools.py`.

**Interfaces:**
- Produces: `Decision` (Enum: `ALLOW`, `ALWAYS`, `DENY`); `PermissionRegistry` con `is_pre_approved(tool: Tool) -> bool` (vero se `tool.read_only` o già `ALWAYS`) e `remember(tool_name: str) -> None`; `ToolError(Exception)`; `ToolResult(output: str, is_error: bool = False, diff: Optional[str] = None)`; `Tool` (ABC con `name`, `description`, `parameters: Dict[str, Any]`, `read_only: bool`, `run(args: Dict[str, Any], root: Path) -> ToolResult`); `resolve_in_root(root: Path, rel: str) -> Path`; `ToolRegistry` con `register(tool)`, `get(name) -> Optional[Tool]`, `schemas() -> List[Dict[str, Any]]`; strumenti `ReadFileTool` (`read_file`, read-only, tronca a 20.000 caratteri), `EditFileTool` (`edit_file`: `path`, `old`, `new`; restituisce `diff` unificato), `GrepTool` (`grep`, read-only), `BashTool` (`bash`: `command`, timeout 30 s, output troncato a 20.000 caratteri); `default_registry() -> ToolRegistry`.

- [ ] **Step 1:** Test: `test_resolve_in_root_rejects_parent_escape`, `test_resolve_in_root_rejects_symlink_outside` (assert `ToolError`); `test_read_file_truncates_large_file`; `test_edit_file_replaces_unique_match_and_returns_diff`; `test_edit_file_missing_old_text_errors_and_leaves_file_intact`; `test_edit_file_ambiguous_match_errors`; `test_bash_times_out` (`sleep 5` con timeout ridotto via parametro del costruttore); `test_tool_invalid_args_returns_error_result` (campo mancante → `ToolResult.is_error`); `test_always_decision_pre_approves_tool`.
- [ ] **Step 2:** `pytest tests/core -v` → FAIL.
- [ ] **Step 3:** Implementare. `resolve_in_root` usa `Path.resolve()` e controlla che il risultato sia sotto `root.resolve()`. `run` non solleva mai: converte `ToolError` e `OSError` in `ToolResult(is_error=True)`. `schemas()` produce il formato `{"type": "function", "function": {...}}`.
- [ ] **Step 4:** `pytest tests/core -v` → PASS.
- [ ] **Step 5:** Commit `feat: permessi e strumenti integrati`.

### Task 3: Client Mistral

**Files:** Create `src/mistral/__init__.py`, `src/mistral/client.py`, `tests/mistral/test_client.py`.

**Interfaces:**
- Consumes: `ToolCall`.
- Produces: `TextDelta(text: str)`, `ToolCallEvent(call: ToolCall)`, `StreamDone()`; `load_api_key() -> Optional[str]` (da `MISTRAL_API_KEY`, `None` se vuota); `MistralClient(api_key: str, model: str = "mistral-large-latest", sdk: Optional[Any] = None)` con `stream(messages: List[Dict[str, Any]], tools: List[Dict[str, Any]]) -> Iterator[Union[TextDelta, ToolCallEvent, StreamDone]]`; `MistralError(Exception)` per ogni errore SDK/rete.

- [ ] **Step 1:** Verificare con Context7 (o `help(Mistral.chat.stream)` se non disponibile) la firma di `client.chat.stream` e la forma dei chunk (`event.data.choices[0].delta.content` / `.tool_calls`). Scrivere i test con un SDK finto: `test_stream_yields_text_deltas_then_done`, `test_stream_yields_tool_call_with_parsed_arguments`, `test_malformed_tool_arguments_become_error_dict` (argomenti non-JSON → `arguments == {"_error": "..."}`, nessuna eccezione), `test_sdk_exception_is_wrapped_in_mistral_error`, `test_load_api_key_empty_returns_none`.
- [ ] **Step 2:** `pytest tests/mistral -v` → FAIL.
- [ ] **Step 3:** Implementare; l'import di `mistralai` avviene solo se `sdk` è `None`, così i test non lo richiedono.
- [ ] **Step 4:** `pytest tests/mistral -v` → PASS.
- [ ] **Step 5:** Commit `feat: client Mistral in streaming`.

### Task 4: Loop agente

**Files:** Create `src/core/agent.py`, `tests/core/test_agent.py`.

**Interfaces:**
- Consumes: `MistralClient.stream`, `ToolRegistry`, `PermissionRegistry`, `Decision`, `Session`.
- Produces: eventi `ToolStarted(call: ToolCall)`, `ToolFinished(call: ToolCall, result: ToolResult)`, `AgentError(message: str)` (più `TextDelta` del client); `AgentLoop(client, tools: ToolRegistry, permissions: PermissionRegistry, root: Path, max_steps: int = 20)` con `run(session: Session, user_text: str, ask: Callable[[ToolCall], Decision], emit: Callable[[Any], None], cancelled: Callable[[], bool]) -> None`.

- [ ] **Step 1:** Test con client finto: `test_plain_answer_appends_assistant_message`, `test_tool_call_runs_tool_and_feeds_result_back` (due giri: tool call, poi risposta testuale), `test_read_only_tool_does_not_ask`, `test_denied_tool_returns_refusal_result_to_model`, `test_always_remembers_tool_for_next_call`, `test_unknown_tool_returns_error_result`, `test_max_steps_stops_loop_with_error`, `test_mistral_error_emits_agent_error_and_leaves_no_partial_message`, `test_cancel_midstream_discards_partial_assistant_message`.
- [ ] **Step 2:** `pytest tests/core/test_agent.py -v` → FAIL.
- [ ] **Step 3:** Implementare. Il messaggio assistant entra nella sessione solo a fine giro riuscito; testo "Rifiutato dall'utente" come contenuto del messaggio `tool` in caso di `DENY`.
- [ ] **Step 4:** `pytest tests/core -v` → PASS.
- [ ] **Step 5:** Commit `feat: loop agente`.

### Task 5: Worker, chat e finestra

**Files:** Create `src/gui/__init__.py`, `src/gui/worker.py`, `src/gui/chat_view.py`, `src/gui/main_window.py`, `src/gui/styles.qss`, `src/main.py`, `tests/gui/test_worker.py`, `tests/gui/test_main_window.py`.

**Interfaces:**
- Consumes: `AgentLoop`, `Decision`, `read_version`, `load_api_key`.
- Produces: `AgentWorker(QThread)` con segnali `text_delta(str)`, `tool_started(object)`, `tool_finished(object, object)`, `permission_requested(object)`, `failed(str)`, `turn_finished()`, e metodi `answer_permission(decision: Decision)`, `cancel()`; `ChatView(QScrollArea)` con `add_user(text)`, `append_assistant(text)`, `add_tool(call)`, `finish_tool(call, result)`, `ask_permission(call)` ed emette `decided(object)`; `MainWindow(root: Path, version_file: Path, worker_factory)` con menu `Aiuto`.

- [ ] **Step 1:** Test: `test_worker_emits_text_and_finishes` (client finto), `test_worker_blocks_until_permission_answered` (la richiesta emette `permission_requested`, `answer_permission(ALLOW)` sblocca), `test_cancel_unblocks_pending_permission`, `test_main_window_without_api_key_shows_warning_and_disables_input`, `test_about_text_contains_version` (`QMessageBox.about` monkeypatchato, versione letta da un `version.txt` temporaneo), `test_aiuto_menu_has_three_entries`, `test_chat_view_renders_diff_with_added_and_removed_lines`.
- [ ] **Step 2:** `pytest tests/gui -v` → FAIL.
- [ ] **Step 3:** Implementare. `AgentWorker` chiama `AgentLoop.run` con `ask` che emette `permission_requested` e attende un `threading.Event`; `cancel()` imposta il flag e sblocca l'attesa con `DENY`. `Controlla Aggiornamenti` in questa fase apre la pagina Releases del repo con `QDesktopServices` (l'autoupdate vero è nel piano 3). `Guida` è un `QDialog` con testo statico in italiano. `styles.qss`: token chiaro/scuro scelti dal tema di sistema (palette del mockup: arancio Mistral, carbone caldo), caricato da `main.py`.
- [ ] **Step 4:** `pytest tests/gui -v` → PASS.
- [ ] **Step 5:** Commit `feat: finestra principale e chat`.

### Task 6: Verifica finale e rilascio versione

**Files:** Modify `version.txt` (→ `0.1.0` già presente; confermare), `README.md`.

- [ ] **Step 1:** `ruff check . --fix` → nessun errore; `pytest` → tutto verde.
- [ ] **Step 2:** `QT_QPA_PLATFORM=offscreen python src/main.py` con `MISTRAL_API_KEY` assente: si apre, mostra l'avviso, nessuna traceback (usare un timeout di chiusura).
- [ ] **Step 3:** `README.md`: come impostare `MISTRAL_API_KEY`, avviare e testare.
- [ ] **Step 4:** Commit `docs: README e verifica fase 1` e push sul branch di lavoro.

---

## Self-Review
- **Copertura spec:** finestra A, streaming, blocchi strumenti/diff/permessi, menu Aiuto, `version.txt`, chiave via ambiente: tutti coperti. Plugin e autoupdate sono dichiarati fuori ambito (piani separati).
- **Coerenza tipi:** `Decision`, `ToolResult`, `ToolCall` definiti nei Task 1-2 e usati senza variazioni nei Task 3-5.
- **Punto aperto:** la forma esatta dei chunk dell'SDK va confermata nel Task 3, Step 1.
