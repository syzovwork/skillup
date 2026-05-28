# SocialChatbot Backend

.NET 8 (LTS) backend for the Multi-Brand Social Media Insights Chatbot PoC.
See `../specs/001-social-chatbot/plan.md` for full architecture and
`../specs/001-social-chatbot/quickstart.md` for the end-to-end run-through.

## Solution Layout

```text
backend/
├── SocialChatbot.sln
├── global.json                          # pins SDK to .NET 8.0.x
├── src/
│   ├── SocialChatbot.Domain/            # entities + pure logic, no I/O
│   ├── SocialChatbot.Application/       # SK orchestration, prompts, validators
│   ├── SocialChatbot.Infrastructure/    # SQL, Chroma, DIAL, PDF ingestion
│   └── SocialChatbot.Api/               # ASP.NET Core minimal API host
└── tests/
    ├── SocialChatbot.UnitTests/
    └── SocialChatbot.IntegrationTests/
```

Dependencies point inward:
`Api → Application → Domain` and `Infrastructure → Application → Domain`
(Api references Infrastructure only for composition-root wiring).

## Build & Run

```pwsh
# from the backend/ directory
dotnet build
dotnet test
dotnet run --project src/SocialChatbot.Api
```

## Configuration

Non-secret defaults live in `src/SocialChatbot.Api/appsettings.json`.
Secrets (`Dial:ApiKey`, `Sql:Password`) are loaded from .NET User Secrets in
local dev and from environment variables in containers. Constitution Security
Standards: **never commit secrets**.

```pwsh
cd src/SocialChatbot.Api
dotnet user-secrets init
dotnet user-secrets set "Dial:ApiKey" "<your-dial-key>"
dotnet user-secrets set "Dial:BaseUrl" "https://<your-dial-endpoint>"
dotnet user-secrets set "Sql:Password" "<sa-password>"
```

## Code Style

Code style is enforced by `../.editorconfig` (root). Notable rules:

- `csharp_style_namespace_declarations = file_scoped:warning`
- `dotnet_diagnostic.IDE0005.severity = warning` (unused-using is a warning)
- 4-space indent for C#, 2-space for csproj/sln/props/targets

### Format before committing

Run `dotnet format` to apply style fixes automatically:

```pwsh
dotnet format
```

### Suggested pre-commit hook

Add a Git pre-commit hook at `.git/hooks/pre-commit` (not committed):

```sh
#!/usr/bin/env sh
set -e
echo "[pre-commit] dotnet format --verify-no-changes"
cd "$(git rev-parse --show-toplevel)/backend"
dotnet format --verify-no-changes --verbosity quiet
```

Make it executable: `chmod +x .git/hooks/pre-commit`.
This blocks commits with style violations and matches what CI will enforce
(see T077).

## Tests

```pwsh
dotnet test                                            # all tests
dotnet test tests/SocialChatbot.UnitTests              # unit only (no docker)
dotnet test tests/SocialChatbot.IntegrationTests       # requires running SQL + Chroma + recorded DIAL fixtures
```

Per constitution Principle IV: unit tests are mandatory for business logic;
integration tests are mandatory for AI pipeline components. Tests use
recorded DIAL responses to remain deterministic.
