# Goblin vs Unicorn — public dev wiki

Read-only reference for **Goblin vs Unicorn**, a Godot 4 mobile game blending Puzzle & Dragons-style tile matching with a short, autonomous RTS battle phase. These files are regenerated from the game and its docs, so they show **what the game does today, what it is meant to be, and what is being worked on**.

## Start here
| File | What it is | Read it for |
|---|---|---|
| [`game-snapshot.md`](game-snapshot.md) | Rules as built + current numbers (units, costs, terrain, economy) + open questions | How the game works **today** |
| [`GameDesignDoc.md`](GameDesignDoc.md) | Written design intent (source of truth for how the game *should* work) | Design feedback, vision |
| [`ROADMAP.md`](ROADMAP.md) | Status, build order, bug list, decisions log | What is built, what is next, and why |
| [`game-data.json`](game-data.json) | Raw numbers exported from the game's data files | Exact stats and costs |

## In development
| Folder | What it is |
|---|---|
| [`plans/`](plans) | Step-by-step plans for upcoming or in-progress work (e.g. the foundation pass) |
| [`reviews/`](reviews) | Code and system reviews and their findings |
| [`vision/`](vision) | Digest of earlier design conversations — **unverified prior reasoning, not decisions** |

Tags: **built** / **planned** / **idea** / **cut**. Numbers marked as assumptions are unmeasured guesses, not playtest results. When a doc here disagrees with `game-snapshot.md`, the snapshot (generated from the game) wins for current behavior; `GameDesignDoc.md` wins for intent.
