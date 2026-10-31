# Makefile Template — BGA Reference

**When to use:** Project setup (Phase 1) — copy and adapt once, then forget.

## Template

```makefile
BGA_USER   := USER
BGA_HOST   := 1.studio.boardgamearena.com
BGA_PORT   := 2022
BGA_GAME   := GAMENAME
BGA_REMOTE := $(BGA_GAME)
BGA_KEY    := $(HOME)/.ssh/id_rsa_bga
BGA_SCP    := scp -i $(BGA_KEY) -P $(BGA_PORT) -o IdentitiesOnly=yes

# Config files: keep the extension the scaffold shipped (.jsonc on recent scaffolds, .json/.inc.php on older ones)
DEPLOY_ROOT := dbmodel.sql $(wildcard gameinfos.jsonc gameinfos.inc.php stats.json stats.jsonc gameoptions.json gameoptions.jsonc gamepreferences.json gamepreferences.jsonc)
DEPLOY_PHP  := modules/php/Game.php modules/php/material.inc.php
DEPLOY_JS   := modules/js/Game.js

check:
	@php -l modules/php/Game.php
	@php -l modules/php/material.inc.php
	@for f in modules/php/States/*.php; do php -l "$$f" || exit 1; done
	@php -l gameinfos.inc.php
	@echo "✓ PHP OK"

deploy: check
	$(BGA_SCP) $(DEPLOY_ROOT) $(BGA_USER)@$(BGA_HOST):$(BGA_REMOTE)/
	$(BGA_SCP) GAMENAME.css $(BGA_USER)@$(BGA_HOST):$(BGA_REMOTE)/GAMENAME.css
	$(BGA_SCP) $(DEPLOY_PHP) $(BGA_USER)@$(BGA_HOST):$(BGA_REMOTE)/modules/php/
	$(BGA_SCP) modules/php/States/*.php $(BGA_USER)@$(BGA_HOST):$(BGA_REMOTE)/modules/php/States/
	$(BGA_SCP) $(DEPLOY_JS) $(BGA_USER)@$(BGA_HOST):$(BGA_REMOTE)/modules/js/
	$(BGA_SCP) img/* $(BGA_USER)@$(BGA_HOST):$(BGA_REMOTE)/img/
	@echo "✓ Deployed"
```

## Notes

- BGA Studio is **SFTP-only** (no SSH shell) — rsync does not work
- Use `scp` with explicit file lists
- Replace `USER`, `GAMENAME` with actual values
- `BGA_KEY`: adapt to the key registered in BGA Studio — existing projects may use another path (check their Makefile); never read the key file itself
- Keep `States/*.php` OUT of `DEPLOY_PHP` — the deploy target already sends them to `modules/php/States/`; listing them in `DEPLOY_PHP` would also copy them flat into `modules/php/`
- Add any additional PHP files under `modules/php/` to `DEPLOY_PHP` as needed
