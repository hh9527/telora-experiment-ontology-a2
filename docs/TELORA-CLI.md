# Telora validation workflow

The workspace provides a fixed Telora validation interface. Run commands from
the workspace root.

## Layout

Each Telora crate contains its reusable modules in `src/` and validation
entries in `bin-src/`.

The ontology crate is `ontology/`; the first enterprise model is `ent-1/`.
Use each crate's `bin-src/main.telora` as its main validation entry and
`bin-src/test.telora` as its focused validation entry.

Each crate's `telora-deps.json` fixes its dependency boundary. Do not modify it.

## Commands

The following commands are the complete executable interface available in the
experiment:

```text
./bin/telora run <crate>/bin-src/main.telora
./bin/telora run <crate>/bin-src/test.telora
./bin/telora types <crate>/bin-src/main.telora
./bin/telora show <crate>/bin-src/main.telora
```

- `run` evaluates the selected entry and prints its exported
  `output` value.
- the second `run` form evaluates `bin-src/test.telora` and prints its exported
  `output` value.
- `types` prints the inferred types for the main entry. It is the module-level type summary: quantified
  definitions retain their `for(...)` schemes, and internal function
  parameters are not listed as module bindings.
- `show` prints the semantic snapshot for the main entry,
  including diagnostics, modules, definitions, references, expressions, and
  types. Generic definition rows retain their quantified schemes. Nested
  parameter and expression rows are uninstantiated debug facts; an `Any` on
  those rows does not replace the enclosing definition's displayed scheme.
Each command preserves Telora's standard output, standard error, and exit
status. A zero exit status means that the requested operation succeeded. A
nonzero exit status means that Telora or the wrapper rejected it; read the
diagnostic, revise the source, and run the relevant command again.

The wrappers accept no source paths or semantic-query positions. They do not
discover files, mutate source, or run multiple validation entries. Rewrite
`test.telora` when a behavior should be tested independently from
`main.telora`.
