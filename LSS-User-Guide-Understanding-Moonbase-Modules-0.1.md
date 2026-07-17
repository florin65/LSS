# Understanding Moonbase Modules

**Project:** Lunar Script System User Guide  
**Status:** Draft 0.1  
**Date:** 2026-07-17  
**Audience:** Lunar Linux users, maintainers, and system administrators  
**Evidence basis:** LSS Evidence Foundation 0.1 — Checkpoints 1–6

## 1. Overview

Moonbase is the collection of module definitions used by LSS.

A module describes how software is:

- identified;
- downloaded;
- configured;
- built;
- installed;
- integrated;
- removed;
- related to other modules.

A module is not only metadata and not only a shell script.

It is a lifecycle definition.

Typical module directory:

```text
/var/lib/lunar/moonbase/<section>/<module>
```

Example:

```text
/var/lib/lunar/moonbase/core/filesys/foremost
```

## 2. Locating a module

Use:

```bash
lvu where module
```

Example:

```bash
lvu where foremost
```

Output:

```text
core/filesys
```

Construct the full path:

```bash
MODULE=foremost
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

printf '%s\n' "$MODULE_DIR"
```

List its files:

```bash
find "$MODULE_DIR" -maxdepth 2 -type f -print | sort
```

## 3. Common module files

A module may contain:

```text
DETAILS
BUILD
DEPENDS
CONFIGURE
OPTIONS
PRE_BUILD
POST_BUILD
POST_INSTALL
PRE_REMOVE
POST_REMOVE
CONFLICTS
patches
auxiliary files
plugin files
```

Not every file is required.

The most common minimum is:

```text
DETAILS
BUILD
```

The presence of extra files means the module extends the default lifecycle.

## 4. `DETAILS`

`DETAILS` defines module identity and source metadata.

It commonly contains fields such as:

```text
MODULE
VERSION
SOURCE
SOURCE_URL
SOURCE_VFY
WEB_SITE
ENTERED
UPDATED
SHORT
cat << EOF
long description
EOF
```

Exact fields may vary by module age and conventions.

Typical responsibilities:

```text
module name
version
source archive name
download location
integrity verification
upstream website
short description
long description
maintenance timestamps
```

Inspect with:

```bash
sed -n '1,240p' "$MODULE_DIR/DETAILS"
```

## 5. Why `DETAILS` matters

`DETAILS` answers:

```text
what is this module?
which version does Moonbase define?
where does the source come from?
how is the source verified?
what does the software do?
```

Before upgrading or debugging a download failure, inspect:

- `VERSION`;
- `SOURCE`;
- `SOURCE_URL`;
- verification fields;
- upstream project location.

A mismatch between installed and Moonbase versions is normal when an update is available.

## 6. `BUILD`

`BUILD` defines build and installation actions.

Example pattern:

```bash
./configure --prefix=/usr &&
make &&
prepare_install &&
make install
```

Many modules use shared helpers:

```bash
default_build
default_make
prepare_install
sedit
```

Inspect with:

```bash
sed -n '1,240p' "$MODULE_DIR/BUILD"
```

## 7. The critical role of `prepare_install`

`prepare_install` marks the semantic boundary between:

```text
build-local activity
```

and:

```text
system installation activity
```

Typical structure:

```bash
configure
make
prepare_install
make install
```

Before `prepare_install`, files are normally created inside the build tree.

After `prepare_install`, installwatch observes filesystem operations performed against the system.

A custom `BUILD` that installs files before calling `prepare_install` may bypass normal ownership tracking.

## 8. Default build helpers

Common helpers reduce repeated shell code.

### `default_build`

Usually performs a conventional sequence equivalent to:

```text
configure
→ make
→ prepare_install
→ make install
```

The exact implementation belongs to the loaded LSS function libraries.

### `default_make`

Usually handles projects that need only:

```text
make
→ prepare_install
→ make install
```

A module using a default helper is still worth inspecting because it may alter variables or patch files before the helper call.

## 9. Example: `foremost`

Observed `BUILD`:

