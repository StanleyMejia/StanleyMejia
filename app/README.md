# Loreforge

Self-hosted worldbuilding and manuscript workspace, in the spirit of Campfire Writing.
One Node process, one SQLite file, no external services.

## Features

- **Worlds** – any number of separate universes, each with its own modules.
- **Modules** – Characters, Locations, Maps, Encyclopedia, Magic, Items, Species, Cultures,
  Factions, Religions, Philosophies, Systems and Research out of the box. Add your own
  (ships, spells, languages…) with custom panel templates.
- **Panels** – every element is a stack of panels you can add, rename, reorder and remove:
  - _Attributes_ – labelled fields (text, long text, number, choice, link to another element)
  - _Text_ – Markdown with `[[wiki links]]` and autocomplete
  - _List_ – organised named items (personality traits, costs, limitations…)
  - _Statistics_ – numeric values rendered as bars
  - _Links_ – curated connections to other elements with notes
  - _Images_ – galleries by URL
- **Wiki links & backlinks** – write `[[Name]]` anywhere; every element shows what mentions it.
  Unknown names become red links that create the element with one click.
- **Relationships** – labelled, directional edges (`mentor of` / `student of`) plus a
  force-directed relationship map.
- **Timeline** – events with free-form date labels, eras and a numeric sort key, so any
  calendar system works.
- **Manuscripts** – books → chapters with a distraction-free Markdown editor, status
  tracking, word counts, focus mode and Ctrl+S.
- **Search & tags** across the whole world.
- **Export** the entire world as a single JSON file.

## Run it

### Docker (recommended)

```bash
cd app
docker compose up -d --build
# → http://localhost:3000
```

The database lives in `./data/loreforge.db` (bind-mounted to `/data`). Back it up by copying
that directory or by using the in-app JSON export.

Behind a reverse proxy, set `ORIGIN` to the public URL (e.g. `https://lore.example.lan`) and
uncomment `PROTOCOL_HEADER` / `HOST_HEADER` in `docker-compose.yml`.

### Bare metal

```bash
cd app
cp .env.example .env      # adjust DATABASE_URL / ORIGIN / PORT
pnpm install
pnpm build
node build                # production server on :3000
```

Development server with hot reload: `pnpm dev`.

## Architecture

```
SvelteKit (Svelte 5, TypeScript, Tailwind v4)
└─ adapter-node → single Node process
   └─ better-sqlite3 (WAL) via Drizzle ORM
      └─ migrations in ./drizzle, applied automatically at startup
```

| Path                                    | Purpose                                                                                              |
| --------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `src/lib/server/db/schema.ts`           | Drizzle schema: worlds, element_types, elements, relationships, links, events, manuscripts, chapters |
| `src/lib/server/repo/*`                 | All queries. Routes never touch the DB directly.                                                     |
| `src/lib/server/panels.ts`              | Validation of panel JSON coming from the browser                                                     |
| `src/lib/types.ts`                      | Panel / field type definitions shared by client and server                                           |
| `src/lib/defaults.ts`                   | Default modules and their panel templates                                                            |
| `src/lib/markdown.ts`                   | Markdown renderer with `[[wiki link]]` extension and HTML sanitising                                 |
| `src/lib/components/PanelEditor.svelte` | The panel-based element editor (also used for type templates)                                        |
| `src/routes/w/[world]/…`                | All world-scoped pages; server load + form actions per route                                         |

### Data model

- `element_types.panels` holds the **template** for a module. Creating an element copies it.
- `elements.panels` holds the element's own panels as JSON, so each element can diverge from
  its template (extra attributes, extra panels) without schema changes.
- `links` is a derived table of `[[wiki links]]` extracted on every save. It powers backlinks
  and the "mentions" edges in the relationship map.
- `relationships` are explicit, user-labelled edges.

### Schema changes

```bash
# edit src/lib/server/db/schema.ts, then:
pnpm db:generate     # writes a new SQL migration into ./drizzle
```

Migrations run on boot, so a container restart upgrades the database.

## Roadmap ideas

- Image uploads to local storage instead of URLs
- Full-text search (SQLite FTS5)
- Interactive maps with pins linked to locations
- Multi-user auth (OIDC) for shared worlds
- JSON import to restore an export
