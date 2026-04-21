# 🛡️ n8n Error Handling Patterns

> **Des workflows qui plantent, tout le monde en a. Des workflows qui se réparent tout seuls, c'est une autre histoire.**

Trois patterns progressifs de gestion d'erreurs dans n8n, avec explications, diagrammes et workflows importables.

<p align="center">
  <img src="https://img.shields.io/badge/n8n-1.x+-FF6D5A?style=flat&logo=n8n&logoColor=white" alt="n8n">
  <img src="https://img.shields.io/badge/Niveau-Débutant_→_Avancé-blue?style=flat" alt="Niveau">
  <img src="https://img.shields.io/badge/Licence-MIT-green?style=flat" alt="Licence">
</p>

---

## 🎯 Pourquoi la gestion d'erreurs est critique

Un workflow sans gestion d'erreurs, c'est une voiture sans freins. Sur un workflow exécuté 100 fois par jour pendant un mois, la probabilité de rencontrer au moins une erreur API, un timeout ou une réponse malformée approche les 100%.

Sans protection, les conséquences sont immédiates : perte de données silencieuse, effet cascade bloquant toute la chaîne, debugging aveugle, et intervention humaine forcée — exactement ce qu'on cherche à éliminer.

---

## 🏗️ Les trois patterns

| Pattern | Complexité | Protection | Cas d'usage |
|---------|-----------|------------|-------------|
| [Pattern 1](#-pattern-1--error-trigger-basique) — Error Trigger | ⭐ | Notification d'erreur | Workflows simples |
| [Pattern 2](#-pattern-2--retry-avec-notification) — Retry + Notification | ⭐⭐ | Récupération automatique | APIs instables |
| [Pattern 3](#-pattern-3--fallback-multi-fournisseurs) — Fallback + Logging | ⭐⭐⭐ | Résilience complète | Production 24/7 |

Chaque pattern **inclut le précédent** : le Pattern 3 intègre les mécanismes des Patterns 1 et 2.

---

## 🟢 Pattern 1 — Error Trigger basique

**Objectif** : Être averti quand un workflow échoue, au lieu de découvrir le problème des heures plus tard.

Par défaut, quand un nœud n8n échoue, le workflow s'arrête silencieusement. L'Error Trigger est un nœud spécial qui s'active automatiquement à chaque échec et route les détails de l'erreur vers un canal de notification (Email, Telegram, Slack).

```
  [Trigger] → [Nœud A] → [Nœud B] → [Nœud C]
                 │           │           │
                 └───── si erreur ───────┘
                              │
                     [Error Trigger]
                           │
                  [Notification Telegram]
```

**Mise en œuvre :**
1. Ajoute un nœud **Error Trigger** sur le canvas (il se déclenche seul, pas besoin de le connecter en amont).
2. Connecte un nœud de notification en sortie (Email, Telegram, Slack).
3. Utilise ces expressions pour le message :
   - `{{ $json.execution.error.message }}` → le message d'erreur
   - `{{ $json.execution.error.node.name }}` → le nœud qui a échoué
   - `{{ $json.execution.id }}` → l'ID de l'exécution
4. Dans les paramètres du workflow (engrenage) → "Error Workflow" → sélectionne ce workflow.

**Limite** : l'erreur est signalée mais pas corrigée — le workflow reste en échec.

📄 [Documentation complète](docs/pattern-1-basic.md) · 📦 [Workflow importable](examples/pattern-1-error-trigger.json)

---

## 🟡 Pattern 2 — Retry avec notification

**Objectif** : Gérer automatiquement les erreurs temporaires (timeout, rate limit, serveur indisponible) sans intervention humaine.

La majorité des erreurs en production sont temporaires. Un serveur renvoie 429 (trop de requêtes) ou 503 (indisponible), mais refonctionne 30 secondes plus tard. Ce pattern réessaie automatiquement avant de signaler.

```
  [Trigger] → [Nœud API Externe]  ←── Retry On Fail (3x, backoff exponentiel)
                     │
                Si échec final
                     │
              [Error Trigger]
                     │
            [Classifier l'erreur]
               │             │
         Temporaire?    Permanente?
            │                │
     [Log discret]    [Alerte urgente]
```

**Mise en œuvre :**
1. Sur chaque nœud appelant une API externe → onglet **Settings** :
   - Active **"Retry On Fail"**
   - Max Tries : **3**
   - Wait Between Tries : **2000** ms
   - Backoff : **exponential** (2s → 4s → 8s)
2. Après l'Error Trigger, ajoute un nœud **IF** pour classifier :
   - Condition : `{{ [429, 500, 502, 503, 504].includes($json.execution.error.statusCode) }}`
   - Vrai → erreur temporaire → log discret (Google Sheets / Airtable)
   - Faux → erreur permanente (401, 404...) → alerte Telegram + Email
3. Configure deux canaux de notification distincts selon la gravité.

**Limite** : si le fournisseur est en panne prolongée, les retries s'épuisent sans alternative.

📄 [Documentation complète](docs/pattern-2-intermediate.md) · 📦 [Workflow importable](examples/pattern-2-retry-notification.json)

---

## 🔴 Pattern 3 — Fallback multi-fournisseurs

**Objectif** : Un workflow qui continue de fonctionner même quand un fournisseur entier est hors service, en basculant automatiquement vers une alternative.

```
  [Trigger] → [Préparer requête]
                     │
              [Fournisseur Principal]  ←── Retry 3x
                     │
               Échec final ?
              Non /     \ Oui
              │          ▼
              │    [Log "Fallback activé"]
              │          │
              │    [Fournisseur Backup]  ←── Retry 2x
              │          │
              │    Échec final ?
              │   Non /     \ Oui
              │    │         ▼
              │    │   [Fournisseur Tertiaire]
              │    │         │
              ▼    ▼         ▼
           [Merge résultat unique]
                     │
              [Traitement aval]

  ─── En parallèle : chaque étape → [Log centralisé] ───
```

**Exemple concret — Agent IA résilient :**

| Priorité | Fournisseur | Coût approx. |
|----------|------------|--------------|
| 1 | Google Gemini 2.0 Flash | ~0.10€ / 1M tokens |
| 2 | OpenAI GPT-4o-mini via OpenRouter | ~0.30€ / 1M tokens |
| 3 | Anthropic Claude Haiku via OpenRouter | ~0.25€ / 1M tokens |

Si Gemini est en panne, le workflow bascule sur GPT-4o-mini. Si OpenAI est aussi indisponible, il passe sur Claude Haiku. L'utilisateur ne remarque rien.

**Mise en œuvre :**
1. Crée un nœud **Set** qui prépare la requête dans un format neutre (indépendant du fournisseur).
2. Construis la chaîne : Fournisseur 1 (retry 3x) → IF échec → Fournisseur 2 (retry 2x) → IF échec → Fournisseur 3.
3. Utilise un nœud **Merge** pour collecter le premier résultat valide.
4. À chaque tentative, envoie un log vers Airtable/Supabase : timestamp, fournisseur, statut, temps de réponse.
5. Si TOUS les fournisseurs échouent → alerte maximale tous canaux + stockage des données en file d'attente.

📄 [Documentation complète](docs/pattern-3-advanced.md) · 📦 [Workflow importable](examples/pattern-3-fallback-logging.json)

---

## 📥 Comment importer les exemples

1. Télécharge le fichier `.json` depuis le dossier `examples/`.
2. Dans n8n, clique sur **"..."** → **"Import from File"**.
3. Sélectionne le fichier JSON.
4. **Important** : configure tes propres credentials (identifiants) pour les nœuds de notification et API — ils ne sont jamais exportés pour des raisons de sécurité.

---

## 💡 Cas d'usage réels

- **Quota Gemini dépassé à 2h du matin** : le Pattern 3 a basculé sur OpenRouter sans perte de message pendant 6 heures.
- **API Telegram instable** : le Pattern 2 a absorbé les timeouts par retry exponentiel, évitant 47 fausses alertes en une journée.
- **OAuth2 expiré** : le Pattern 1 a notifié en 10 minutes au lieu de découvrir le lendemain que 200 emails n'avaient pas été envoyés.

---

## 🚀 Aller plus loin

- **[n8n-ai-agent-blueprints](https://github.com/walbuilds/n8n-ai-agent-blueprints)** — Patterns d'agents IA multi-étapes
- **[pinecone-memory-consolidation](https://github.com/walbuilds/pinecone-memory-consolidation)** — Mémoire persistante et déduplication sémantique
- **[n8n-google-workspace-automation](https://github.com/walbuilds/n8n-google-workspace-automation)** — Intégrations Google Workspace

---

## 🤝 Contribuer

Contributions bienvenues ! Consultez [CONTRIBUTING.md](CONTRIBUTING.md) pour les conventions.

## 📄 Licence

MIT — voir [LICENSE](LICENSE).

---

<p align="center">
  <strong>Créé par <a href="https://github.com/walbuilds">walbuilds</a></strong><br>
  <a href="https://www.linkedin.com/in/walid-abir-7b9138388/">LinkedIn</a> · <a href="https://dev.to/walbuilds">Dev.to</a> · <a href="https://hashnode.com/@walbuilds">Hashnode</a>
</p>
