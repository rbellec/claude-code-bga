# Optional: PHP Code Quality Loop

> **Optional reference for users comfortable with PHP tooling.** Skip this section if you're not. The skill works fine without these tools — they add a local "PR review" loop on top of the BGA framework code.

This guide sets up four PHP quality tools (PHP-CS-Fixer, Rector, PHPStan, PHPMD) to run as a `make audit` target. Read-only by default, fast (~10 s for a small project), and installable once for all your BGA projects.

## Why

`make check` (PHP syntax lint) catches *parse errors* but nothing else. The four tools below catch:

- **PHP-CS-Fixer** — formatting drift (preset `@PER-CS2x0` + `@PHP8x4Migration`)
- **Rector** — mechanical simplifications (`empty($x)` → `$x === []`, dead code, code-quality rulesets)
- **PHPStan** at level max — type inference, undefined methods, mixed-type leaks
- **PHPMD** — cyclomatic & NPath complexity, unused parameters, design smells

They are layered from cheapest (cosmetic auto-fix) to most expensive (semantic analysis). Together they form a local "PR review" loop you can run before each commit.

## One-time install: shared toolbox at `~/.bga-tools/`

Install once, used by every BGA project. Keeps tools isolated from your system Composer global, no per-project re-install.

```bash
# Composer (skip if already installed)
brew install composer

# Toolbox
mkdir -p ~/.bga-tools
cd ~/.bga-tools
composer init --no-interaction --name="yourname/bga-tools" --type=project
composer require --dev \
  friendsofphp/php-cs-fixer \
  phpstan/phpstan \
  rector/rector \
  phpmd/phpmd

# Add to ~/.zshrc (or ~/.bashrc)
echo 'export PATH="$HOME/.bga-tools/vendor/bin:$PATH"' >> ~/.zshrc
exec $SHELL  # reload shell so the PATH change takes effect

# Verify
php-cs-fixer --version && phpstan --version && rector --version && phpmd --version
```

**Updates later**: `cd ~/.bga-tools && composer update`.

### Known issue — PHPMD on PHP 8.5

PHPMD 2.15.0 emits deprecation warnings on PHP 8.5 (PDepend has not yet shipped a release without `Implicitly marking parameter as nullable` constructs). The output is functional but noisy. Workaround: invoke PHPMD via `php -d error_reporting='E_ALL & ~E_DEPRECATED & ~E_USER_DEPRECATED'` (already wired in the Makefile snippet below).

## Per-project setup

Drop these five files at the root of your BGA project. Skill assumes the standard `modules/php/` + `gameinfos.inc.php` layout.

### `.php-cs-fixer.php`

```php
<?php

declare(strict_types=1);

$finder = PhpCsFixer\Finder::create()
    ->in(__DIR__ . '/modules/php')
    ->append([__DIR__ . '/gameinfos.inc.php'])
    ->name('*.php')
    ->notPath('bga_initial_code_template');

return (new PhpCsFixer\Config())
    ->setRiskyAllowed(true)
    ->setRules([
        '@PER-CS2x0' => true,
        '@PER-CS2x0:risky' => true,
        '@PHP8x4Migration' => true,
        'declare_strict_types' => true,
        'array_syntax' => ['syntax' => 'short'],
        'no_unused_imports' => true,
        'ordered_imports' => ['sort_algorithm' => 'alpha'],
        'single_quote' => true,
        'trailing_comma_in_multiline' => ['elements' => ['arrays', 'arguments', 'parameters']],
        'no_trailing_whitespace' => true,
        'no_whitespace_in_blank_line' => true,
    ])
    ->setFinder($finder)
    ->setCacheFile(__DIR__ . '/.php-cs-fixer.cache');
```

### `phpstan.neon`

```neon
includes:
    - phpstan-baseline.neon

parameters:
    level: max
    paths:
        - modules/php
        - gameinfos.inc.php
    excludePaths:
        - bga_initial_code_template
    bootstrapFiles:
        - stubs/bga-framework.stub.php
    treatPhpDocTypesAsCertain: false
    reportUnmatchedIgnoredErrors: false
    ignoreErrors:
        - identifier: property.notFound
          path: modules/php/*
        - '#^Class .*Deck.*not found#'
```

You'll generate `phpstan-baseline.neon` once on the first run (see Workflow below). Commit it.

### `stubs/bga-framework.stub.php`

The BGA framework is loaded server-side, so PHPStan can't resolve `\Bga\GameFramework\*` symbols by itself. A small stub file tells PHPStan what those classes look like. Start minimal — extend as you use new framework symbols.

