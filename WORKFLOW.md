# Development Workflow

1. Before work: run `git status --short`, confirm branch, inspect existing files and conventions.
2. Keep changes focused and avoid unrelated refactoring.
3. Use DTOs for API contracts; keep Controllers thin; put business operations in Services.
4. Pass CancellationToken through async operations.
5. Enforce company ownership and authorization on every tenant-owned operation.
6. Before commit, run:
```powershell
dotnet build StockPilot.slnx
git diff --check
git status --short
```
7. Review `git diff`, stage only intended files, and use descriptive commit messages.
8. A local commit is not a push. Verify push success before saying GitHub is updated.
9. Update progress documents and DECISIONS.md after meaningful milestones.
10. Never commit secrets, credentials, customer data, or local database files.
