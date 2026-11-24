# DIAGNOSTIC DES PROBLÈMES RENCONTRÉS — POURQUOI ÇA N'A PAS AVANCÉ

---

## 1. HISTORIQUE DES BLOCAGES ET ANOMALIES

Durant les nombreuses itérations précédentes, plusieurs problèmes se sont entremêlés, créant une confusion entre le fonctionnement réel du LLM local et des comportements parasites :

### A. La présence d'un "Fast Path" trompeur (Faux script automatique)
- **Le problème** :
  Dans `pc-agent.js` (lignes 425 à 752 de l'ancienne version), un développeur précédent avait injecté 327 lignes de code de substitution (Fast Path regex).
  Dès que l'utilisateur tapait *"Ouvre YouTube"* ou *"Lance le bloc-notes"*, une regex interceptait la commande AVANT le LLM et lançait directement un `Start-Process msedge https://youtube.com` via PowerShell en 50 millisecondes, puis affichait : *"C'est tout bon, j'ai effectué toutes les actions demandées"*.
- **La conséquence** :
  L'utilisateur a légitimement constaté que le LLM n'exécutait rien : c'était un script automatique codé en dur qui masquait l'absence de fonctionnement réel de l'IA agentique.
- **La solution apportée** :
  Ce bloc de 327 lignes a été supprimé pour que 100% des tâches passent obligatoirement par le LLM local `CWAgent`.

### B. Le comportement du modèle 3B compact (Qwen 2.5 3B GGUF)
- **Le problème** :
  Le modèle compact 3B est très rapide et léger (1.9 Go), mais il a des limites cognitives strictes :
  Lorsqu'on lui demande dans le system prompt :
  `1. Écris ta pensée entre <thought>...</thought>`
  `2. Ensuite appelle un outil avec <tool_call>{"name": "...", "args": {...}}</tool_call>`
  Le modèle produit souvent :
  `<thought>L'utilisateur veut ouvrir YouTube. C'est un site web. J'utilise app_launch avec l'URL directement https://youtube.com.</thought>`
  ... et il s'arrête là ! Il n'émet PAS la balise `<tool_call>`.
- **La conséquence dans le code** :
  Dans l'ancien `cw-agent.js`, `_parseToolCall` cherchait uniquement la chaîne `<tool_call>`. Comme il ne la trouvait pas :
  - Soit il considérait le texte visible comme une réponse prose finale, et terminait la tâche immédiatement sans rien exécuter (affichant *"Tâche terminée"* avec *"Réflexion en cours"* sur la même carte).
  - Soit il envoyait un "nudge" (*"Tu as bien réfléchi, maintenant passe à l'action"*), et le modèle recommençait à réfléchir en boucle jusqu'au timeout !
- **La solution apportée** :
  Un parseur d'intention V2 a été écrit pour détecter automatiquement quand le modèle mentionne une URL ou un outil dans sa pensée.

### C. La confusion entre le Chat et CloudWorks
- Le chat principal utilise les providers configurés dans les paramètres utilisateur (par défaut Puter, ou OpenAI, ou Pollinations).
- Si l'utilisateur est sur un compte Puter sans crédits (`Low balance`), le chat renvoie une erreur d'IA.
- L'utilisateur s'attendait à ce que CloudWorks prenne le relais automatiquement ou soit 100% indépendant. Mais lorsque le chat tente de générer une réponse conversationnelle avant de déclencher CloudWorks, le chat échoue à cause du quota Puter.
- L'utilisateur a formellement refusé toute bascule automatique silencieuse vers Pollinations ("ses non pas de fall back").

### D. La carte Tracker UI (Désynchronisation Firestore / Local)
- En mode local (Tâche libre), l'interface PC ne créait pas systématiquement le document Firestore avant d'appeler `appendCloudWorksTracker()`.
- De ce fait, le tracker attendait des événements Firestore qui n'étaient jamais poussés, bloquant l'UI avec un bouton désactivé.

---

## 2. POURQUOI CES BLOCAGES PERSISTENT-ILS ?

1. **Fragmentation du flux d'exécution** :
   Il existe actuellement 3 chemins de code différents pour faire la même chose :
   - Chemin 1 : Chat EVA (`messages.js` -> `file-gen.js` -> `executeEvaAction`).
   - Chemin 2 : Écouteur Firestore (`pc-agent.js` -> `runAgenticLoop`).
   - Chemin 3 : Tâche libre CloudWorks (`cloudworks.js` -> `_cwRunAgenticTask` -> `CWAgent`).
   Cette triplication fait qu'une correction sur l'un des chemins n'a aucun effet sur les deux autres !

2. **Divergence entre le Web et le Desktop** :
   Sur le Web, le code n'a évidemment AUCUN accès aux API Windows locales (`window.eva` n'existe pas). Il doit obligatoirement passer par Firestore.
   Sur le Desktop, le code a un accès direct aux API locales, mais écoute également Firestore pour les ordres distants. Si les deux se mélangent, des boucles de feedback ou des doubles exécutions se produisent.

3. **Absence de grammaire GBNF stricte sur le LLM local** :
   Sans contraindre la génération de tokens de `node-llama-cpp` via une grammaire formelle GBNF ou un schéma JSON strict, le modèle 3B continuera de varier sa syntaxe de sortie d'un run à l'autre.