```php
<?php

declare(strict_types=1);

namespace Bga\GameFramework {

    /**
     * @property-read \Bga\GameFramework\Components\Globals $globals
     * @property-read \Bga\GameFramework\Components\TableStats $tableStats
     * @property-read \Bga\GameFramework\Components\PlayerStats $playerStats
     * @property-read \Bga\GameFramework\Components\TableOptions $tableOptions
     */
    class BgaContext {}

    abstract class Table
    {
        /** @var BgaContext */
        protected $bga;
        public function __construct() {}
        protected function initTable(): void {}
        protected function setupNewGame(array $players, array $options = []): mixed { return null; }
        public function getAllDatas(int $currentPlayerId): array { return []; }
        public function getGameProgression(): int { return 0; }

        protected function getActivePlayerId(): int { return 0; }
        protected function getCurrentPlayerId(): int { return 0; }
        protected function getGameinfos(): array { return []; }
        protected function getPlayersBasicInfos(): array { return []; }
        protected function getPlayersNumber(): int { return 0; }

        /** @return array<string,mixed>|null */
        protected function getObjectFromDB(string $sql): ?array { return null; }
        /** @return array<int|string,array<string,mixed>|mixed> */
        protected function getCollectionFromDb(string $sql): array { return []; }
        /** @return list<array<string,mixed>> */
        protected function getObjectListFromDB(string $sql): array { return []; }
        protected function getUniqueValueFromDB(string $sql): mixed { return null; }
        protected function DbQuery(string $sql): void {}

        protected function notifyAllPlayers(string $type, string $message, array $args = []): void {}
        protected function notifyPlayer(int $playerId, string $type, string $message, array $args = []): void {}
        protected function giveExtraTime(int $playerId, ?int $specificTime = null): void {}

        public function gamestate(): \Bga\GameFramework\GameStateController { return new \Bga\GameFramework\GameStateController(); }
    }

    class UserException extends \Exception {}

    class StateType
    {
        public const GAME = 'game';
        public const ACTIVE_PLAYER = 'activePlayer';
        public const MULTIPLE_ACTIVE_PLAYER = 'multipleActivePlayer';
        public const MANAGER = 'manager';
    }

    class GameStateController
    {
        public function nextState(string $transition): void {}
        public function jumpToState(int $stateId): void {}
        public function setActivePlayer(int $playerId): void {}
        public function changeActivePlayer(int $playerId): void {}
    }
}

namespace Bga\GameFramework\States {

    abstract class GameState
    {
        public function __construct() {}
        public function getId(): int { return 0; }
        public function getName(): string { return ''; }
        public function getType(): string { return ''; }
        public function onEnteringState(int $activePlayerId): mixed { return null; }
        public function getArgs(): array { return []; }
        public function getPossibleActions(): array { return []; }
    }

    #[\Attribute(\Attribute::TARGET_METHOD)]
    class PossibleAction
    {
        public function __construct(public string $name = '') {}
    }
}

namespace Bga\GameFramework\Components {

    class Globals
    {
        public function set(string $key, mixed $value): void {}
        public function get(string $key, mixed $default = null): mixed { return $default; }
        public function inc(string $key, int $delta = 1): int { return 0; }
        public function has(string $key): bool { return false; }
    }

    class TableStats
    {
        public function init(string $name, mixed $value): void {}
        public function set(string $name, mixed $value): void {}
        public function get(string $name): mixed { return null; }
        public function inc(string $name, int $delta = 1): void {}
    }

    class PlayerStats
    {
        public function init(string $name, mixed $value, int $playerId): void {}
        public function set(string $name, mixed $value, int $playerId): void {}
        public function get(string $name, int $playerId): mixed { return null; }
        public function inc(string $name, int $delta = 1, int $playerId = 0): void {}
    }

    class TableOptions
    {
        public function get(string $key): mixed { return null; }
    }
}
```

### `rector.php`

