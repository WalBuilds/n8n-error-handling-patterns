# Glossaire — Termes techniques utilisés dans ce dépôt

Ce glossaire explique en langage clair tous les termes techniques rencontrés dans la documentation.

## A

**API (Application Programming Interface)** : Une interface qui permet à deux logiciels de communiquer entre eux. Quand ton workflow n8n envoie un message à Telegram, il utilise l'API de Telegram. C'est comme un guichet : tu poses ta question dans un format précis, et tu reçois une réponse dans un format précis.

## B

**Backoff exponentiel** : Technique où le temps d'attente entre chaque nouvelle tentative double à chaque fois (2s → 4s → 8s → 16s). L'idée est de ne pas surcharger un serveur déjà en difficulté en lui envoyant des requêtes trop rapprochées.

## C

**Canvas** : L'espace de travail visuel de n8n où tu places et connectes tes nœuds pour construire un workflow.

**Credentials** : Les identifiants de connexion (clés API, mots de passe, tokens) qui permettent à n8n de se connecter à des services externes. Ils ne sont jamais exportés avec les workflows pour des raisons de sécurité.

**Cron** : Un système de programmation horaire qui déclenche une tâche automatiquement à des moments prédéfinis (toutes les heures, tous les jours à 8h, etc.).

## E

**Error Trigger** : Un nœud spécial de n8n qui se déclenche automatiquement quand une erreur survient dans le workflow. Il n'a pas besoin d'être connecté en amont — il fonctionne de manière autonome.

**Exécution** : Une instance unique d'un workflow qui a été lancée. Chaque exécution a un identifiant unique et est enregistrée dans l'historique de n8n.

## F

**Fallback** : Un plan de secours. Si le système principal échoue, le fallback prend automatiquement le relais. Exemple : si Gemini ne répond pas, on bascule sur OpenAI.

## H

**HTTP Status Code (Code de statut HTTP)** : Un nombre renvoyé par un serveur pour indiquer le résultat d'une requête :
- **200** : tout va bien.
- **400** : ta requête est mal formée (erreur permanente).
- **401** : identifiant ou mot de passe incorrect (erreur permanente).
- **403** : accès interdit (erreur permanente).
- **404** : la ressource demandée n'existe pas (erreur permanente).
- **429** : trop de requêtes envoyées trop vite (erreur temporaire — réessayer plus tard).
- **500** : erreur interne du serveur (erreur temporaire — le serveur a un problème).
- **502/503/504** : serveur indisponible ou saturé (erreur temporaire).

## L

**Logging** : L'enregistrement systématique des événements (succès, erreurs, temps de réponse) dans un journal. Permet d'analyser après coup ce qui s'est passé et de détecter des tendances.

## N

**Nœud (Node)** : Un bloc fonctionnel dans n8n. Chaque nœud effectue une action spécifique (envoyer un email, appeler une API, transformer des données). Les nœuds sont connectés entre eux pour former un workflow.

## O

**OAuth2** : Un protocole d'authentification qui permet à une application (n8n) d'accéder à un service (Gmail, Google Drive) au nom d'un utilisateur, sans stocker son mot de passe directement.

## R

**Rate limit** : Une limite imposée par un serveur sur le nombre de requêtes qu'il accepte par minute/heure. Dépasser cette limite provoque une erreur 429.

**Retry On Fail** : Un paramètre de n8n qui relance automatiquement un nœud en échec un certain nombre de fois avant de le considérer comme définitivement échoué.

## T

**Timeout** : Le temps maximum qu'un nœud attend une réponse avant de considérer que la requête a échoué. Par défaut, n8n attend 300 secondes (5 minutes).

**Token** : Dans le contexte des modèles d'IA, un token est approximativement un mot ou un morceau de mot. Les APIs d'IA facturent généralement à l'usage en nombre de tokens envoyés et reçus.

## W

**Webhook** : Une URL qui reçoit automatiquement des données quand un événement se produit ailleurs. Exemple : Gumroad envoie un webhook à n8n chaque fois qu'une vente est effectuée.

**Workflow** : Un enchaînement de nœuds dans n8n qui automatise un processus complet, du déclenchement initial au résultat final.
