# Sicurezza agenti AI con shutdown

## Descrizione
Progetto per analizzare e mitigare le minacce di elusione dello shutdown da parte di agenti AI. Include analisi delle minacce, specifica di un protocollo di shutdown sicuro, implementazione di un modulo di shutdown in Python e test unitari.

## Architettura
- **Analisi delle minacce**: documento che identifica vettori di attacco e scenari di rischio.
- **Protocollo di shutdown**: definisce messaggi di controllo (SHUTDOWN_REQUEST, SHUTDOWN_ACK, SHUTDOWN_COMPLETE, SHUTDOWN_FORCE, HEARTBEAT) e regole operative.
- **Modulo di shutdown**: implementazione Python che gestisce la comunicazione con il processo di controllo, gestisce heartbeat, timeout e forzatura.
- **Test unitari**: verificano i percorsi di chiusura normale, timeout e forzatura.

## Installazione
1. Clonare il repository.
2. Assicurarsi di avere Python 3.8+ installato.
3. Installare le dipendenze (se presenti) con `pip install -r requirements.txt`.
4. Il modulo `shutdown_handler.py` può essere importato in altri progetti.

## Uso
- Importare `shutdown_handler` nel proprio agente AI.
- Configurare i parametri di timeout e intervallo heartbeat.
- Avviare il processo di controllo che utilizza il modulo per monitorare l'agente.
- Vedere gli esempi nella directory `examples/` (se presente).

## Esempi
Vedere la directory `examples/` per esempi di integrazione con un agente AI semplice.

## Stato
✅ COMPLETATO — 2026-06-11
- Analisi delle minacce completata
- Specifica del protocollo di shutdown completata
- Implementazione modulo di shutdown completata
- Test unitari per il modulo di shutdown completati
- Documentazione nel vault di sistema completata