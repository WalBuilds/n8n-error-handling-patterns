# Pattern 1 — Error Trigger Basique

## Qu'est-ce que c'est ?

Le Pattern 1 est le niveau de protection le plus simple. Son rôle est uniquement de **t'alerter** quand quelque chose casse dans un workflow n8n. Il ne répare rien, il ne réessaie rien — il t'envoie un message pour te dire "ça a planté, voici pourquoi".

C'est l'équivalent d'une alarme incendie : elle ne combat pas le feu, mais elle te réveille pour que tu puisses agir.

## Prérequis

- Une instance n8n fonctionnelle (self-hosted ou cloud).
- Au moins un canal de notification configuré : Email (SMTP), Telegram Bot, ou Slack Webhook.

## Comment fonctionne l'Error Trigger dans n8n ?

L'**Error Trigger** est un nœud spécial disponible dans n8n. Contrairement aux autres nœuds, il n'est pas connecté dans la chaîne principale du workflow. Il vit "à côté" et se réveille uniquement quand une erreur se produit n'importe où dans le workflow.

Quand il se déclenche, il reçoit automatiquement un objet JSON contenant toutes les informations sur l'erreur :

```json
{
  "execution": {
    "id": "1234",
    "url": "https://ton-instance.n8n.cloud/execution/1234",
    "error": {
      "message": "Request failed with status code 401",
      "node": {
        "name": "Appel API Gmail",
        "type": "n8n-nodes-base.gmail"
      }
    },
    "lastNodeExecuted": "Appel API Gmail",
    "mode": "trigger"
  }
}
```

## Mise en œuvre détaillée

### Étape 1 — Créer le workflow d'erreur

Tu as deux options :

**Option A (recommandée pour débuter)** : ajouter l'Error Trigger directement dans le workflow à surveiller. C'est plus simple mais cela signifie que chaque workflow contient sa propre gestion d'erreurs.

**Option B (recommandée en production)** : créer un **workflow dédié aux erreurs**, séparé, qui centralise la gestion d'erreurs pour tous tes workflows. Tu le configures ensuite comme "Error Workflow" dans chaque workflow à surveiller.

### Étape 2 — Configurer le nœud Error Trigger

1. Ouvre ton workflow (ou crée le workflow dédié).
2. Clique sur le bouton **"+"** pour ajouter un nœud.
3. Cherche **"Error Trigger"** dans la barre de recherche.
4. Place-le sur le canvas. Il n'a aucun paramètre à configurer — il fonctionne immédiatement.

### Étape 3 — Connecter la notification

Connecte un nœud de notification en sortie de l'Error Trigger.

**Exemple avec Telegram :**

Configure le nœud Telegram avec les champs suivants :

- **Chat ID** : l'identifiant de ton chat personnel ou d'un groupe Telegram.
- **Message** (en utilisant les expressions n8n) :

```
🚨 ERREUR WORKFLOW

📌 Workflow : {{ $workflow.name }}
❌ Nœud en échec : {{ $json.execution.error.node.name }}
💬 Message : {{ $json.execution.error.message }}
🔗 Exécution : {{ $json.execution.url }}
🕐 Date : {{ $now.format('dd/MM/yyyy HH:mm:ss') }}
```

**Exemple avec Email :**

- **Sujet** : `🚨 Erreur n8n — {{ $workflow.name }}`
- **Corps** : même contenu que ci-dessus, en format HTML si souhaité.

### Étape 4 — Lier le workflow d'erreur

Si tu utilises l'Option B (workflow dédié) :

1. Ouvre le workflow que tu veux surveiller.
2. Clique sur l'icône **engrenage** (paramètres du workflow).
3. Dans la section **"Error Workflow"**, sélectionne ton workflow dédié aux erreurs.
4. Sauvegarde.

Désormais, toute erreur dans ce workflow déclenchera automatiquement ton workflow d'erreur.

## Bonnes pratiques

- **Teste toujours ton Error Trigger** en provoquant volontairement une erreur (par exemple, un nœud HTTP Request vers une URL inexistante). Un Error Trigger non testé est un Error Trigger qui ne fonctionne peut-être pas.
- **Inclus toujours l'URL d'exécution** dans le message de notification. Cela te permet de cliquer directement sur le lien depuis Telegram ou ton email pour voir les détails dans l'interface n8n.
- **Privilégie un workflow d'erreur centralisé** (Option B) dès que tu gères plus de 3 workflows. Cela évite la duplication et garantit que toutes les erreurs passent par le même canal.

## Limites connues

- L'Error Trigger ne se déclenche pas si le workflow lui-même ne démarre pas (par exemple, une erreur de cron ou un webhook inaccessible).
- Si le nœud de notification échoue aussi (email SMTP down, bot Telegram bloqué), l'alerte est perdue silencieusement.
- Aucune récupération automatique : tu dois relancer manuellement le workflow après correction.

Ces limites sont adressées par le [Pattern 2](pattern-2-intermediate.md).