```bash
export CFLAGS+=" -fcommon"
sedit 's:/usr/local:/usr:g;s:/usr/man:/usr/share/man:g;s:/usr/etc:/etc:g' Makefile config.c &&
sedit "s:-O2:$CFLAGS:" Makefile &&
default_make
```

Interpretation:

```text
add compiler compatibility flag
→ normalize installation paths
→ inject Lunar compiler flags
→ run default make/install lifecycle
```

This module had no custom remove hooks.

## 10. Example: `xxhash`

The reference module was useful because its build installed a static archive and then removed it.

Conceptually:

```text
make
→ prepare_install
→ make install
→ remove /usr/lib/libxxhash.a
```

Installwatch observed the file lifecycle, but the final manifest omitted the removed archive.

This shows why reading the complete `BUILD` matters.

The final state may differ from the immediate result of `make install`.

## 11. `DEPENDS`

`DEPENDS` declares module relationships.

It may express:

- required dependencies;
- optional dependencies;
- enabled or disabled selections;
- purpose-specific parameters;
- conditional choices.

Inspect with:

```bash
test -f "$MODULE_DIR/DEPENDS" &&
  sed -n '1,240p' "$MODULE_DIR/DEPENDS"
```

The persistent result is represented in:

```text
/var/state/lunar/depends
```

But the source declaration and the persistent state are not the same thing.

## 12. Declaration versus persistent dependency state

`DEPENDS` describes what the module can or should depend on.

`/var/state/lunar/depends` describes the resolved stored relationship for the current system.

Therefore:

```text
DEPENDS
→ declaration and choices

depends database
→ current resolved state
```

A change in `DEPENDS` does not automatically prove that the persistent database has already changed.

The module must be configured, installed, or otherwise processed.

## 13. Required and optional dependencies

A required dependency is necessary for the selected module configuration.

An optional dependency may enable or disable functionality.

Before removing a module, inspect both:

```bash
lvu depends target
lvu leert target
```

Also inspect direct records:

```bash
grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends
```

A module may be operationally important even when it is only an optional dependency.

## 14. `CONFIGURE`

`CONFIGURE` is used when a module needs interactive or stored configuration decisions.

It may ask questions such as:

- enable a feature;
- select a backend;
- choose an implementation;
- include optional integration;
- use a particular build mode.

The answers can influence:

```text
dependencies
compiler flags
configure switches
installed files
runtime features
```

Inspect with:

```bash
test -f "$MODULE_DIR/CONFIGURE" &&
  sed -n '1,240p' "$MODULE_DIR/CONFIGURE"
```

## 15. `OPTIONS`

`OPTIONS` may define selectable build options in a structured way.

Depending on module conventions, options may map user choices to:

```text
configure flags
feature toggles
dependency activation
environment variables
```

Inspect with:

```bash
test -f "$MODULE_DIR/OPTIONS" &&
  sed -n '1,240p' "$MODULE_DIR/OPTIONS"
```

Configuration and options should be reviewed before comparing two builds.

Two systems can build the same module version with materially different results.

## 16. `PRE_BUILD`

`PRE_BUILD` runs before the main `BUILD`.

Typical uses:

- patch source files;
- generate files;
- adjust environment;
- remove bundled components;
- prepare build directories;
- apply compatibility fixes.

Inspect with:

```bash
test -f "$MODULE_DIR/PRE_BUILD" &&
  sed -n '1,240p' "$MODULE_DIR/PRE_BUILD"
```

A failure here is reported as a `PRE_BUILD` lifecycle failure.

## 17. `POST_BUILD`

`POST_BUILD` runs after `BUILD` and before final installation processing completes.

Typical uses:

- remove unwanted installed files;
- adjust generated files;
- create symlinks;
- normalize permissions;
- perform cleanup that must affect final ownership.

This stage is particularly important because it can change the final manifest.

A file installed during `BUILD` and removed during `POST_BUILD` may appear in raw installwatch activity but not in the final ownership manifest.

## 18. `POST_INSTALL`

`POST_INSTALL` runs after the core installation transaction.

The module lock is released before this stage.

Typical uses:

