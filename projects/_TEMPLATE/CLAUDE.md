# <project name>

<one or two sentences: what it is, who it's for, prod or not>

## Stack

- <language / framework, version>
- <database, cache, queue>
- Runs via `docker-compose.dev.yml` (or whatever filename your project uses)

## Commands

Replace service names with the real ones from this project's compose file.

- Bring up the stack: `docker compose -f docker-compose.dev.yml up -d`
- Tear it down: `docker compose -f docker-compose.dev.yml down`
- Composer / pip / npm install: `docker compose -f docker-compose.dev.yml exec app <package-manager> install`
- Tests: `docker compose -f docker-compose.dev.yml exec app <test-runner>`
- Logs: `docker compose -f docker-compose.dev.yml logs -f app`
- Migrations: `docker compose -f docker-compose.dev.yml exec app <migration-command>`

## Secrets

`.env` is local to the project and gitignored. The application reads it at startup.

## Relationships with other projects

- <who calls this service>
- <what this service calls>
- See `shared/architecture.md` for the overall map.
