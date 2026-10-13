# Glossary

Domain vocabulary for the DnD lobby app, settled at wayfinder charting. This file is a glossary only — the v1 spec lives in `docs/spec-v1.md` when it exists.

## Terms

- **Lobby** — a persistent campaign container: named, with a roster of Players, exactly one DM, shared Lobby notes, and visible on the lobby page. Created by any Player; the DM can delete it. The word "campaign" is an alias for Lobby, not a separate concept.
- **Player** — a human user. Identified within the app by a chosen display name bound to a single browser (a per-browser token). There are no accounts: two browsers are two Players, and identities are never merged across devices.
- **Character** — the persona a Player plays in a Lobby. Exactly one Character per Player per Lobby. A Character owns a Character sheet.
- **Character sheet** — the structured-but-freeform data describing a Character: identity block, ability scores with derived modifiers, combat block, skills with derived modifiers, features & traits, inventory/equipment, spells, and Character notes. The app computes the trivial derivations; it does not enforce DnD rules and ships no content database.
- **DM (Dungeon Master)** — a role within a Lobby held by exactly one Player at a time. The Lobby's creator starts as DM; the current DM can transfer the role to any roster member. DM *views* are a phase-2 feature; v1 only records the role.
- **Lobby notes** — notes shared by the whole Lobby (recap, house rules); any member sees them.
- **Character notes** — notes private to one Character and its Player.
- **Session** — deliberately **not** a domain entity. "Starting a session" means the DM using the DM role in the Lobby.
- **Visible lobby** — a Lobby listed on the lobby page, joinable by any Player without approval.