- print notices;
- refresh external indexes;
- run integration commands;
- perform actions that should happen after ownership state is committed.

A failure here is distinct from a build failure.

The software may already be installed even if `POST_INSTALL` reports a problem.

## 19. `PRE_REMOVE`

`PRE_REMOVE` runs before normal ownership removal.

Typical uses:

- stop a service;
- unregister integration;
- prepare data migration;
- remove generated runtime state;
- warn the administrator.

Inspect before removing sensitive modules:

```bash
test -f "$MODULE_DIR/PRE_REMOVE" &&
  sed -n '1,240p' "$MODULE_DIR/PRE_REMOVE"
```

A remove operation can have effects not visible in the install manifest when hooks are present.

## 20. `POST_REMOVE`

`POST_REMOVE` runs after the main removal operation.

Typical uses:

- refresh caches;
- rebuild indexes;
- remove residual integration;
- print administrative guidance.

Inspect with:

```bash
test -f "$MODULE_DIR/POST_REMOVE" &&
  sed -n '1,240p' "$MODULE_DIR/POST_REMOVE"
```

Do not assume that `lrm` only deletes manifest paths.

Hooks can add lifecycle-specific actions.

## 21. `CONFLICTS`

`CONFLICTS` declares modules or conditions that cannot coexist safely.

Possible reasons:

- duplicate implementations;
- overlapping files;
- incompatible daemons;
- competing providers;
- mutually exclusive toolchains.

Inspect with:

```bash
test -f "$MODULE_DIR/CONFLICTS" &&
  sed -n '1,240p' "$MODULE_DIR/CONFLICTS"
```

Conflict handling happens before the main installation transaction.

## 22. Patches and auxiliary files

A module may ship patches or helper files in its directory.

List everything:

```bash
find "$MODULE_DIR" -maxdepth 2 -type f -print | sort
```

Common auxiliary content:

```text
*.patch
*.diff
configuration templates
service files
desktop files
scripts
udev rules
plugin files
```

The presence of a file does not prove it is used.

Search references:

```bash
grep -R -n -F 'filename' "$MODULE_DIR"
```

## 23. Installed plugins

A Moonbase module may install LSS plugin files under a plugin directory such as:

```text
plugin.d
```

This means a module can extend LSS itself.

Before removing such a module, inspect:

- installed plugin files;
- module manifest;
- plugin registration;
- current running `lin` processes.

Plugins may be reloaded through a signal path involving `USR1`.

A plugin-bearing module deserves more caution than a simple leaf utility.

## 24. Shell semantics

Moonbase modules are executable shell logic.

This gives them flexibility, but also means behavior can depend on:

- environment variables;
- shell functions;
- command availability;
- path layout;
- toolchain selection;
- configuration state;
- previously installed files.

Read modules as programs, not static metadata.

Important shell features include:

```text
&& chains
conditional branches
variable expansion
command substitution
function calls
environment export
file mutation
```

## 25. Understanding `&&` chains

Example:

```bash
step_one &&
step_two &&
step_three
```

Meaning:

```text
run step_one
→ only if successful, run step_two
→ only if successful, run step_three
```

A failure stops the chain.

When diagnosing a module, identify the first failed command rather than only the final lifecycle label.

## 26. Environment variables

Modules commonly modify:

```text
CFLAGS
CXXFLAGS
LDFLAGS
PATH
CC
CXX
MAKEFLAGS
```

Example:

```bash
export CFLAGS+=" -fcommon"
```

Toolchain-sensitive modules may require explicit:

```bash
CC=/usr/bin/clang
CXX=/usr/bin/clang++
```

A toolchain mismatch can produce errors unrelated to the upstream source.

Always inspect module-local exports and global Lunar configuration.

## 27. `sedit`

`sedit` is a Lunar helper used to edit files in place.

Example:

```bash
sedit 's:/usr/local:/usr:g' Makefile
```

Operationally:

```text
modify source/build file before compilation or installation
```

When a generated path looks unexpected, search the module for `sedit` calls.

## 28. Module path conventions

Modules are grouped into sections.

Examples:

```text
core/libs
core/filesys
core/utils
```

