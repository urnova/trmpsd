# PROMPT OFFICIEL POUR GPT 6.2 SOL — AUDIT ET RÉSOLUTION ARCHITECTURALE DU PROJET E.V.A

---

## 🎯 OBJECTIF DU PROMPT
Tu es GPT 6.2 SOL, expert mondial en architectures multiplateformes (Web PWA + Desktop Electron + Firebase Cloud + IA Locale GGUF/node-llama-cpp).
Le projet E.V.A (Evolutionary Virtual Assistant) est un assistant virtuel IA multiplateforme développé par Astral Technologie.
Le projet subit un blocage persistant sur la phase D1 (système agentique CloudWorks et son moteur LLM local). Après de multiples itérations, des bugs de boucle, de détection d'outils et de synchronisation subsistent.

Tu disposes dans cette archive de l'intégralité du code source architectural : l'application PC Electron (`eva-pc`), l'application Web PWA (`EVA_V4_fixed_v4`), les adaptateurs Firebase Firestore, les scripts de chat, et le moteur agentique local CloudWorks.

Ta mission est d'effectuer un audit chirurgical, de pointer exactement pourquoi le système échoue, et de fournir une solution définitive, robuste et sans bricolage.

---

## 🏗️ PRÉSENTATION DE L'ARCHITECTURE E.V.A

Le système est composé de 3 piliers interconnectés :

1. **L'Application Navigateur Web (PWA / Vercel)** :
   - Hébergée sur Vercel, accessible depuis n'importe quel navigateur (PC, smartphone, tablette).
   - Utilise Firebase Authentication (email/password) et Cloud Firestore (SDK Web modulaire v9/v10 compat).
   - Permet de discuter avec EVA via des providers Cloud (Puter, OpenAI, Pollinations).
   - Permet d'envoyer des commandes de contrôle à distance à un PC connecté via Firestore (`cloudworks/{uid}/commands/{cmdId}`).

2. **L'Application PC Desktop (Electron 31 / Node.js 20)** :
   - Packagée avec `electron-builder` en installateur NSIS sous Windows.
   - Embarque un moteur LLM local autonome basé sur `node-llama-cpp` v3 exécutant un modèle GGUF 3B Q4_K_M (`EVA-PC-Agentic-3B-Q4_K_M-v5.gguf`).
   - Gère le cycle de vie du LLM (`llm:start`, `llm:chat`, `llm:stop`, `llm:status`) via IPC Electron (`main.ts` -> `preload.ts` -> `window.eva.system`).
   - Possède deux modes d'exécution :
     - **Mode Écoute Distante (`pc-agent.js`)** : Écoute en temps réel les documents Firestore `cloudworks/{uid}/commands` pour exécuter les demandes venant du web ou du chat.
     - **Mode Local Direct (`cloudworks.js` / `cw-agent.js`)** : L'utilisateur utilise la section "Tâche IA libre (LLM local)" ou le chat directement dans l'application PC.

3. **Le Pont de Synchronisation (Firebase Cloud Firestore)** :
   - Collection `cloudworks/{uid}/devices/{deviceId}` : Présence des PC connectés (`online: true`, hostname, LLM disponible).
   - Collection `cloudworks/{uid}/commands/{commandId}` : Tâches agentiques créées soit par le chat, soit par la tâche libre.
   - Les champs d'une commande :
     `{ id, type: 'agentic_task', status: 'pending'|'running'|'done'|'error'|'cancelled', step: string, steps: array, payload: { prompt }, createdAt, updatedAt, result }`.
   - Les interfaces (Web et PC) écoutent ce document pour afficher une carte de progression ("tracker card") en direct.

---

## ⚠️ LES SYMPTÔMES ET LES BLOCAGES RENCONTRÉS (PHASE D1)

Voici ce qui se produit concrètement lors des tests réels de l'utilisateur :

### 1. Dans le Chat EVA ("Ouvre YouTube", "Ouvre la calculatrice", etc.) :
- **Comportement constaté précédemment** :
  Soit EVA répondait *"Je m'en occupe tout de suite"* mais rien ne se lançait, soit un ancien script d'expressions régulières (Fast Path) court-circuitait le LLM local et ouvrait l'application instantanément sans que le LLM ne réfléchisse, soit le chat affichait une erreur *"Low balance Puter"* car l'appel au LLM de chat utilisait un compte Puter à 0 crédit.
- **Ce qui est attendu** :
  L'utilisateur veut que le chat déclenche CloudWorks et que le travail soit réellement opéré par le LLM local sur le PC, sans faux script et sans consommer de tokens payants/cloud pour la partie agentique.

