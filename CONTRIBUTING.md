# Contribuer à n8n Error Handling Patterns

Merci de considérer une contribution ! Voici les conventions à respecter.

## Types de contributions acceptées

- **Nouveaux patterns** : si vous avez un pattern de gestion d'erreurs non couvert, ouvrez une Issue pour en discuter avant de soumettre une PR.
- **Améliorations de documentation** : corrections, clarifications, traductions.
- **Exemples supplémentaires** : workflows JSON illustrant un cas d'usage spécifique.
- **Corrections de bugs** : dans les workflows d'exemple.

## Conventions

### Structure d'un nouveau pattern

Chaque pattern doit inclure :

1. Un fichier de documentation dans `docs/` expliquant le problème, la solution et la mise en œuvre.
2. Un workflow JSON importable dans `examples/`.
3. Une mise à jour du README principal avec un lien vers le nouveau pattern.

### Workflows JSON

- **Anonymiser** : ne jamais inclure de clés API, tokens, mots de passe ou données personnelles.
- **Documenter** : chaque nœud doit avoir un nom descriptif en français.
- **Tester** : le workflow doit être fonctionnel après import (hors credentials à configurer).

### Style de documentation

- Français pour la documentation principale, anglais accepté pour les commentaires de code.
- Ton accessible : expliquer les concepts techniques comme si le lecteur les découvrait.
- Inclure des exemples concrets tirés de situations réelles.

## Processus

1. **Fork** le dépôt.
2. Crée une **branche** descriptive (`feature/pattern-circuit-breaker`).
3. Effectue tes modifications.
4. Soumets une **Pull Request** avec une description claire du changement.

## Code de conduite

Soyez respectueux et constructif. Ce projet est maintenu bénévolement.
