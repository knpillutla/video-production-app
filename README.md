# AI Music Video Studio

One application for reusable artists and multilingual music, cinematic videos, sessions, concerts, and release operations. This repository implements the **Phases 1–7 local studio: identities, production formats, multilingual lyrics, song/audio, direction, generation, editing, mastering, final exports and YouTube operations**, plus initial multi-user security. Lyrics adapters support OpenAI (default) and Gemini; music adapters support MusicAPI.ai (default), ElevenLabs and Stable Audio. Paid generation requires private configuration, a saved budget estimate, budget and explicit approval. YouTube metadata, thumbnails, clips, authorized publishing and supported analytics are implemented locally; live Google integration requires authorized account verification. See [Phase 7](docs/phase7.md). No paid API calls were made during verification.

## Start on Windows

Install Docker Desktop with Linux containers and PowerShell 7. From this directory:

```powershell
pwsh -File .\run-local.ps1
```

The script creates a git-ignored `.env` with random local credentials and starts the stack. Find `MUSICSTUDIO_API_TOKEN` in that file and enter its value in the UI.

- [Studio UI](http://localhost:3000)
- [API documentation](http://localhost:8000/docs)
- [Readiness](http://localhost:8000/api/v1/health/ready)

Or copy `.env.example` to `.env`, replace both example secrets with random **alphanumeric** values, then run the specified default:

```powershell
docker compose up -d --build
```

With no `.env`, Compose works with explicitly local development credentials (`local-development-only` for the UI). Only ports 3000 and 8000 are bound, on loopback. PostgreSQL and Redis have no host ports. These development defaults must never be exposed to a network. Local mode provides one development owner. Google mode adds workspace membership, server sessions and tenant-aware authorization; see [security setup and limits](docs/security.md).

## Use the shared CLI

The CLI is installed in the API image and calls the same REST endpoints as React:

```powershell
docker compose exec api musicstudio doctor --json
docker compose exec api musicstudio performers create --type solo --name "Telugu Melody Star" --language te --json
docker compose exec api musicstudio performers create --type band --name "Telugu Melodies" --language te --members 7 --json
docker compose exec api musicstudio performers list --json
docker compose exec api musicstudio storage verify --queue generation --json
docker compose exec api musicstudio jobs watch JOB_ID --json
```

For a host installation with Python 3.12+: `pip install -e ./backend`. Set `MUSICSTUDIO_API_URL=http://localhost:8000` and `MUSICSTUDIO_API_TOKEN` to your `.env` token. The Python package intentionally does not load `.env` automatically. Google users should use `musicstudio auth login`; service credentials can be injected with `MUSICSTUDIO_SERVICE_TOKEN`. CLI exits: 0 success, 1 API/job failure, 2 watch timeout or invalid arguments. Language accepts repeated `--language` flags. Culture and region remain independent of language.

## Storage and persistence

`MUSICSTUDIO_MEDIA_PATH=./media` binds the project's Windows `media` folder to `/media` in API and worker containers. To use another Windows folder, set e.g. `MUSICSTUDIO_MEDIA_PATH=C:/Musicstudio/media` in `.env` and create that folder before starting. PostgreSQL uses the named `musicstudio_postgres-data` volume. Redis uses a persistent append-only volume. Media bytes never enter PostgreSQL.

Storage checks create `media/workspaces/WORKSPACE_UUID/checks/JOB_UUID.txt`, save SHA-256 and provenance in PostgreSQL, and expose an authenticated, integrity-checked download. Logical keys are provider independent; traversal and conflicting overwrites are rejected. The immutable write uses hard links: use a local NTFS/Linux filesystem supporting hard links, not an unsupported network filesystem. Storage paths must be controlled by the studio administrator.

On Linux, grant container UID/GID 10001 write access to the media folder (for example using a directory ACL) before startup. Do not make an existing media library world-writable. Windows Docker Desktop bind mounts support the tested local path.

`docker compose restart` preserves data. `docker compose down` also preserves named volumes. **Do not use `down -v` unless you intend to erase the database.** Back up PostgreSQL with `pg_dump` and back up media separately. Restoring only one side is insufficient. Changing POSTGRES_PASSWORD in `.env` does not rotate the password of an initialized database.

## Tests

No host Python or Node installation is needed:

```powershell
docker compose --profile test run --build --rm test
docker compose --profile test run --build --rm web-test
docker compose --profile test run --build --rm e2e
pwsh -File .\scripts\verify-local.ps1
```

Backend tests use a separate PostgreSQL 17 database named `musicstudio_test`, isolated from application data, and refuse any other database name. They execute migrations and check model/migration drift. Browser tests exercise the running application and intentionally create labeled E2E artists and test assets; they never erase user records. Screenshots are saved in `test-results`. The verification script creates a uniquely named solo performer through the CLI, checks the API, verifies all worker queues, restarts services, and confirms data and media persist. It leaves the verification records in place.

## Development

React/TypeScript/Vite/Tailwind, shadcn-style Radix primitives, React Router, TanStack Query, and Zustand live under `web/`. Routes are lazy-loaded. Lists are server-paginated (24 records), artist search is debounced, and jobs refresh every three seconds. Event streaming and arbitrary media upload are later work. The current library contains small verification artifacts, not video originals.

Python/FastAPI/Pydantic/SQLAlchemy/Alembic/Typer/Celery live under `backend/`. API and workers are different image targets. FFmpeg/FFprobe are installed only in the worker image. A dedicated migration container must exit successfully before application services start.

See [Google authentication, workspace ownership and custom instructions](docs/security.md), [architecture and phase gates](docs/architecture.md), [operations](docs/operations.md), and [executed verification](docs/verification.md).

This machine's verified configuration uses API port **8001** because Docker could not bind 8000. The UI remains on port 3000. Set MUSICSTUDIO_API_PORT / MUSICSTUDIO_WEB_PORT in .env to override host ports. Container networking remains unchanged.

A GitHub Actions workflow reference is available at scripts/foundation-ci.example.yml. It has not run on hosted CI; copy it into .github/workflows when the repository is hosted there.


## Security and custom instructions

Google sign-in, workspace membership, personal/team workspace creation, role management, revocable sessions, invitation acceptance, and a tenant-aware media/worker path are implemented. **Users & workspace** manages collaborators, sessions and CLI pairing. **Creative instructions** saves optional Unicode direction, reusable drafts/presets and artist prompt history. See [Google setup and limits](docs/security.md).

Existing records from before tenant migration are preserved but quarantined until their authorized owner is explicitly confirmed by an operator. They are never assigned to the first Google account. Do not run the legacy-assignment command until ownership is known.

No real Google, Azure or YouTube account integration has been verified here. The full media production workflow, project-level controls and cloud deployment remain incomplete. See [Phase 2 usage and limitations](docs/phase2.md) for identity approval, 13 production formats, Look Bible versions and continuity review.

## Identity and production planning

Open **Artists & bands → Manage identity** to configure member roles, upload permitted references and approve an immutable identity version. Then open **Productions** to lock that identity into a format-specific Look Bible. Creative planning runs locally; no paid provider is called. See [Phase 2](docs/phase2.md).



## Production direction (Phase 4)

Open a production → **Open direction studio** to create setlists, storyboards, stages and multi-camera shot plans. Approve the Look Bible first. Plans and reusable stages retain version history and explicit approvals. See [Phase 4 guide](docs/phase4.md).

Phase 3 includes the Lyrics studio, licensed song imports, background audio analysis, previews, approvals, provider generation requests and cost reconciliation. Phase 4 can pin approved masters and analysis revisions; unlinked timing remains provisional. See [provider configuration](docs/provider-configuration.md) for independent lyrics/image/video routing.


## Lyrics and music (Phase 3)

Use **Lyrics studio**, **Song library** and **Providers** in the sidebar. See the [Phase 3 guide](docs/phase3.md) and [private provider configuration](docs/provider-configuration.md). MusicAPI.ai is the default music provider; OpenAI remains the default for lyrics. Keys stay in the ignored `providers.local.json` file and all generation routes ship disabled.

## Phase 5 media and editing

Production projects now open a private media library and proxy timeline editor. Import recordings with release evidence, generate approved visual takes through configurable OpenAI/fal adapters, review media, then save versioned multitrack edits and timecoded comments. Background review renders preserve originals and audio masters. Final UHD finishing/export is implemented in Phase 6. See [Phase 5 workflow, supported providers and limits](docs/phase5.md).

## Phase 6 finishing

Open a production → **Finishing & final exports**. Approved edits render from originals into 1080p/4K landscape or portrait MP4s and versioned 24-bit stereo WAV masters. Quality reports include measured audio/video gates, scaling provenance and required human continuity review. Checkpointed rendering supports cancellation and recovery. See [Phase 6 workflow and limits](docs/phase6.md).


Artist creation now includes world-language/culture typeahead, custom values and randomized band-name suggestions. Visual prompts include cultural context, song/performance style and additional creative instructions while preserving approved continuity. See [artist creation](docs/artist-creation.md).