### 2. Dans la section CloudWorks "Tâche IA libre (LLM local)" :
- **Comportement constaté sur capture d'écran réelle** :
  L'utilisateur clique sur "Exécuter avec le LLM" pour *"Ouvre YouTube"*.
  Le LLM local (3B Qwen) démarre, réfléchit et écrit dans sa pensée (`<thought>`) :
  > *"L'utilisateur veut ouvrir YouTube. YouTube est un site web, pas une application Windows installée. La bonne approche est d'ouvrir https://youtube.com dans le navigateur par défaut. Je n'ai pas besoin de chercher "YouTube" dans le registre. J'utilise app_launch avec l'URL directement."*
  Mais ensuite :
  - La carte affiche *"TÂCHE TERMINÉE"* tout en laissant *"Réflexion en cours..."*.
  - AUCUN navigateur ne s'ouvre !
  - Aucune action réelle n'est effectuée sur la machine !
  - L'utilisateur a l'impression d'une boucle infinie ou d'un résultat bidon.
- **Pourquoi ce blocage arrive** :
  Le petit modèle 3B Qwen a exprimé sa décision d'action directement dans son texte de réflexion, mais n'a pas produit la balise JSON stricte `<tool_call>{"name": "app_launch", "args": {"target": "https://youtube.com"}}</tool_call>`.
  L'agent ReAct (`cw-agent.js`) ne trouvait pas de balise `<tool_call>`, considérait le texte restant comme une réponse finale et fermait la tâche avec `success: true` sans jamais appeler `app_launch` !

### 3. La bulle / fenêtre CloudWorks manquante :
- Sur la version web ou PC, l'utilisateur s'attend à voir la bulle tracker en haut ou l'overlay CloudWorks réagir de façon synchrone. Si Firestore ou les écouteurs locaux ne sont pas synchronisés au millimètre près, la bulle n'apparaît pas ou reste figée sur "Démarrage du LLM...".

---

## 📁 FICHIERS FOURNIS DANS CETTE ARCHIVE ZIP

1. **`01_RAPPORT_ARCHITECTURE_COMPLETE.md`** : Description ultra-détaillée des protocoles et flux.
2. **`02_DIAGNOSTIC_PROBLEMES_RENCONTRES.md`** : Historique et causes racines précises des bugs D1.
3. **`03_SPECIFICATIONS_ET_REGLES_FIREBASE.md`** : Règles de sécurité `firestore.rules` et schéma de données.
4. **Code Electron PC** :
   - `main.ts` : Processus principal Electron, pont IPC, moteur `node-llama-cpp`.
   - `preload.ts` : Exposition sécurisée de l'API `window.eva`.
   - `pc-bridge.js` : Pont côté UI renderer.
5. **Code Moteur CloudWorks (Agent ReAct local)** :
   - `cw-agent.js` : Boucle ReAct (`run`), prompt système, parseur d'outils, appel LLM.
   - `cw-tools.js` : Outils système (`app_launch`, `app_resolve`, `file_create`, `web_browse`, PowerShell).
   - `cloudworks.js` : UI du tableau de bord CloudWorks, bouton tâche libre.
   - `pc-agent.js` : Boucle d'exécution des tâches Firestore.
   - `cw-modal.js` / `cw-modal-enhanced.js` : Cartes de suivi UI.
   - `cw-engine.js` : Machine d'états.
6. **Code Chat & Providers** :
   - `messages.js` : Dispatcher du chat, détection d'intentions CloudWorks.
   - `core.js` : Moteur de discussion principal.
   - `providers.js` : Gestion des providers IA (Puter, Qwen, Pollinations, OpenAI).
   - `memory.js` : Mémoire évolutive.

---

## 📋 QUESTIONS ET LIVRABLES ATTENDUS DE TA PART

En tant que GPT 6.2 SOL, réponds avec le plus haut niveau de rigueur technique aux questions suivantes :

1. **Architecture Unifiée Chat vs CloudWorks** :
   Comment unifier proprement le déclenchement d'une action système, qu'elle vienne du Chat Web, du Chat PC ou de la Tâche Libre PC, pour que le flux soit rigoureusement identique, sans duplication de code et sans double-écouteur Firestore ?

2. **Fiabilité de l'Agent ReAct avec un modèle compact (Qwen 3B)** :
   Un modèle 3B local ne respectera jamais à 100% un formatage JSON rigide `<tool_call>`.
   Quelle est la stratégie définitive pour contraindre le modèle (via le système prompt, la grammaire GBNF de node-llama-cpp, ou un parseur tolérant) afin qu'une pensée comme *"J'utilise app_launch sur https://youtube.com"* ou *"Je lance Discord"* déclenche SYSTÉMATIQUEMENT l'outil sans jamais terminer prématurément la tâche ?

3. **Cycle de vie Firestore & UI Tracker** :
   Comment garantir que la carte de suivi (dans le chat et dans CloudWorks) passe de manière fluide et garantie par :
   `running` (Démarrage LLM) ➔ `running` (Réflexion) ➔ `executing` (Nom de l'outil en cours) ➔ `done` (Résultat final vérifié) ?
   Comment s'assurer que le bouton d'envoi et l'input ne restent JAMAIS verrouillés ?

4. **Plan d'implémentation direct** :
   Donne le code exact, corrigé et prêt à l'emploi pour les fichiers centraux (`cw-agent.js`, `pc-agent.js`, `main.ts`, `messages.js`).