```php
<?php

declare(strict_types=1);

use Rector\Config\RectorConfig;
use Rector\DeadCode\Rector\ClassMethod\RemoveParentDelegatingConstructorRector;
use Rector\DeadCode\Rector\StaticCall\RemoveParentCallWithoutParentRector;

return RectorConfig::configure()
    ->withPaths([
        __DIR__ . '/modules/php',
        __DIR__ . '/gameinfos.inc.php',
    ])
    ->withSkip([
        __DIR__ . '/bga_initial_code_template',
        __DIR__ . '/stubs',

        // === BGA framework safety: MUST stay skipped ===
        // The PHPStan stubs declare framework methods (Table::__construct, etc.)
        // with empty bodies. Rector reads stubs as truth and concludes that
        // calls like parent::__construct() delegate to a no-op, then deletes
        // them. In production, the real Bga\GameFramework\Table::__construct()
        // initializes $bga, $gamestate, deckFactory, etc. Removing the parent
        // call leaves those null and breaks game launch with no explicit error.
        //
        // See "Known incidents" section below for the full incident report.
        RemoveParentDelegatingConstructorRector::class,
        RemoveParentCallWithoutParentRector::class,
    ])
    ->withPhpSets(php84: true)
    ->withPreparedSets(
        codeQuality: true,
        deadCode: true,
        codingStyle: false,
        typeDeclarations: false,  // disabled in first pass — too aggressive on framework signatures
        privatization: false,
        naming: false,
        instanceOf: true,
        earlyReturn: true,
        strictBooleans: false,
    )
    ->withImportNames(removeUnusedImports: true);
```

### `phpmd.xml`

```xml
<?xml version="1.0"?>
<ruleset name="BGA PHPMD ruleset"
         xmlns="http://pmd.sf.net/ruleset/1.0.0"
         xsi:noNamespaceSchemaLocation="http://pmd.sf.net/ruleset_xml_schema.xsd"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

    <exclude-pattern>*/bga_initial_code_template/*</exclude-pattern>
    <exclude-pattern>*/stubs/*</exclude-pattern>

    <rule ref="rulesets/cleancode.xml">
        <exclude name="StaticAccess"/>
        <exclude name="ElseExpression"/>
    </rule>

    <rule ref="rulesets/codesize.xml"/>

    <rule ref="rulesets/design.xml">
        <exclude name="CouplingBetweenObjects"/>
    </rule>

    <rule ref="rulesets/naming.xml">
        <!-- BGA conventions: short ids ($id, $pid), snake_case from DB columns -->
        <exclude name="ShortVariable"/>
        <exclude name="LongVariable"/>
    </rule>

    <rule ref="rulesets/unusedcode.xml"/>
</ruleset>
```

### Makefile additions

Add these targets to your Makefile (keeping `check` and `deploy` unchanged). The PATH export at the top makes the targets work regardless of shell init state.

```makefile
# Quality tools toolbox (shared across BGA projects)
export PATH := $(HOME)/.bga-tools/vendor/bin:$(PATH)

# ... existing BGA_USER, BGA_HOST, etc.

# Auto-apply: CS-Fixer + Rector. Use after reviewing fix-dry.
fix:
	@echo "→ PHP-CS-Fixer (apply)"
	@php-cs-fixer fix
	@echo "→ Rector (apply)"
	@rector process

# Dry-run, no file changes.
fix-dry:
	@echo "→ PHP-CS-Fixer (dry-run)"
	@php-cs-fixer fix --dry-run --diff || true
	@echo "→ Rector (dry-run)"
	@rector process --dry-run || true

# Static analysis — never modifies files.
analyze:
	@echo "→ PHPStan (level max + baseline)"
	@phpstan analyse --memory-limit=512M --no-progress || true
	@echo "→ PHPMD"
	@php -d error_reporting='E_ALL & ~E_DEPRECATED & ~E_USER_DEPRECATED' \
	     -d display_errors=stderr \
	     $$HOME/.bga-tools/vendor/phpmd/phpmd/src/bin/phpmd \
	     modules/php,gameinfos.inc.php text phpmd.xml 2>/dev/null || true

# Read-only "PR review" — safe to run anytime, including before commit.
audit: check fix-dry analyze
	@echo "✓ Audit complete (read-only)"
```

**Important**: `deploy` does **not** depend on `audit`. The audit must never block a deploy in mid-session — quality is advisory, not gating. The skill's deploy-test rhythm stays unblocked.

## Workflow

### Initial bootstrap (once per project)

```bash
# 1. Drop in the 5 config files above + Makefile additions.

# 2. Generate the PHPStan baseline to freeze legacy framework noise.
phpstan analyse --memory-limit=512M --no-progress --generate-baseline=phpstan-baseline.neon
git add phpstan-baseline.neon

# 3. Run the audit to confirm clean output.
make audit
```

The first PHPStan run will produce hundreds of errors (framework symbols typed as `mixed`, protected method access from State classes, missing classes like `Deck`). The baseline freezes them. From that point on, `make analyze` only reports **new** errors introduced after the baseline.

### Day-to-day

```bash
make audit          # before commit, read-only review
make fix-dry        # ⚠️ ALWAYS read this diff before make fix (see Known incidents)
make fix            # apply CS-Fixer + Rector when fix-dry's diff looks safe
make check && make deploy
# 🔴 ALWAYS verify game launch on Studio after deploying any 'make fix' result.
#    The local audit cannot detect framework-init regressions.
```

