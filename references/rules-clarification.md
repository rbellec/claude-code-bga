# Rules Clarification — BGA Reference

**When to load:** At the very start of a BGA project, before implementation begins. Also load whenever a new ambiguity surfaces during development.

## Why This Matters

A board game rulebook is written for humans who can interpret ambiguities on the fly. A program cannot. Every edge case, implicit assumption, and unclear interaction must be resolved explicitly before it can be coded. Starting implementation with unresolved rules leads to rework and bugs.

This phase should happen **as early as possible** — ideally right after converting the rules to Markdown (Phase 1) and before writing any game logic (Phase 2). However, it must not block implementation entirely: questions can be categorized by impact, and work can proceed on unambiguous parts.

## Process Overview

```
Rules PDF → Markdown conversion → Systematic analysis → Questions document → Author feedback → Update assumptions → Implement
                                                              ↑                                       |
                                                              └── New questions during dev ────────────┘
```

## Step 1 — Convert Rules to Markdown

Create `doc/RULES.md` from the rulebook PDF. This becomes the primary reference for implementation. Only go back to the PDF for graphical information (component illustrations, board layout) that can't be captured in text.

**Why Markdown first:** avoids re-reading the PDF repeatedly (slow, expensive in tokens), makes rules searchable and diffable, and forces a careful first reading that surfaces ambiguities.

## Step 2 — Systematic Analysis

Read through RULES.md with a programmer's mindset. For every rule, ask:
- What are the **inputs** (player choices) and **outputs** (state changes)?
- What happens at **boundaries** (edge of board, empty deck, no valid targets)?
- What are the **interactions** between this rule and others?
- Is the **timing** explicit (before/after other effects)?

## Step 3 — Categorize Questions

Each question gets a **status** and a **category**.

### Statuses

| Status | Meaning |
|--------|---------|
| `OPEN` | Needs author answer — blocks implementation of this specific feature |
| `ASSUMED` | We picked a reasonable default — can implement, but needs confirmation |
| `DEVELOPER` | Needs analysis by the developer (human) — not a rules question per se |
| `CLOSED` | Answered by authors or resolved — record the answer |

### Categories

| Category | Description | Example |
|----------|-------------|---------|
| `RULES-MISSING` | Rule doesn't cover this case at all | "What happens when the deck runs out?" |
| `RULES-AMBIGUOUS` | Rule text can be read multiple ways | "Does 'place' include 'move'?" |
| `RULES-IMPLICIT` | Answer is deductible but not stated | "Can you move off the board edge?" |
| `RULES-VISUAL` | Answer might be in illustrations but not text | "What's the grid size?" |
| `IMPLEMENTATION` | Rules are clear but implementation approach needs a decision | "Should we use Deck component or custom table?" |
| `FEEDBACK` | Suggestion for improving the rulebook (not blocking) | "Adding a diagram would clarify setup" |

## Step 4 — Document Format

Maintain a single `doc/AUTHOR_QUESTIONS.md` file structured for easy communication with authors:

```markdown
# Game Name — Questions pour les auteurs

Brief intro explaining why questions are asked (program can't interpret, every case must be explicit).

## Questions ouvertes (OPEN)

### Q1. [RULES-MISSING] Title of question
**Status:** OPEN
**Ce que dit la règle :** exact quote from rulebook
**Le problème :** why this is ambiguous/missing for implementation
**Ce qui nous aiderait :** what kind of answer we need (yes/no, diagram, sentence)

## Questions avec hypothèse (ASSUMED)

### Q2. [RULES-IMPLICIT] Title of question
**Status:** ASSUMED → [H3] in ASSUMPTIONS.md
**Ce que dit la règle :** exact quote
**Notre hypothèse :** what we assumed and why
**Impact si faux :** what we'd need to change

## Questions fermées (CLOSED)

### Q3. [RULES-AMBIGUOUS] Title of question
**Status:** CLOSED (2025-04-15)
**Réponse :** author's answer
**Impact :** what was changed in the code (if anything)
```

### Key principles for the document:

- **Written for the authors**, not for developers — no code, no jargon
- **Explain "why"** — authors don't know what's obvious or not to a program
- **Structured for scanning** — authors should be able to quickly find open questions
- **Quote the exact rule text** — so authors can see what we're working from
- **Suggest improvements** — helps authors improve their rulebook (category FEEDBACK)

## Step 5 — Maintain During Development

New questions **will** surface during implementation. When they do:

1. Add them to `AUTHOR_QUESTIONS.md` with status OPEN or ASSUMED
2. If ASSUMED, add the corresponding hypothesis to `doc/ASSUMPTIONS.md`
3. If the feature can be implemented with the assumption, proceed
4. If it truly blocks (can't even guess a reasonable default), flag it and work on something else
5. When an author answers, update the status to CLOSED, record the answer and date

## Step 6 — Assumptions File

Maintain `doc/ASSUMPTIONS.md` as a companion:

```markdown
# Game Name — Implementation Assumptions

**[H1] Grid is 6x6** — deduced from illustrations. Ref: Q1 in AUTHOR_QUESTIONS.md
**[H2] Blockers on empty cells only** — assumed from "where you like". Ref: Q5
```

Each assumption has an ID `[Hx]` referenced in both the questions file and in code comments (`// [H3] blocker on free cell only`). When an author confirms or corrects, update both files and the code.

## Anti-patterns

- **Don't block all implementation** waiting for answers — categorize and work on what's clear
- **Don't assume silently** — every assumption must be documented and linked to a question
- **Don't ask authors implementation questions** — they don't know what a "state machine" is; translate to game terms
- **Don't dump a wall of text** — group questions by theme, prioritize, keep each one short
- **Don't forget to close questions** — stale OPEN questions create confusion

## Skill Integration Notes

- This reference should be loaded when the game rules are first provided
- The rules-to-Markdown conversion should happen in Phase 1 (step 1.2, after scaffold download)
- The systematic analysis should happen before Phase 2 (Implementation)
- The SKILL.md should reference this process as "Phase 1.5 — Rules Analysis"
