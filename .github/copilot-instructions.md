# RenderCV AI Agent Instructions

## Project Overview
RenderCV is a CV/resume generator that converts YAML → Typst → PDF. Core pipeline: Parse YAML with `ruamel.yaml`, validate with Pydantic models, generate Typst files via Jinja2 templates ([src/rendercv/renderer/templater](../src/rendercv/renderer/templater)), compile to PDF with `typst` package.

**Architecture:** Three main modules in [src/rendercv](../src/rendercv):
- **`schema/`**: Pydantic models, YAML parsing ([yaml_reader.py](../src/rendercv/schema/yaml_reader.py)), validation, JSON schema generation
- **`cli/`**: Typer-based CLI with subcommands in `*_command/` folders, auto-imported by [app.py](../src/rendercv/cli/app.py)
- **`renderer/`**: Jinja2 template rendering ([templater/](../src/rendercv/renderer/templater)), Typst/Markdown/HTML generation, PDF/PNG compilation

## Essential Workflows

### Development Setup
```bash
just sync        # Install dependencies with uv, locked versions from uv.lock
just format      # Format with black + ruff
just check       # Run ruff linting + pyright type checking (must pass with zero errors)
just test        # Run pytest suite
```

**Critical:** Always run `just format` and `just check` before committing. Type checking is strict—every function must have full type annotations (Python 3.12+ syntax: `str | None`, not `Optional[str]`).

### Testing Conventions
- Tests mirror `src/` structure: `tests/schema/test_*.py` tests `src/rendercv/schema/*.py`
- Reference file testing: `tests/renderer/test_*.py` compare generated output against `tests/renderer/testdata/`. Update with `just update-testdata` after intentional changes, then **manually verify** before committing.
- Custom pytest option `--update-testdata` available via [conftest.py](../tests/conftest.py) fixture

### Scripts and Automation
[scripts/](../scripts) automate maintenance tasks (not part of RenderCV runtime):
- `update_schema.py`: Regenerate [schema.json](../schema.json) from Pydantic models (run via `just update-schema`)
- `update_examples.py`: Regenerate all [examples/](../examples) YAML files and PDFs
- `create_executable.py`: Build standalone executable

## Code Conventions

### Docstring Requirements
**Google-style docstrings with mandatory "Why" section** explaining purpose/motivation:
```python
def build_rendercv_dictionary(
    main_input_file_path_or_contents: pathlib.Path | str,
    **kwargs: Unpack[BuildRendercvModelArguments],
) -> CommentedMap:
    """Merge main YAML with overlays and CLI overrides into final dictionary.

    Why:
        Users need modular configuration (separate design/locale files) and
        quick testing (CLI overrides). This pipeline applies all modifications
        before validation, ensuring users see complete configuration errors.

    Args:
        main_input_file_path_or_contents: Primary CV YAML file or string.
        kwargs: Optional YAML overlay paths, output paths, and CLI overrides.

    Returns:
        Merged dictionary ready for validation.
    """
```
Include "Example" section when adding value. Args/Returns sections are mandatory.

### Type Annotations
- **Strict typing required** (enforced by pyright)—no `Any` without justification
- Use modern syntax: `type` aliases, `[T]` type parameters, pipe unions
- Use `# pyright: ignore[errorCode]` or `#NOQA: errorCode` only when absolutely necessary

### Error Handling
Custom exceptions in [exception.py](../src/rendercv/exception.py):
- `RenderCVValidationError`: User input validation errors with YAML location tracking
- `RenderCVUserError`: General user-facing errors
- `RenderCVInternalError`: Internal bugs/unexpected states

## Project-Specific Patterns

### Variant Pydantic Model Generation
[variant_pydantic_model_generator.py](../src/rendercv/schema/variant_pydantic_model_generator.py) dynamically creates theme-specific Pydantic models. Each theme (Classic, Moderncv, etc.) shares structure but has different default colors/fonts. Variants inherit validation logic while exposing different defaults in JSON schema for IDE autocomplete.

### Build Pipeline
[rendercv_model_builder.py](../src/rendercv/schema/rendercv_model_builder.py) implements overlay system:
1. Load main YAML file
2. Apply optional overlays (design/locale/settings YAML files)
3. Apply CLI overrides (e.g., `--cv.phone="+123456"`)
4. Validate with Pydantic

All modifications happen **before** validation so users see errors on final merged config.

### CLI Structure
Subcommands live in `cli/*_command/` folders with matching `*_command.py` files (e.g., `render_command/render_command.py`). Auto-imported by [app.py](../src/rendercv/cli/app.py) via dynamic module loading. Error handling centralized in [error_handler.py](../src/rendercv/cli/error_handler.py) decorator.

## Key Files to Reference
- [pyproject.toml](../pyproject.toml): Dependencies, tool configs, entry points (heavily commented)
- [justfile](../justfile): All dev commands
- [docs/developer_guide/](../docs/developer_guide/): Architecture deep dives
- [src/rendercv/schema/models/](../src/rendercv/schema/models/): All Pydantic model definitions

## Dependency Management
Using `uv` for fast, deterministic installs. **Never edit `uv.lock` manually**—use `just lock` to update dependencies. Commit lockfile changes to ensure reproducible environments.
