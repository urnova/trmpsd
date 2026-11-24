# SPÉCIFICATIONS ET RÈGLES FIREBASE FIRESTORE — PROJET E.V.A

---

## 1. STRUCTURE DES COLLECTIONS CLOUDWORKS

Toutes les données CloudWorks sont partitionnées par utilisateur sous le chemin racine :
`/cloudworks/{userId}/`

### A. Collection `devices` : `/cloudworks/{userId}/devices/{deviceId}`
Enregistre chaque appareil connecté au compte :
```json
{
  "id": "MSG-10-31-DF-00",
  "hostname": "DESKTOP-2A9PUI2",
  "platform": "win32",
  "os": "Windows 11",
  "online": true,
  "lastSeen": "2026-11-24T12:35:00.000Z",
  "llmInstalled": true,
  "llmRunning": true,
  "version": "5.6.22"
}
```

### B. Collection `commands` : `/cloudworks/{userId}/commands/{commandId}`
Enregistre chaque ordre ou tâche agentique :
```json
{
  "id": "task_1732450000000",
  "type": "agentic_task",
  "status": "pending | running | executing | done | error | cancelled",
  "step": "Exécution : app_launch...",
  "steps": [
    { "text": "Démarrage du LLM local...", "ts": "2026-11-24T12:35:01.000Z" },
    { "text": "Réflexion en cours...", "ts": "2026-11-24T12:35:03.000Z" },
    { "text": "Exécution : app_launch...", "ts": "2026-11-24T12:35:06.000Z" }
  ],
  "payload": {
    "prompt": "Ouvre YouTube",
    "deviceId": "MSG-10-31-DF-00"
  },
  "result": {
    "success": true,
    "summary": "Application lancée avec succès."
  },
  "createdAt": "Timestamp",
  "updatedAt": "Timestamp"
}
```

---

## 2. RÈGLES DE SÉCURITÉ FIRESTORE (`firestore.rules`)

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function isAuth() {
      return request.auth != null;
    }

    function isOwner(userId) {
      return isAuth() && request.auth.uid == userId;
    }

    // Profils et données privées
    match /users/{userId} {
      allow read, write: if isOwner(userId);

      match /sessions/{sessionId} {
        allow read, write: if isOwner(userId);
      }
      match /conversations/{convId} {
        allow read, write: if isOwner(userId);
        match /messages/{msgId} {
          allow read, write: if isOwner(userId);
        }
      }
    }

    // EVA CLOUDWORKS
    match /cloudworks/{userId} {
      allow read, write: if isOwner(userId);

      match /devices/{deviceId} {
        allow read, write: if isOwner(userId);
      }

      match /commands/{commandId} {
        allow read, write: if isOwner(userId);
      }
    }
  }
}
```

---

## 3. LES ÉTATS DU CYCLE DE VIE D'UNE COMMANDE

| État | Déclenché par | Description |
|---|---|---|
| `pending` | Émetteur (Web ou Chat) | La commande est écrite dans Firestore, en attente de prise en charge. |
| `running` | PC Desktop (`pc-agent.js`) | Le PC a détecté la commande et démarre le traitement / LLM local. |
| `executing` | Agent ReAct (`cw-agent.js`) | Un outil spécifique est en cours d'exécution sur le système. |
| `done` | Agent ReAct (`cw-agent.js`) | La tâche s'est achevée avec succès. |
| `error` | Agent ReAct (`cw-agent.js`) | Échec technique ou timeout. |
| `cancelled` | Utilisateur (UI) | Interruption volontaire via le bouton d'annulation. |
