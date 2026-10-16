# Threat model



## Scope and trust boundary

python-discovery finds Python interpreters from an explicit path or specification, preferred candidates, the running
interpreter, `PATH`, Windows PEP 514 registrations and uv-managed installations. It executes candidates with a bundled
interrogation script, parses the result into `PythonInfo`, checks it against the requested `PythonSpec`, and can cache
that metadata on disk.

Executing an interpreter explicitly named by the caller, found on the caller's trusted `PATH`, or registered by the same
user is expected code execution. The library's security responsibilities are to preserve discovery order and selection
constraints, avoid command-line injection while interrogating a candidate, and keep cache operations within the
configured cache root.

Inputs to treat as hostile or malformed include:

- interpreter specifications, absolute paths, preferred candidates and predicates supplied by an API caller;
- `PATH`, home and uv state variables in the supplied environment mapping;
- directory entries, symlinks and shims encountered during discovery;
- stdout, stderr, exit status and reported metadata from a candidate process;
- Windows registry values for PEP 514, especially executable paths and `ExecutableArguments` raw command-line text;
- JSON cache entries, lock files and filesystem objects in a shared or previously populated cache directory;
- very slow, non-Python or adversarial executables that match a Python-like filename.



## Components to examine

Prioritize `_discovery.py` for candidate ordering, path resolution and shim handling. A relative specification must not
turn the current working directory into an implicit executable search ahead of `PATH`; version, implementation,
architecture, free-threaded and debug constraints must all apply to the final selected interpreter.

`_cached_py_info.py` constructs the subprocess command, bounds execution time, separates attacker output from the
cookie-delimited payload and handles Windows launcher arguments. Review quoting of `ExecutableArguments` for command
injection and ensure a malicious candidate cannot make the library accept metadata from surrounding output or a prior
process. Killing a timed-out process must also reap children and drain pipes without hanging.

Review `_cache.py` for symlink and race attacks on cache JSON and lock paths, unsafe permissions in a shared cache, and
cache-key aliasing between different executable paths or arguments. A cache entry must not remain valid after the
interpreter or interrogation script changes. `_windows/` registry enumeration and uv store traversal need platform-
specific path tests.



## Exercising the image

The project is installed from `/src` into `/opt/venv` with its test group. Run `python -m pytest tests`. Tests include
fake interpreter scripts, cache corruption and timeout cases. Security reproductions should pass an explicit environment
mapping and temporary `DiskCache`, then create executable fixtures that emit crafted cookie markers, partial JSON, large
output or delayed child processes.

Linux cannot execute Windows PEP 514 paths. Add a focused unit test around the Windows command builder and include the
exact Windows parsing rule in reports for Windows-only findings.



## Severity guidance

- High: an interpreter specification that should select only a constrained version or path instead executes a separate
  attacker-controlled command through argument injection, without the attacker already controlling trusted `PATH` or
  registry configuration.
- Medium: a different local user can poison a shared cache so a victim accepts false interpreter metadata or performs
  filesystem changes outside the cache root; a small candidate output causes sustained memory or process exhaustion.
- Low: discovery returns the wrong interpreter without crossing a privilege boundary, a malformed candidate crashes
  discovery, or stale data affects only a cache controlled by the same user.

Reports should state the platform, API call, environment mapping, candidate filesystem or registry state, cache root,
spawned argv and selected `PythonInfo`.



## Out of scope

- Code execution by an interpreter the caller explicitly selected or placed on its trusted `PATH`.
- A malicious Windows registration or uv installation owned and writable by the same user running discovery.
- Arbitrary behavior in a caller-supplied predicate.
- Incorrect metadata deliberately printed by an already-executing hostile interpreter, unless it makes discovery break a
  selection guarantee relevant to another security boundary.
- Bugs in filelock, the operating-system registry API or uv's installer that reproduce without python-discovery.