**Mandatory post-deploy step**: open the game on Studio and start a new table. The audit suite is silent on regressions that only surface at runtime — the only safety net is your eyes on the dry-run diff and a real game launch.

### Optional: pre-commit hook

If you want `make audit` to run automatically before each commit:

```bash
cat > .git/hooks/pre-commit <<'HOOK'
#!/usr/bin/env bash
set -e
exec make audit
HOOK
chmod +x .git/hooks/pre-commit
```

## Tuning notes

- **Don't enable Rector's `typeDeclarations` set in the first pass.** It rewrites method signatures aggressively, which can drift from the BGA framework's expected contracts. Re-enable it only after your signatures are stable and you have stub coverage.
- **PHPMD's `controversial` ruleset** is not included. It enforces things like `CamelCasePropertyName`, which fights with BGA's snake_case DB column conventions.
- **`UnusedFormalParameter` from PHPMD** will flag framework-injected parameters like `$playerId` in `actX($playerId)` and `$activePlayerId` in `onEnteringState`. These are framework contracts, not real bugs. Tag with `@SuppressWarnings("PHPMD.UnusedFormalParameter")` per-method if you want a clean report, or accept them as background noise.
- **PHPStan errors after the baseline**: only meaningful for *new* code. If you refactor old code, regenerate the baseline (`--generate-baseline=phpstan-baseline.neon`).

## When the audit reveals real findings

A first run on a healthy BGA project usually surfaces ~20–40 actionable items split across:

- 5–15 cosmetic (CS-Fixer auto-apply)
- 5–15 mechanical simplifications (Rector auto-apply)
- 0–5 PHPStan semantic issues (after baseline) — **read these carefully**
- 5–10 PHPMD complexity findings on central classes (`Game.php`, big state methods)

Treat the auto-apply ones as a single "tooling commit" reviewed end-to-end. Treat PHPStan/PHPMD findings as a separate refactoring backlog — they're often worth addressing one method at a time, not in a sweep.

## ⚠️ Known incidents

### Rector eats `parent::__construct()` on Game.php (Duelly, 2026-04-26)

**Symptom**: After `make fix` + deploy, the game would not launch on BGA Studio. No explicit error in the logs — the table never reached the first state.

**Root cause**: Two interacting issues, both in `modules/php/Game.php`.

1. **`parent::__construct()` was removed by Rector.** The PHPStan stub declares
   `Bga\GameFramework\Table::__construct() {}` with an empty body. Rector's
   `RemoveParentDelegatingConstructorRector` (in the `deadCode` set) reads the
   stub as truth, concludes the parent call is a delegation to a no-op, and
   deletes it. In production, the **real** parent constructor wires
   `$this->bga`, `$this->gamestate`, the deck factory, and other framework
   plumbing. Without it, every later `$this->bga->...` access hits null.

2. **`public $bga;` and `public $gamestate;` were added as untyped child properties.**
   These properties are *injected by the framework* on the parent `Table` class.
   Declaring them in the child class shadows the inherited slot, leaving them
   `null` after construction.

**Why the audit didn't catch it**: PHPStan and PHPMD remained `[OK]`. The code
was syntactically and statically valid — the regression only manifests at runtime
inside the BGA framework's bootstrap. **The audit suite has no runtime
verification.**

**Fix and prevention**:
- The `rector.php` template in this guide already includes
  `RemoveParentDelegatingConstructorRector` and `RemoveParentCallWithoutParentRector`
  in `withSkip()` — keep them there on every BGA project.
- Restore `parent::__construct()` in `Game::__construct()` if Rector ever drops it.
- Never declare `public $bga;` or `public $gamestate;` in your `Game` class —
  they are framework-injected on the parent.

**Pre-flight checklist before any `make fix`**:

1. Run `make fix-dry` and read the Rector section of the diff line by line.
2. **Reject any line that removes a `parent::method()` call** in classes
   extending `Bga\GameFramework\Table` or `Bga\GameFramework\States\GameState`.
3. **Reject any line that adds an untyped property declaration on Game.php** —
   especially `$bga`, `$gamestate`, `$cards`, or anything resembling a
   framework-injected name.
4. Apply `make fix`, run `make check`, then deploy.
5. **Open the game on Studio and start a new table.** This is the only check
   that catches framework-init regressions.

The same trap applies to any rule in the `deadCode` set that touches calls
into stub methods. If you encounter another `make fix` that breaks game launch,
inspect the Rector diff for similar shape: a `-` line that removes a call into
a framework class.
