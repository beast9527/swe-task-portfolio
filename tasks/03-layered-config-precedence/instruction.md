# Instruction: Add Config File Support with Layered Precedence and Validation

## Overview

This task extends the existing command-line configuration system to support an optional JSON configuration file. The configuration file can set defaults for the mod directory, filter settings, mod loader, Minecraft version, and stability. The effective configuration is determined by a precedence order: CLI arguments override file settings, which override built-in defaults. The effective configuration is validated before any application action is taken. A new `--print-config` flag outputs the effective configuration as JSON and exits.

## Public API and Behavior

### New Functions

#### `find_config_file(explicit_path: Optional[str]) -> Optional[Path]`

- **Discovery order**: Returns the first existing path among:
  1. `explicit_path` (if provided)
  2. `Path('./minecraft_mod_manager.json')`
  3. `Path.home() / '.minecraft_mod_manager.json'`
- If `explicit_path` is provided but does not exist, it is skipped and the search continues with the default locations.
- If none of the candidate paths exist, returns `None`.
- No exceptions are raised for missing files.

#### `load_config_file(path: Path) -> Dict[str, Any]`

- Opens and reads the file at `path` as UTF-8 text and parses it as JSON.
- Raises `ConfigFileError` if:
  - The file does not exist.
  - The file is not readable (e.g., permission error).
  - The file contains invalid JSON.
  - The top-level JSON value is not an object.
- The error message must include the path.
- After parsing, the top-level object is checked for allowed keys: `'dir'`, `'filter'`, `'mod_loader'`, `'minecraft_version'`, and `'stability'`. Any other key raises `ConfigFileError` with a message listing the offending key.
- The loaded object is normalized:
  - `'dir'` must be a string.
  - `'filter'` must be a dict whose optional `'version'` and `'stability'` values must be strings.
- If any constraint is violated, raises `ConfigFileError` with a message identifying the invalid key.

#### `merge_settings(file_settings: Dict[str, Any], cli_args: Namespace) -> Dict[str, Any]`

- Returns a dictionary with exactly the keys: `'dir'`, `'filter'`, `'mod_loader'`, `'minecraft_version'`, and `'stability'`.
- For each top-level key, the value is taken from `cli_args` if the corresponding attribute exists and is not `None`, otherwise from `file_settings` if the key exists, otherwise from the built-in default.
- Built-in defaults:
  - `'dir'` = `'.'`
  - `'filter'` = `{'version': None, 'stability': None}`
  - `'mod_loader'` = `None`
  - `'minecraft_version'` = `None`
  - `'stability'` = `None`
- For the nested `'filter'` key, the merge is performed per subkey (`'version'`, `'stability'`) with the same precedence. The result always contains both subkeys.

#### `validate_config(merged: Dict[str, Any]) -> None`

- Checks that the `'dir'` value is a string and that the directory either exists or can be created (i.e., its parent exists and is writable). If not, adds an error message to the collected errors.
- Checks that `'mod_loader'` is one of `'forge'`, `'fabric'`, or `'quilt'` if present, and that `'stability'` is one of `'release'`, `'beta'`, or `'alpha'` if present. Any violation adds an error message to the collected errors.
- If any validation errors were collected, raises `ConfigValidationError` whose message contains all collected error messages separated by newlines.
- If no errors, returns `None`.

### Modified Function

#### `config.add_arg_settings(args: Namespace, file_settings: Optional[Dict[str, Any]] = None) -> None`

- If `file_settings` is not `None`, it merges `file_settings` with `args` using `merge_settings` and updates the global config state (the module-level `config` object) so that each attribute corresponding to a top-level key in the merged dictionary is set to that key's value. Specifically, after the call:
  - `config.dir` equals `merged['dir']`
  - `config.filter` equals a `Filter` object with attributes `version` and `stability` set from `merged['filter']['version']` and `merged['filter']['stability']`
  - `config.mod_loader` equals `merged['mod_loader']`
  - `config.minecraft_version` equals `merged['minecraft_version']`
  - `config.stability` equals `merged['stability']`
- If `file_settings` is `None`, the function behaves exactly as the existing implementation: it reads attributes from `args` and sets the same config attributes accordingly, using the same attribute names and defaults as before.

### `main()` Flow

- After parsing args, if `args.config` is present, call `find_config_file(args.config)`; otherwise call `find_config_file(None)`.
- If a path is returned, call `load_config_file(path)` to obtain `file_settings`.
- Then call `merge_settings(file_settings if file_settings is not None else {}, args)` to obtain `merged`.
- Then call `validate_config(merged)`.
- Then call `config.add_arg_settings(args, merged)`.
- If no config file is found, `file_settings` is `None` and the flow proceeds exactly as before: `config.add_arg_settings(args)` is called with no `file_settings`.
- If `validate_config` raises `ConfigValidationError`, `main()` prints the error message to stderr and exits with a non-zero exit code without executing any application action.
- When `--print-config` is present in args, `main()` prints the effective merged configuration as a JSON object to stdout and exits with code 0 without executing any application action. The JSON object contains exactly the keys `'dir'`, `'filter'`, `'mod_loader'`, `'minecraft_version'`, and `'stability'` with their effective values. The effective values are those produced by `merge_settings(file_settings if file_settings is not None else {}, args)` after validation has passed. The JSON is printed with `json.dumps` using default separators and no indentation, followed by a newline.

## Error Types

- `ConfigFileError`: Raised for file discovery/loading/normalization errors. The message must include relevant details (e.g., path, offending key).
- `ConfigValidationError`: Raised when validation fails. The message contains all collected error messages separated by newlines.

## Implementation Constraints

- The functions must be implemented in the specified modules: `find_config_file` and `load_config_file` in `minecraft_mod_manager/config_file.py`; `merge_settings` in `minecraft_mod_manager/config_merge.py`; `validate_config` in `minecraft_mod_manager/config_validation.py`.
- The existing `config.add_arg_settings` and `main()` must be modified to integrate the new functionality.
- The behavior when no config file is present must remain unchanged from the existing implementation.
- The `--print-config` flag must be added to the CLI argument parser.
- The global config object must be updated as specified.

## Compatibility

- All existing CLI arguments and behaviors must remain functional.
- The new config file support is optional; if no config file is found, the system behaves as before.
- The precedence rules (CLI > file > defaults) must be strictly followed.
- The exact set of allowed keys and validation rules must be enforced as described.

## Acceptance Criteria

- The functions `find_config_file`, `load_config_file`, `merge_settings`, and `validate_config` behave as specified.
- The `config.add_arg_settings` function correctly merges file settings when provided and preserves existing behavior when not.
- The `main()` flow correctly discovers, loads, merges, and validates configuration before any action.
- The `--print-config` flag outputs the effective configuration as JSON and exits with code 0.
- Validation errors are printed to stderr and cause a non-zero exit.
- All error messages are informative and include necessary details.
