# Pattern 2 — Retry Automatique avec Notification Intelligente

## Qu'est-ce que c'est ?

Le Pattern 2 ajoute une couche de **récupération automatique** au-dessus de l'alerte du Pattern 1. Quand un nœud échoue, au lieu de planter immédiatement, il **réessaie automatiquement** plusieurs fois avec des intervalles croissants. Si l'erreur persiste après toutes les tentatives, il **classifie l'erreur** pour envoyer l'alerte adaptée.

C'est la différence entre une alarme incendie (Pattern 1) et un système sprinkler qui tente d'éteindre le feu avant de sonner l'alarme (Pattern 2).

## Concepts clés expliqués

### Retry On Fail (réessai automatique)

Chaque nœud n8n possède un paramètre caché dans l'onglet "Settings" qui permet de réessayer automatiquement en cas d'échec. Au lieu de s'arrêter à la première erreur, le nœud attend un délai puis retente l'opération.

### Backoff exponentiel

Le **backoff exponentiel** est une technique où le délai entre chaque tentative double à chaque fois : 2 secondes, puis 4, puis 8, puis 16... L'idée est que si un serveur est surchargé, lui envoyer des requêtes toutes les 2 secondes ne fait qu'aggraver le problème. En espaçant progressivement les tentatives, on lui laisse le temps de récupérer.

### Classification d'erreurs

Toutes les erreurs ne se valent pas :

- **Erreurs temporaires** (codes HTTP 429, 500, 502, 503, 504) : le serveur a un problème momentané. Réessayer a de bonnes chances de fonctionner.
- **Erreurs permanentes** (codes HTTP 400, 401, 403, 404) : la requête elle-même est invalide (mauvais identifiant, URL inexistante, permission refusée). Réessayer ne changera rien.

## Mise en œuvre détaillée

### Étape 1 — Activer le Retry sur les nœuds critiques

Pour chaque nœud qui appelle un service externe (HTTP Request, Gmail, Telegram, OpenAI, etc.) :

1. Ouvre le nœud et clique sur l'onglet **"Settings"** (en haut).
2. Descends jusqu'à la section **"On Error"**.
3. Change la valeur de **"Continue On Fail"** en gardant la valeur par défaut (Stop).
4. Active **"Retry On Fail"**.
5. Configure :
   - **Max Tries** : `3` (trois tentatives au total)
   - **Wait Between Tries (ms)** : `2000` (2 secondes avant le premier retry)
   - **Backoff** : `Exponential`

Avec cette configuration, si le nœud échoue :
- Tentative 1 : immédiate → échec
- Tentative 2 : après 2 secondes → échec
- Tentative 3 : après 4 secondes → échec
- → L'erreur est propagée au workflow

### Étape 2 — Créer le classificateur d'erreurs

Après ton Error Trigger (configuré selon le Pattern 1), ajoute un nœud **IF** :

**Configuration du nœud IF :**

- Condition : `Expression`
- Valeur 1 : `{{ $json.execution.error.message }}`
- Opération : `Contains`
- Valeur 2 : les indicateurs d'erreurs temporaires

En pratique, utilise un nœud **Code** (JavaScript) pour une classification plus précise :

```javascript
const errorMessage = $json.execution.error.message || '';
const statusCode = parseInt(errorMessage.match(/status code (\d+)/)?.[1] || '0');

const temporaryCodes = [429, 500, 502, 503, 504];
const isTemporary = temporaryCodes.includes(statusCode)
  || errorMessage.includes('ETIMEDOUT')
  || errorMessage.includes('ECONNRESET')
  || errorMessage.includes('socket hang up')
  || errorMessage.includes('rate limit');

return [{
  json: {
    ...($json),
    errorType: isTemporary ? 'temporary' : 'permanent',
    statusCode: statusCode,
    timestamp: new Date().toISOString()
  }
}];
```

### Étape 3 — Router selon le type d'erreur

Après le classificateur, ajoute un nœud **IF** :
- Condition : `{{ $json.errorType }}` est égal à `temporary`

**Branche "Vrai" (erreur temporaire)** :
- Nœud **Google Sheets** ou **Airtable** : enregistre l'erreur dans un journal avec la date, le type et le message. C'est un log discret pour analyse ultérieure.
- Pas d'alerte immédiate : ces erreurs sont normales et attendues en production.

**Branche "Faux" (erreur permanente)** :
- Nœud **Telegram** + **Email** : alerte urgente nécessitant une action humaine.
- Message incluant le contexte complet : quel workflow, quel nœud, quel code d'erreur, quel timestamp.

### Étape 4 — Mettre en place le tableau de suivi

Crée une feuille Google Sheets ou une table Airtable avec les colonnes suivantes :

| Colonne | Contenu | Exemple |
|---------|---------|---------|
| Date | Timestamp ISO | 2026-04-21T14:32:00Z |
| Workflow | Nom du workflow | Mon Agent Telegram |
| Nœud | Nœud en échec | Appel Gemini API |
| Type | temporary / permanent | temporary |
| Code | Code HTTP | 429 |
| Message | Message d'erreur complet | Rate limit exceeded |
| Résolu | Oui / Non (auto-résolu par retry) | Oui |

Ce tableau te permet d'identifier rapidement les fournisseurs chroniquement instables et de prendre des décisions éclairées (changer de fournisseur, ajuster les quotas, etc.).

## Quand utiliser ce pattern

- Tout workflow qui appelle une API externe au moins une fois par heure.
- Workflows de synchronisation de données entre services (Gmail → Airtable, Stripe → Google Sheets).
- Bots Telegram ou WhatsApp qui appellent des modèles d'IA.

## Limites connues

- Si le fournisseur est en panne prolongée (plusieurs heures), les retries s'épuisent sans alternative.
- Aucun mécanisme de bascule vers un fournisseur de remplacement.
- Le retry consomme des exécutions n8n supplémentaires (pertinent sur les plans cloud limités).

Ces limites sont adressées par le [Pattern 3](pattern-3-advanced.md).
