# Claude Code + Board Game Arena : Développement de jeux assisté par IA

## Pour les développeurs BGA : démarrage rapide

**Vous voulez utiliser Claude Code pour développer votre jeu BGA ?** Ce repo fournit un fichier skill et un workflow qui donnent à Claude Code les connaissances nécessaires pour travailler efficacement dans les contraintes de BGA Studio.

> **Ce projet est en cours de développement.** Le fichier skill et le workflow sont activement affinés à travers plusieurs expériences de jeux. Ils fonctionnent, mais attendez-vous à des imperfections. Vos retours sont précieux — [ouvrez une issue](../../issues) si quelque chose ne fonctionne pas, manque, ou pourrait être plus clair.

### Démarrage (5 minutes)

1. **Installez Claude Code** — [CLI](https://claude.ai/claude-code), application desktop, ou extension IDE
2. **Installez [Claude in Chrome](https://chrome.google.com/webstore/detail/claude-in-chrome)** — nécessaire pour la boucle de test automatisée sur BGA Studio
3. **Copiez le fichier skill** dans votre répertoire de skills Claude Code :
   ```bash
   mkdir -p ~/.claude/skills/bga-alpha
   cp SKILL.md ~/.claude/skills/bga-alpha/SKILL.md
   ```
4. **Ayez votre compte BGA Studio prêt** — jeu enregistré, clé SSH configurée ([docs BGA](https://en.doc.boardgamearena.com/Studio))
5. **Démarrez Claude Code dans votre répertoire de projet** et demandez-lui d'implémenter votre jeu :
   ```
   Je veux implémenter [NOM DU JEU] sur BGA Studio.
   Voici les règles : [collez les règles ou un lien vers le livre de règles]
   Mon nom d'utilisateur BGA est [USERNAME] et le nom du jeu est [GAMENAME].
   ```

Claude Code utilisera le fichier skill pour gérer la configuration BGA (Makefile, fichiers de config, machine à états, workflow de déploiement) et l'extension navigateur pour tester directement sur BGA Studio.

### Ce que couvre le fichier skill

- Scaffolding du projet (Makefile, structure de répertoires, fichiers de config)
- Patterns du nouveau framework BGA (PHP 8.4, classes d'état, notifications, globales)
- Structure du client JavaScript (modules ES6, gestionnaires d'état)
- Boucle deploy-test automatisée via l'extension Chrome
- Pièges connus de BGA et comment les éviter (commentaires SQL, noms de tables, problèmes de requêtes DB)

### Ce que vous devez encore faire vous-même

- Créer le compte BGA Studio et enregistrer votre jeu
- Configurer la clé SSH pour l'accès SFTP (peut prendre jusqu'à 1h pour se propager)
- Quitter manuellement les tables de jeu bloquées quand l'interface BGA le requiert
- Reconnecter l'extension Chrome si elle se déconnecte
- Relire le code généré — c'est une alpha, pas du code prêt pour la production

Consultez `workflow.md` pour la séquence de développement complète, et `prompts/` pour des exemples de prompts issus de la première expérience.

---

## À propos de ce projet

Ce dépôt documente une méthode pour développer des jeux sur [Board Game Arena](https://studio.boardgamearena.com) en utilisant [Claude Code](https://claude.ai/claude-code) comme outil de développement principal — des règles publiées à un jeu hotseat jouable, sans écriture manuelle de code.

La première expérience utilise le **Quantum Tic-Tac-Toe** (Allan Goff, AJP 2006). D'autres expériences avec d'autres jeux sont en cours. L'objectif de ce repo n'est pas un jeu spécifique — c'est la **méthode** : un workflow reproductible pour utiliser un assistant de codage IA dans un environnement réel et contraint.

### Ce que cela démontre

1. **Développement sous contraintes** — BGA est un environnement fixe : framework PHP spécifique, déploiement SFTP uniquement, pas d'accès shell, UI basée sur Svelte. L'IA doit travailler dans ces contraintes, pas autour.
2. **Les règles comme spécification** — Les règles du jeu publiées servent de spécification formelle. Pas de chef de produit, pas de document de conception — juste un livre de règles.
3. **Le navigateur comme banc de test** — Sans accès SSH aux serveurs BGA, le navigateur (via Claude in Chrome) est le seul moyen d'observer le comportement à l'exécution, lire les erreurs, et vérifier l'état du jeu.
4. **Les fichiers skill comme connaissance accumulée** — Le `SKILL.md` capture les patterns BGA spécifiques, les pièges et les comportements du framework découverts pendant le développement. Il persiste entre les sessions et évite de répéter les mêmes erreurs.

### Ce que cela ne prétend PAS

- Que l'IA peut développer de manière autonome des logiciels de production
- Que le code du jeu résultant est optimal ou complet
- Que cette approche s'adapte à des jeux complexes sans supervision humaine significative

## Structure du dépôt

```
README.md               ← version anglaise (vous êtes ici)
README.fr.md            ← version française
SKILL.md                ← fichier skill Claude Code pour le développement BGA
TECHNICAL_NOTES.md      ← explications détaillées des pièges et de leurs causes
workflow.md             ← workflow de développement étape par étape
CONTRIBUTING.md         ← comment contribuer ou adapter cette méthode
prompts/
  README.md             ← comment utiliser les templates de prompts
  01-setup.md           ← scaffolding du projet et configuration BGA
  02-core-mechanics.md  ← implémentation de la logique de jeu
  03-ui.md              ← rendu côté client et interaction
screenshots/            ← captures d'écran annotées des moments clés
game/                   ← code source du jeu (quantictactoe)
experiments/            ← résultats d'expériences avec d'autres jeux
```

## Résultats

Ce workflow a été développé et testé sur deux jeux jusqu'à présent :

### Quantum Tic-Tac-Toe — [source](https://github.com/rbellec/BGA_quantum_tic_tac_toe)

*Complété en une seule session Claude Code (~3 heures de temps réel) :*

- Implémentation complète du jeu BGA : 4 classes d'état PHP, 1 client JS, schéma SQL, fichiers de config
- Mécaniques de jeu fonctionnelles : placement de marques quantiques, graphe d'intrication, détection de cycles (DFS), cascade d'effondrement, détection de victoire
- 3 bugs trouvés et corrigés pendant la boucle de test (suppression de commentaires SQL, conflit de nom de table, collision de clé `getCollectionFromDb`)
- Première partie complète jouée de bout en bout sur BGA Studio (7 coups, 2 effondrements, GoOn0 gagne)

*Métriques :*

- Lignes de code produites : ~850 (PHP + JS + SQL + CSS)
- Cycles deploy-test : ~15
- Interventions humaines : ~5 (configuration clé SFTP, Express Stop sur tables bloquées, reconnexion de l'extension navigateur)

### Go On Rasalva — [source](https://github.com/rbellec/Go-On-Rasalva-on-BGA)

*(Résultats à documenter)*

## Limitations

- **Stade précoce** — le fichier skill a été validé sur deux jeux pour l'instant ; davantage d'expériences sont nécessaires
- **Règles simplifiées uniquement** — la première expérience a implémenté une variante simplifiée ; les règles complètes nécessitent plus d'itérations
- **Pas de tests** — toute la validation se fait via la boucle de test navigateur ; pas de tests unitaires
- **UI minimale** — fonctionnelle mais non peaufinée
- **Connaissance BGA spécifique requise** — le fichier skill encode une connaissance du framework durement acquise ; sans lui, l'IA répète les mêmes pièges
- **Fragilité de l'extension navigateur** — la connexion Claude in Chrome peut se couper en milieu de session
- **Pas d'adversaire IA** — hotseat 2 joueurs uniquement

## Prochaines étapes

- [ ] Valider le workflow sur 2-3 autres jeux de complexité croissante
- [ ] Extraire un fichier skill entièrement générique (supprimer les spécificités quantum-tic-tac-toe)
- [ ] Compléter les templates de prompts avec des exemples réels et testés
- [ ] Documenter l'utilisation de tokens/coûts par phase
- [ ] Ajouter des patterns de tests unitaires pour la logique de jeu BGA
- [ ] Intégration du tutoriel intégré BGA

## Licence

Licence MIT. Voir [LICENSE](LICENSE).

Le jeu *Quantum Tic-Tac-Toe* a été inventé par Allan Goff. Les règles du jeu sont créditées et attribuées ; ce dépôt contient uniquement le code d'implémentation BGA et la méthodologie de développement.
