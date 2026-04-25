# Claude Code + Board Game Arena : Skill de développement BGA

Un skill [Claude Code](https://claude.ai/claude-code) pour développer des jeux sur [Board Game Arena](https://studio.boardgamearena.com) Studio — des règles publiées à un jeu hotseat jouable.

Distribué à la fois comme **plugin Claude Code** et comme **package Vercel `skills`** — choisissez la méthode qui vous convient.

## Installation

### Option A — Plugin Claude Code (recommandé)

Dans Claude Code :

```
/plugin marketplace add rbellec/claude-code-bga
/plugin install board-game-arena@claude-code-bga
```

### Option B — `npx skills`

Depuis votre projet (ou en global avec `-g`) :

```bash
npx skills add rbellec/claude-code-bga -a claude-code
```

### Option C — Copie manuelle

```bash
git clone https://github.com/rbellec/claude-code-bga.git
mkdir -p ~/.claude/skills
ln -s "$PWD/claude-code-bga/skills/board-game-arena" ~/.claude/skills/board-game-arena
```

## Démarrage

1. **Installez [Claude in Chrome](https://chrome.google.com/webstore/detail/claude-in-chrome)** — nécessaire pour la boucle de test automatisée sur BGA Studio.
2. **Ayez votre compte BGA Studio prêt** — jeu enregistré, clé SSH configurée ([docs BGA](https://en.doc.boardgamearena.com/Studio)).
3. **Démarrez Claude Code dans votre répertoire de projet** et demandez-lui d'implémenter votre jeu :
   ```
   Je veux implémenter [NOM DU JEU] sur BGA Studio.
   Voici les règles : [collez les règles ou un lien vers le livre de règles]
   Mon nom d'utilisateur BGA est [USERNAME] et le nom du jeu est [GAMENAME].
   ```

Claude Code utilisera le skill pour gérer la configuration BGA (Makefile, fichiers de config, machine à états, workflow de déploiement) et l'extension navigateur pour tester directement sur BGA Studio.

### Optionnel : autoriser git/make/scp sans confirmation

Le skill commite automatiquement à chaque étape et déploie via `scp`/`make`. Pour éviter les demandes de confirmation, ajoutez dans votre `.claude/settings.local.json` (non commité) :

```json
{
  "permissions": {
    "allow": ["Bash(git *)", "Bash(make *)", "Bash(scp *)"]
  }
}
```

## Ce que couvre le skill

- Scaffolding du projet (Makefile, structure de répertoires, fichiers de config)
- Patterns du nouveau framework BGA (PHP 8.4, classes d'état, notifications, globales)
- Structure du client JavaScript (modules ES6, gestionnaires d'état)
- **Références des librairies BGA** — guides détaillés pour Deck, BgaCards, Stock, Counter, Scrollmap, et 10 autres librairies du framework BGA, chargées à la demande pour économiser le contexte (voir [`skills/board-game-arena/references/`](skills/board-game-arena/references/))
- Boucle deploy-test automatisée via l'extension Chrome
- Pièges connus de BGA et comment les éviter (commentaires SQL, noms de tables, problèmes de requêtes DB)

Voir [`TECHNICAL_NOTES.md`](TECHNICAL_NOTES.md) pour le *pourquoi* derrière chaque règle.

## Ce que vous devez encore faire vous-même

- Créer le compte BGA Studio et enregistrer votre jeu
- Configurer la clé SSH pour l'accès SFTP (peut prendre jusqu'à 1h pour se propager)
- Quitter manuellement les tables de jeu bloquées quand l'interface BGA le requiert
- Reconnecter l'extension Chrome si elle se déconnecte
- Relire le code généré — c'est une alpha, pas du code prêt pour la production

## Jeux construits avec ce skill

- [Visite Royale / Royal Visit](https://github.com/rbellec/bga_visite_royale) — jeu de cartes, mécaniques de tir à la corde
- [Quantum Tic-Tac-Toe](https://github.com/rbellec/BGA_quantum_tic_tac_toe) — mécaniques de superposition quantique, graphe d'intrication, détection de cycles
- [Go On Rasalva](https://github.com/rbellec/Go-On-Rasalva-on-BGA) — placement de tuiles, gestion de ressources

## Article

Un article détaillé sur la méthodologie et les leçons apprises arrive bientôt.

## Licence

Licence MIT. Voir [LICENSE](LICENSE).