Section names organize Moonbase.

The module name remains the normal command argument:

```bash
lin foremost
```

not:

```bash
lin core/filesys/foremost
```

Use `lvu where` to map name to location.

## 29. Reading a module before installation

Minimal review:

```bash
MODULE=name
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

echo '=== DETAILS ==='
sed -n '1,240p' "$MODULE_DIR/DETAILS"

echo
echo '=== BUILD ==='
sed -n '1,240p' "$MODULE_DIR/BUILD"

echo
echo '=== DEPENDS ==='
test -f "$MODULE_DIR/DEPENDS" &&
  sed -n '1,240p' "$MODULE_DIR/DEPENDS"

echo
echo '=== HOOKS ==='
for hook in PRE_BUILD POST_BUILD POST_INSTALL PRE_REMOVE POST_REMOVE; do
  test -f "$MODULE_DIR/$hook" || continue
  echo "--- $hook"
  sed -n '1,240p' "$MODULE_DIR/$hook"
done
```

## 30. Reading a module before removal

Focus on:

```text
reverse dependencies
PRE_REMOVE
POST_REMOVE
installed manifest
configuration files
services and plugins
shared files
```

Template:

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

lvu depends "$MODULE"
lvu leert "$MODULE"

for hook in PRE_REMOVE POST_REMOVE; do
  test -f "$MODULE_DIR/$hook" &&
    sed -n '1,240p' "$MODULE_DIR/$hook"
done

cat "/var/log/lunar/install/${MODULE}-${VERSION}"
```

## 31. Reading a module during troubleshooting

When build fails:

1. note the lifecycle phase;
2. inspect compile log;
3. inspect `BUILD`;
4. inspect `PRE_BUILD`;
5. inspect toolchain variables;
6. inspect patches;
7. inspect source version and checksum;
8. compare with a known working module pattern.

Useful searches:

```bash
grep -R -n -E 'CC=|CXX=|CFLAGS|LDFLAGS|prepare_install|default_'   "$MODULE_DIR"
```

## 32. Module behavior versus final ownership

A module can:

- create temporary files;
- remove installed files;
- generate files dynamically;
- install shared paths;
- modify external state;
- run commands after installation.

The final install manifest only captures attributed final paths.

It does not fully describe:

- every command executed;
- every transient file;
- every external side effect;
- every service action;
- every plugin action.

For complete understanding, combine:

```text
module source
compile log
raw installwatch when instrumented
final manifest
persistent state
activity log
```

## 33. Safe assumptions

Usually safe:

```text
DETAILS defines module identity and source
BUILD defines main build/install logic
DEPENDS defines dependency declarations
hooks extend lifecycle stages
install manifest records final owned paths
```

Not safe without inspection:

```text
module has no side effects
leaf means harmless
same version means same build
no manifest difference means no metadata change
no dependency records means no operational dependency
```

## 34. A compact anatomy map

```text
DETAILS
→ identity, version, source, description

CONFIGURE / OPTIONS
→ user choices and build variation

DEPENDS
→ dependency declarations

PRE_BUILD
→ prepare source and environment

BUILD
→ compile and install

POST_BUILD
→ adjust final installed state

POST_INSTALL
→ actions after core installation

PRE_REMOVE
→ actions before ownership removal

POST_REMOVE
→ actions after removal

CONFLICTS
→ incompatible modules or states

patches / auxiliary files
→ implementation support
```

## 35. User checklist

Before install:

```text
read DETAILS
read BUILD
inspect DEPENDS
inspect options
note toolchain requirements
```

Before rebuild:

```text
record installed version
save old manifest
check changed module files
check compiler environment
```

Before removal:

```text
check reverse dependencies
read remove hooks
save logs
inspect /etc ownership
check services and plugins
```

## 36. Summary

A Moonbase module is an executable lifecycle definition.

Its structure connects:

```text
source metadata
→ configuration
→ dependency decisions
→ build logic
→ observed installation
→ final ownership
→ removal behavior
```

The practical rule is:

```text
do not judge a module by its name or size
→ read the module
→ inspect current state
→ understand its lifecycle role
