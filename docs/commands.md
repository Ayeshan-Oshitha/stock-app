# Commands

A learning log of the commands used to build this project, in the order they were run.

---

## 1. Create the .NET solution

```bash
dotnet new sln -n StockApp
```

**What it does:** Creates an empty solution file named `StockApp.slnx` in the current folder. A solution groups all the .NET projects (gateway, services, shared libraries) so they can be built, tested and opened together.

**Run it from:** the repo root (`stock-app/`).

**Notes:**
- `-n StockApp` sets the name. Without it, the solution is named after the folder.
- With .NET 10, the default format is the newer XML-based `.slnx` instead of the older `.sln`.
- Projects are added later with `dotnet sln add <path-to-project.csproj>`.
