# Other Build Systems

Periphery can't build these projects itself. The user's build produces an index store, and Periphery reads it through a JSON project config. Docs: https://periphery.pro/docs/build-systems#other-build-systems.

Step 3 (schemes) of the main skill does not apply. Steps 2, 4, 5, 7 and 8 still do.

## 1. Find or produce the index store

The build must compile with indexing enabled. For `swiftc`, that means `-index-store-path <dir>`. Ask the user how they build and where the index store goes. Every target that references the code must be built, including tests.

## 2. Write the project config

For example `periphery-project.json`. Show it to the user before writing it:

```json
{
    "indexstores": ["path/to/indexstore"],
    "test_targets": ["MyTests"],
    "plists": [],
    "xibs": [],
    "xcdatamodels": [],
    "xcmappingmodels": []
}
```

- Relative paths are relative to the directory Periphery runs from.
- List the Info.plists, XIBs/storyboards and Core Data models, because code they reference would otherwise look unused.

## 3. Reference it from `.periphery.yml`

```yaml
generic_project_config: periphery-project.json
```

## 4. Run the first scan

Build first so the index store is current, then run:

```sh
periphery scan --format agent-prompt > <output file>
```

Tell the user to rebuild before each scan. A stale index store gives wrong results.
