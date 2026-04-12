# Claude Code + Board Game Arena : Skill de développement BGA

Un fichier skill [Claude Code](https://claude.ai/claude-code) pour développer des jeux sur [Board Game Arena](https://studio.boardgamearena.com) Studio — des règles publiées à un jeu hotseat jouable.

## Démarrage (5 minutes)

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

## Ce que couvre le fichier skill

- Scaffolding du projet (Makefile, structure de répertoires, fichiers de config)
- Patterns du nouveau framework BGA (PHP 8.4, classes d'état, notifications, globales)
- Structure du client JavaScript (modules ES6, gestionnaires d'état)
- Boucle deploy-test automatisée via l'extension Chrome
- Pièges connus de BGA et comment les éviter (commentaires SQL, noms de tables, problèmes de requêtes DB)

Voir `TECHNICAL_NOTES.md` pour les explications détaillées des pièges et de leurs causes.

## Ce que vous devez encore faire vous-même

- Créer le compte BGA Studio et enregistrer votre jeu
- Configurer la clé SSH pour l'accès SFTP (peut prendre jusqu'à 1h pour se propager)
- Quitter manuellement les tables de jeu bloquées quand l'interface BGA le requiert
- Reconnecter l'extension Chrome si elle se déconnecte
- Relire le code généré — c'est une alpha, pas du code prêt pour la production

## Jeux construits avec ce skill

- [Quantum Tic-Tac-Toe](https://github.com/rbellec/BGA_quantum_tic_tac_toe) — mécaniques de superposition quantique, graphe d'intrication, détection de cycles
- [Go On Rasalva](https://github.com/rbellec/Go-On-Rasalva-on-BGA) — placement de tuiles, gestion de ressources

## Article

Un article détaillé sur la méthodologie et les leçons apprises arrive bientôt.

## Licence

Licence MIT. Voir [LICENSE](LICENSE).
