# Pattern 3 — Fallback Multi-Fournisseurs avec Logging Centralisé

## Qu'est-ce que c'est ?

Le Pattern 3 est le niveau de protection maximal. Il combine les alertes du Pattern 1, les retries du Pattern 2, et ajoute une **bascule automatique entre fournisseurs**. Si ton fournisseur principal tombe, le workflow passe au backup, puis au tertiaire. Chaque étape est journalisée.

C'est l'équivalent d'un datacenter avec générateurs de secours : si le courant coupe, le générateur prend le relais instantanément et personne ne remarque la coupure.

## Concepts clés

**Fallback** : mécanisme de remplacement activé quand le système principal échoue. Tu configures plusieurs fournisseurs capables de rendre le même service, classés par priorité.

**Requête normalisée** : pour basculer entre fournisseurs, la requête est préparée dans un format neutre puis traduite dans le format spécifique de chaque API au moment de l'appel.

**Logging centralisé** : chaque tentative est enregistrée dans une base unique pour analyser la fiabilité de chaque fournisseur.

## La clé technique : "Continue Using Error Output"

Dans les paramètres de chaque nœud (Settings → On Error), l'option **"Continue Using Error Output"** est cruciale. Au lieu de stopper le workflow, le nœud produit une sortie contenant les détails de l'erreur, que le nœud suivant peut analyser et utiliser pour décider de basculer vers le fournisseur suivant.

## Mise en œuvre

### 1. Normaliser la requête

Un nœud **Set** prépare les données dans un format neutre :

```json
{
  "prompt": "Résume ce document en 3 points",
  "context": "Contenu du document...",
  "max_tokens": 500,
  "temperature": 0.3,
  "request_id": "unique_id_pour_traçabilité"
}
```

### 2. Chaîne Principale → Backup → Tertiaire

Chaque fournisseur est un nœud HTTP Request avec Retry On Fail activé et On Error réglé sur "Continue Using Error Output". Après chaque nœud, un nœud **IF** vérifie si une erreur est présente (`$json.error !== undefined`). Si oui → log + passage au fournisseur suivant. Si non → extraction du résultat.

| Fournisseur | Retry | On Error | Priorité |
|-------------|-------|----------|----------|
| Gemini 2.0 Flash | 3x, exponentiel | Continue Using Error Output | Principal |
| GPT-4o-mini via OpenRouter | 2x, exponentiel | Continue Using Error Output | Backup |
| Claude Haiku via OpenRouter | 1x | Continue Using Error Output | Tertiaire |

### 3. Collecter le premier résultat valide

Un nœud **Code** (JavaScript) vérifie séquentiellement quelle branche a produit un résultat :

```javascript
const gemini = $('Appel Gemini').first().json;
const openai = $('Appel OpenAI').first().json;
const claude = $('Appel Claude').first().json;

let result, provider;

if (gemini && !gemini.error) {
  result = gemini.candidates?.[0]?.content?.parts?.[0]?.text;
  provider = 'gemini';
} else if (openai && !openai.error) {
  result = openai.choices?.[0]?.message?.content;
  provider = 'openai';
} else if (claude && !claude.error) {
  result = claude.choices?.[0]?.message?.content;
  provider = 'claude';
} else {
  result = null;
  provider = 'none';
}

return [{ json: { result, provider_used: provider, all_failed: provider === 'none' } }];
```

### 4. Gérer l'échec total

Si `all_failed` est vrai : alerte maximale tous canaux + stockage de la requête en file d'attente pour retraitement ultérieur.

### 5. Logger chaque tentative

À chaque étape, un nœud envoie vers Airtable ou Google Sheets :

| Champ | Description | Exemple |
|-------|-------------|---------|
| timestamp | Date et heure | 2026-04-21T14:32:00Z |
| request_id | ID unique de la requête | a8f3k2_1713710000 |
| provider | Fournisseur appelé | gemini-2.0-flash |
| status | Résultat | failed / success |
| error_code | Code HTTP si échec | 429 |
| response_time_ms | Temps de réponse | 2340 |

Ce journal permet d'identifier les fournisseurs chroniquement instables et d'ajuster la chaîne de priorité en conséquence.

## Quand utiliser ce pattern

- Systèmes de production critiques 24/7 (bots, agents IA, pipelines de données).
- Tout workflow où le coût d'une interruption dépasse le coût de maintenance de fournisseurs multiples.
- Agents IA devant garantir une disponibilité quasi-totale pour des utilisateurs finaux.

## Coût réel du multi-fournisseurs

Le surcoût est minime. En fonctionnement normal, seul le fournisseur principal est utilisé (le moins cher). Les backup ne sont facturés que quand ils sont effectivement appelés. Sur un mois typique avec 99% de disponibilité du principal, le surcoût des fallbacks représente moins de 2% du budget API total.

## Évolutions possibles

- **Rotation automatique** : changer le fournisseur principal dynamiquement selon les performances observées dans les logs.
- **Circuit breaker** : si un fournisseur échoue X fois en Y minutes, le désactiver temporairement pour éviter de gaspiller des tentatives.
- **File d'attente persistante** : stocker les requêtes échouées dans Redis ou Supabase pour retraitement automatique quand le fournisseur redevient disponible