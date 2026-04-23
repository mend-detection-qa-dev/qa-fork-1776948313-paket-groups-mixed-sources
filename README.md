# paket-groups-mixed-sources

## Feature exercised

This probe exercises Paket's named dependency groups combined with heterogeneous source types: the default (Main) group uses a standard NuGet source, the `Scripts` group uses a GitHub file source, and the `Tools` group uses a git repository source. This is the most structurally complex `paket.lock` layout — GROUP separators interleaved with non-NUGET section headers (GITHUB, GIT).

## Probe metadata

- **Pattern**: `groups-mixed-sources`
- **Target framework**: `net8.0`
- **Storage mode**: `storage: none`
- **Generated**: 2026-04-22T00:00:00Z

## Groups and sources declared

| Group   | Source type | Declaration                                                      |
|---------|-------------|------------------------------------------------------------------|
| Main    | nuget       | `https://api.nuget.org/v3/index.json`                            |
| Scripts | github      | `fsprojects/Chessie src/Chessie/ErrorHandling.fs`               |
| Tools   | git         | `https://github.com/fsprojects/Chessie.git master`              |

## Expected dependency tree

### Packages that must appear

| artifactId                     | version / commit                                   | dependencyType | Group   | Section |
|--------------------------------|----------------------------------------------------|----------------|---------|---------|
| `Newtonsoft.Json`              | `13.0.3`                                           | `NUGET`        | Main    | NUGET   |
| `Serilog`                      | `3.1.1`                                            | `NUGET`        | Main    | NUGET   |
| `src/Chessie/ErrorHandling.fs` | `1de397e9a95aba37ec9399c0eb7f83d851f43b01`         | `GITHUB`       | Scripts | GITHUB  |
| `Chessie` (git repo)           | `1de397e9a95aba37ec9399c0eb7f83d851f43b01`         | `GIT`          | Tools   | GIT     |

### Key assertions for Mend detection

- The default NUGET section (Main group) must resolve `Newtonsoft.Json 13.0.3` and `Serilog 3.1.1` as top-level dependencies with no transitive children.
- `GROUP Scripts` introduces a GITHUB section immediately after its header. A parser that only reads NUGET sections within groups will drop this dependency entirely.
- `GROUP Tools` introduces a GIT section immediately after its header. A parser that stops processing after encountering a non-NUGET section header will drop this dependency entirely.
- No transitive dependencies are expected from GITHUB or GIT source types.
- All four entries must appear in the output tree — any subset indicates a parser that fails to handle GROUP + non-NUGET combinations.

## Mend detection failure modes targeted

- **GROUP separator breaks parsing**: Parser reads the default NUGET block correctly but treats `GROUP Scripts` as a terminator and discards everything after it, producing only 2 of 4 expected entries.
- **Non-NUGET section within group silently skipped**: Parser traverses GROUP blocks but only handles NUGET sub-sections, dropping GITHUB and GIT sections inside named groups.
- **GITHUB section within group dropped**: Specifically, `GROUP Scripts / GITHUB` is not parsed because the GITHUB handler is only invoked for top-level (ungrouped) sections.
- **GIT section within group dropped**: Same failure mode for `GROUP Tools / GIT`.
- **Commit SHA not captured for group-scoped sources**: Version field is empty or missing for the git/github entries even when the entry itself is detected.
