# D365FO Development Guide

Shared knowledge base for all D365FO metadata projects. Project-specific settings (model name, prefix, paths) live in the project CLAUDE.md.

## Development Environment

D365FO metadata development requires:

- Visual Studio with Dynamics 365 Finance + Operations developer tools extension
- A running D365FO dev VM or cloud-hosted instance
- The model folder placed in the correct Packages path
- Compiler (`xppc.exe`) and best practice checker (`xppbp.exe`) in the platform `bin` folder

Pick the newest version folder:

```powershell
Get-ChildItem "$env:LOCALAPPDATA\Microsoft\Dynamics365" | Sort-Object Name -Descending | Select-Object -First 1
```

Platform bin path pattern:

```text
C:\Users\larsg\AppData\Local\Microsoft\Dynamics365\<version>\PackagesLocalDirectory\bin\
```

There are no `test` or `lint` CLI commands — everything runs inside the D365FO toolchain.

## Artifact Types and Folder Structure

Each artifact is a single XML file. File names match the artifact name exactly.

| Folder | Contents |
| --- | --- |
| `AxClass/` | X++ classes (business logic, handlers, services) |
| `AxTable/` | Custom tables |
| `AxTableExtension/` | Extensions to standard tables |
| `AxForm/` | Custom forms |
| `AxFormExtension/` | Extensions to standard forms |
| `AxDataEntityView/` | Data entities (OData/DMF) |
| `AxSecurityPrivilege/` | Security privileges |
| `AxLabelFile/` | Label files (en-US, nb-NO) |

## Naming Conventions

Replace `<PREFIX>` and `<MODEL>` with values from the project CLAUDE.md.

- Custom artifacts: `<PREFIX><BusinessConcept>` (e.g., `FKAAddressType`, `ZTLShipment`)
- Extension classes: `<OriginalClass>_<MODEL>_Extension` with `[ExtensionOf(...)]` attribute
- Extension artifacts: `<OriginalArtifact>.<MODEL>` (e.g., `CustTable.FKACommon`)
- Staging tables: `<PREFIX><Entity>Staging` paired with `<PREFIX><Entity>Entity`
- Labels: referenced as `@<MODEL>:LabelKey` in XML metadata

## Cross Reference

Use the D365Xref MCP tool to find cross references between metadata artifacts and code elements.
The tool expects path names on the format `Class/<ClassName>` to identify an object.
Use these together with the project file to identify related elements. Ask the user for a relevant project file if needed.

## Key Architectural Patterns

**Extension over modification**: Use D365FO extension points (`[ExtensionOf]`, table/form extensions) rather than overlayering. Never modify standard objects directly.

**Data Management Framework (DMF)**: Bulk import/export uses staging tables (`<PREFIX>*Staging`) paired with data entities (`<PREFIX>*Entity`). The staging table holds temporary data; the entity maps it to the target table.

**A2X document framework**: Report/document generation uses the A2X framework (`AX2XML` module dependency). Abstract base classes have concrete subclasses implementing document-specific logic.

**Business events**: Integration modules use D365FO business event contracts and handlers.

## Labels and Localization

Add new labels to `AxLabelFile/<MODEL>_en-US.xml`. Norwegian translations go in `AxLabelFile/<MODEL>_nb-NO.xml`. Label IDs must be unique within the file. Reference labels in code as `"@<MODEL>:YourLabelId"`.

## Compiling with xppc.exe

### Compile a module

```bat
set PKG=C:\Users\larsg\AppData\Local\Microsoft\Dynamics365\<version>\PackagesLocalDirectory
set REPOS=<MetadataRoot>

"%PKG%\bin\xppc.exe" ^
  -metadata="%REPOS%" ^
  -compilermetadata="%PKG%" ^
  -referenceFolder="%PKG%" ^
  -referenceFolder="%REPOS%" ^
  -modelmodule=FKACommon ^
  -appBase="%PKG%\bin" ^
  -refPath="%REPOS%\FKACommon\bin" ^
  -output="%REPOS%\FKACommon\bin" ^
  -log="%REPOS%\FKACommon\BuildModelResult.log" ^
  -xmllog="%REPOS%\FKACommon\BuildModelResult.xml" ^
  -verbose
```

| Parameter | Purpose |
| --- | --- |
| `-metadata` / `-compilermetadata` / `-referenceFolder` | Points to `PackagesLocalDirectory`; resolves references to standard and ISV modules |
| `-modelmodule` | The package (module) name — not the model name |
| `-output` / `-refPath` | Where compiled module assemblies are written |
| `-log` | Plain-text log, mirrors VS Output window |
| `-xmllog` | Full structured log with all severities |

### Reading the log files

Three artifacts are written next to the module after every build:

- `BuildModelResult.log` — flat text, one issue per line. Quick `grep` target.
- `BuildModelResult.xml` — full `<Diagnostics>` document, one `<Diagnostic>` per finding.

Each `<Diagnostic>` has: `DiagnosticType`, `Severity`, `Path`, `Line`, `Column`, `Moniker`, `Message`.

### Workflow after a code change

1. Run `xppc.exe` for the affected module.
2. Read `BuildModelResult.xml` and filter for `Severity = Error` or `Fatal` — if there are none, the module compiles clean.
3. Otherwise extract `<Path>` + `<Line>` + `<Message>`, open the file in `AxClass/`, `AxTable/`, etc., and fix.
4. Re-run and confirm the error is gone.
5. Scan `BuildModelResult.log` for new `Compile Warning` / `Metadata Warning` lines.

```powershell
# Quick error summary
([xml](Get-Content "$REPOS\<MODELMODULE>\BuildModelResult.xml")).Diagnostics.Items.Diagnostic |
    Where-Object { $_.Severity -in 'Error','Fatal' } |
    Select-Object Severity, Path, Line, Moniker, Message | Format-Table -AutoSize
```

## Best Practice Checks with xppbp.exe

`xppbp.exe` lives next to `xppc.exe`. Run after a clean compile.

```bat
"%PKG%\bin\xppbp.exe" ^
  -metadata="%PKG%" ^
  -compilermetadata="%PKG%" ^
  -module=<MODELMODULE> ^
  -model=<MODELMODULE> ^
  -all ^
  -log="%PKG%\<MODELMODULE>\BPCheckResult.log" ^
  -xmllog="%PKG%\<MODELMODULE>\BPCheckResult.xml"
```

| Goal | Command fragment |
| --- | --- |
| All artifacts in a module | `-module=<M> -all` |
| One artifact type | `-module=<M> class:*` (also `form:*`, `table:*`, `query:*`) |
| One specific element | `-module=<M> class:<ClassName>` |

`BPCheckResult.xml` uses the same `<Diagnostics>` schema as the compiler log. BP findings have `DiagnosticType=BestPractice` and a rule `Moniker` (e.g. `BPRuleClassNamePrefix`).

### Pre-PR loop

1. Run `xppc.exe` — ensure `BuildModelResult.log` does not conatin errors.
2. Run `xppbp.exe` with `-all`.
3. Triage `BPCheckResult.xml` by `Severity` then `Moniker`. Fix naming-prefix violations, missing labels, public methods without TTSBegin/TTSCommit.
4. Treat any new `BPRule*` finding introduced by the change as a blocker.
5. Never silence a `Compile Error` with a BP suppression — fix the source first.

BP rules are evaluated against **English** identifiers and **English + Norwegian** labels.
