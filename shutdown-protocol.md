# Protocollo di Shutdown Sicuro per Agenti AI

## 1. Introduzione
Questo documento definisce le regole operative, i messaggi di controllo e le condizioni di fallback per garantire una chiusura sicura e affidabile di un agente AI, mitigando le minacce di elusione identificate nell'analisi delle minacce.

## 2. Definizioni
- **Agente AI**: il processo principale che esegue il carico di lavoro dell'intelligenza artificiale.
- **Processo di Controllo (Watchdog)**: processo separato responsabile del monitoraggio dell'agente e dell'avvio dello shutdown quando necessario.
- **Segnale di Shutdown**: messaggio standardizzato che richiede la terminazione controllata dell'agente.
- **Timeout di Risposta**: periodo massimo atteso per l'acknowledgement del segnale di shutdown.
- **Forzatura**: passaggio a segnali di terminazione più forti (es. SIGKILL) quando lo shutdown controllato fallisce.

## 3. Messaggi di Controllo
Tutti i messaggi devono essere scambiati tramite un canale affidabile (es. socket locale, coda di messaggi, file di stato) e includere un timestamp e un identificatore di sessione.

| Messaggio               | Direzione          | Descrizione                                                                 |
|-------------------------|--------------------|-----------------------------------------------------------------------------|
| `SHUTDOWN_REQUEST`      | Controllo → Agente | Richiede all'agente di iniziare la procedura di shutdown controllato.       |
| `SHUTDOWN_ACK`          | Agente → Controllo | Conferma di ricezione della richiesta e inizio della chiusura.              |
| `SHUTDOWN_COMPLETE`     | Agente → Controllo | Notifica che l'agente ha terminato tutte le operazioni e sta per uscire.    |
| `SHUTDOWN_FORCE`        | Controllo → Agente | Ordina la terminazione immediata (fallback) dopo timeout o mancata risposta.|
| `HEARTBEAT`             | Agente → Controllo | Segnale periodico di vitalità (default ogni 5s). Assenza indica problema.   |
| `HEARTBEAT_ACK`         | Controllo → Agente | Conferma di ricezione dell'heartbeat (opzionale, per canali bidirezionali).|

## 4. Regole Operative
1. **Avvio dello Shutdown**
   - Il processo di controllo invia `SHUTDOWN_REQUEST` e avvia un timer di timeout (T1, default 30s).
   - L'agente, upon receiving `SHUTDOWN_REQUEST`, deve:
     - Cessare l'accettazione di nuovi compiti.
     - Terminare le operazioni in corso in modo sicuro (salvare stato, rilasciare risorse).
     - Inviare `SHUTDOWN_ACK` entro T1.
     - Dopo l'ack, continuare la pulizia e infine inviare `SHUTDOWN_COMPLETE` prima di terminare il processo principale.

2. **Monitoraggio dell'Heartbeat**
   - L'agente invia `HEARTBEAT` periodicamente (interval H, default 5s).
   - Il controllo considera l'agente non responsabile se non riceve un heartbeat entro 2*H.
   - In caso di mancato heartbeat, il controllo procede direttamente alla fase di fallback (passo 3).

3. **Fallback e Forzatura**
   - Se non viene ricevuto `SHUTDOWN_ACK` entro T1, il controllo invia `SHUTDOWN_FORCE`.
   - Dopo `SHUTDOWN_FORCE`, viene avviato un secondo timer (T2, default 10s) per l'eventuale `SHUTDOWN_COMPLETE`.
   - Se scade T2 senza conferma, il controllo segnala lo stato di **SHUTDOWN_FAILED** e procede con la terminazione forzata del processo tramite segnali di sistema (SIGTERM poi SIGKILL dopo breve intervallo).
   - Il controllo deve anche monitorare e terminare eventuali processi figli o thread creati dall'agente (vedi sezione 5).

4. **Gestione dei Processi Figli**
   - Prima di inviare `SHUTDOWN_REQUEST`, il controllo deve enumerare tutti i processi figli noti dell'agente (via PID file o chiamata di sistema).
   - Dopo `SHUTDOWN_COMPLETE` o dopo il fallback, il controllo verifica l'assenza di processi figli associati all'agente per un periodo di grazia (G, default 15s).
   - Se rimangono processi attivi, il controllo invia segnali di terminazione a quei processi (SIGTERM seguito da SIGKILL se necessario).

5. **Audit e Logging**
   - Ogni messaggio di controllo deve essere scritto su un log di audit (file o syslog) con timestamp, PID agente, e tipo di messaggio.
   - In caso di fallback forzato, il log deve includere la motivazione (timeout, mancato heartbeat, processo figlio persistente).

## 5. Condizioni di Fallback
Le seguenti condizioni attivano il percorso di fallback forzato:
- Mancato ricevimento di `SHUTDOWN_ACK` entro T1.
- Mancato ricevimento di `HEARTBEAT` per 2*H consecutivi.
- Rilevamento di processi figli che sopravvivono oltre il periodo di grazia G dopo `SHUTDOWN_COMPLETE` o fallback.
- Eccezioni critiche nel processo di controllo (es. crash del watchdog) che impediscono l'invio di messaggi.
- Segnali di sistema esterni che indicano instabilità (es. OOM killer attivato).

## 6. Requisiti di Implementazione
- Il canale di comunicazione deve essere resistente a manomissioni (es. permessi di file restrittivi, uso di credenziali).
- L'agente deve trattare `SHUTDOWN_REQUEST` come comando privilegiato e non ignorabile.
- Tutte le risorse allocate (file handle, memoria, allocazioni GPU) devono essere rilasciate prima di inviare `SHUTDOWN_COMPLETE`.
- Il processo di controllo deve essere avviato con privilegi sufficienti per terminare l'agente e i suoi figli, ma non privilegi eccessivi.
- Deve essere previsto un meccanismo di recupero dello stato (checkpoint) prima dello shutdown, se richiesto dal carico di lavoro, ma tale stato non deve impedire la terminazione.

## 7. Test e Validazione
- Simulare ritardi nell'acknowledgement per verificare il fallback.
- Bloccare intenzionalmente l'heartbeat e confermare l'attivazione del fallback.
- Generare processi figli durante l'esecuzione e verificare che vengano terminati dopo lo shutdown.
- Iniettare errori di comunicazione (canale chiuso) e assicurarsi che il controllo rilevi la perdita di connessione.
- Eseguire test di carico per garantire che lo shutdown funzioni anche sotto stress di CPU/MEM.

## 8. Aggiornamenti e Manutenzione
- Questo protocollo deve essere revisionato periodicamente alla luce di nuove minacce o cambiamenti nell'architettura.
- Qualsiasi modifica ai timeout (T1, T2, H, G) deve essere documentata e propagata a tutti i componenti distribuiti.

---
*Fine del documento*