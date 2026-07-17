# Plugins and Extensibility

**Project:** Lunar Script System User Guide  
**Status:** Draft 0.1  
**Date:** 2026-07-17  
**Audience:** Advanced Lunar Linux users, maintainers, and system administrators  
**Evidence basis:** LSS Evidence Foundation 0.1 — Checkpoints 1–6

## 1. Overview

LSS can be extended through plugins.

Plugins allow additional behavior to be inserted without rewriting the core lifecycle.

They may:

- intercept an operation;
- provide an alternative implementation;
- add checks;
- add integration;
- modify decision flow;
- extend module behavior;
- reload dynamically in selected contexts.

This makes LSS extensible while preserving a relatively small core.

## 2. Plugin loading order

The runtime initialization order is conceptually:

```text
/etc/lunar/config
→ core function libraries
→ local configuration
→ path normalization
→ display and sound setup
→ plugin loading
```

This means plugins are loaded after the core functions exist.

A plugin can therefore extend or intercept known behavior rather than replacing an undefined environment.

## 3. Plugin directory

Plugins are loaded from the configured plugin directory.

A common conceptual location is:

```text
plugin.d
```

The exact path is determined by LSS configuration.

Inspect active configuration:

```bash
grep -R -n 'PLUGIN' /etc/lunar /var/lib/lunar/functions   2>/dev/null
```

List plugin files:

```bash
find /var/lib/lunar   -type f   -name '*.plugin'   -print
```

## 4. Module-installed plugins

A Moonbase module may install plugin files.

This means a normal software module can extend LSS itself.

Search Moonbase:

```bash
grep -R -n -E 'plugin\.d|\.plugin'   /var/lib/lunar/moonbase
```

Search installed manifests:

```bash
grep -R -n -E 'plugin\.d|\.plugin'   /var/log/lunar/install
```

A plugin-bearing module deserves additional review before update or removal.

## 5. Registration order

Plugins are processed in registration order.

This matters because an earlier plugin may handle an operation and prevent later plugins from running.

Conceptually:

```text
plugin 1
→ plugin 2
→ plugin 3
→ default behavior
```

The chain may stop before reaching the end.

## 6. Return codes

Observed plugin return semantics are:

```text
0
→ operation handled successfully
→ stop processing

1
→ operation handled unsuccessfully
→ stop processing

2
→ continue to next plugin or default behavior
```

These codes are control-flow decisions, not only success/failure values.

## 7. Why return code 2 matters

In ordinary shell conventions:

```text
0
→ success

non-zero
→ failure
```

LSS plugin semantics add a third operational meaning:

```text
2
→ not handled here; continue
```

A plugin author must not treat every non-zero value as equivalent.

## 8. Plugin chain example

Conceptually:

```text
plugin A returns 2
→ plugin B runs

plugin B returns 0
→ chain stops successfully

default implementation
→ not called
```

Another case:

```text
plugin A returns 1
→ chain stops as failure
```

## 9. Plugins as interception points

A plugin may act at a specific lifecycle boundary.

Possible uses:

- source acquisition;
- cache recovery;
- dependency handling;
- policy checks;
- installation behavior;
- reporting;
- external integration.

The exact interception points depend on the plugin API exposed by LSS.

Inspect plugin code directly.

## 10. Inspecting a plugin

Read the full file:

```bash
sed -n '1,260p' /path/to/plugin.plugin
```

Search function definitions:

```bash
grep -n -E '^[[:space:]]*[a-zA-Z_][a-zA-Z0-9_]*[[:space:]]*\(\)'   /path/to/plugin.plugin
```

Search return values:

```bash
grep -n -E 'return[[:space:]]+[012]'   /path/to/plugin.plugin
```

Search registration:

```bash
grep -n -E 'register|plugin|hook'   /path/to/plugin.plugin
```

## 11. Plugin state and side effects

A plugin may interact with:

- filesystem;
- network;
- external services;
- package state;
- caches;
- logs;
- environment variables;
- module configuration.

Do not assume plugin effects appear in the module manifest.

A plugin can act outside the observed installation transaction.

## 12. Plugin provenance

Before trusting a plugin, determine:

```text
which module installed it?
which repository supplied it?
which version?
which manifest owns it?
which configuration enables it?
```

Search ownership:

```bash
grep -R -F -x '/path/to/plugin.plugin'   /var/log/lunar/install 2>/dev/null
```

## 13. Plugin removal

Removing the owning module may delete the plugin file.

However, a running `lin` process may already have loaded the plugin.

Therefore:

```text
file removed
≠ code unloaded from running process
```

Avoid removing plugin modules while LSS operations are active.

## 14. Runtime reload

Source analysis indicates plugin reload can be triggered through `USR1` into the parent `lin` process.

Conceptually:

```text
plugin set changes
→ signal parent lin
→ plugin environment reloads
```

This is advanced behavior.

Do not send signals to active package-manager processes without understanding the exact code path.

## 15. Inspecting active `lin` processes

```bash
ps -ef | grep '[l]in'
```

Inspect process tree:

```bash
pstree -ap | grep -A5 -B5 lin
```

Before plugin replacement or removal, wait for active operations to finish.

## 16. Plugin reload risks

Potential risks:

- partially changed plugin set;
- changed functions during active operation;
- mismatched state;
- plugin file removed before reload;
- new plugin loaded with old configuration;
- parent/child process disagreement.

Use reload only where explicitly supported.

## 17. Plugin debugging

Preserve:

```text
plugin file
registration order
LSS version
Moonbase revision
console output
activity log
environment
```

Run in a container.

Change one plugin at a time.

## 18. Detecting which plugin handled an operation

Possible methods:

- verbose console output;
- plugin-specific logs;
- activity log;
- temporary debug messages;
- controlled return-code changes;
- source-level tracing.

Use minimal instrumentation.

Do not modify several plugins simultaneously.

## 19. Plugin order conflicts

Two plugins may claim the same operation.

Because order matters:

```text
earlier plugin handles
→ later plugin never runs
```

A plugin can appear broken when it is simply shadowed.

Inspect registration sequence.

## 20. Default behavior fallback

If every plugin returns:

```text
2
```

the default LSS implementation may run.

This means a plugin can observe eligibility and decline handling without causing failure.

## 21. Successful interception

A plugin returning:

```text
0
```

takes responsibility for the operation.

It must leave the system in a state compatible with the expectations of the caller.

Success means more than command completion.

It may require:

- correct files;
- correct logs;
- correct state;
- correct return flow.

## 22. Failed interception

A return value of:

```text
1
```

halts the chain as failure.

Use this only when the plugin intentionally blocks or fails the operation.

A plugin that merely cannot handle the case should normally continue with:

```text
2
```

## 23. Plugin configuration

A plugin may use:

- global Lunar configuration;
- plugin-specific files;
- environment variables;
- module metadata;
- external credentials.

Search:

```bash
grep -R -n -E   '/etc/|CONFIG|config|credential|token|URL|PATH'   /path/to/plugin.plugin
```

Remove secrets before sharing debug bundles.

## 24. Plugin and cache behavior

A plugin may provide alternative cache or source behavior.

When a module installs unexpectedly quickly, determine whether:

- default cache resurrection ran;
- a plugin handled recovery;
- a plugin redirected source acquisition.

Inspect console and plugin order.

## 25. Plugin and dependency behavior

A plugin may influence dependency decisions.

This can change:

- selected provider;
- resolved relationship;
- build ordering;
- failure policy.

Compare `/var/state/lunar/depends` before and after plugin changes.

## 26. Plugin and ownership behavior

If a plugin installs files outside normal `prepare_install` observation, those paths may not appear in the module manifest.

This creates potential ownership gaps.

A plugin that changes installation behavior should preserve LSS ownership guarantees.

## 27. Safe plugin development

Recommended workflow:

```text
create plugin in isolated environment
→ register last in chain
→ return 2 by default
→ log decisions
→ handle one narrow case
→ test success
→ test decline
→ test failure
→ verify fallback
```

Start conservatively.

## 28. Safe plugin update

Before:

```bash
cp -a /path/to/plugin.plugin   /root/plugin.before

cp -a /var/state/lunar/packages   /root/packages.before

cp -a /var/state/lunar/depends   /root/depends.before
```

After:

- verify plugin syntax;
- verify registration;
- run a low-risk operation;
- inspect state;
- restore on unexpected behavior.

## 29. Safe plugin removal

Before removing the owning module:

1. identify active operations;
2. inspect plugin ownership;
3. inspect fallback behavior;
4. preserve plugin file;
5. stop or finish active `lin`;
6. remove module;
7. verify plugin directory;
8. run a low-risk LSS command.

## 30. Plugin syntax checking

Because plugins are shell code, use:

```bash
bash -n /path/to/plugin.plugin
```

This catches syntax errors, not semantic errors.

Also consider:

```bash
shellcheck /path/to/plugin.plugin
```

when available.

Review warnings manually.

## 31. Plugin logging

A useful plugin should log:

```text
interception point
decision
reason
return code
important external result
```

Avoid excessive logs.

The goal is traceability, not noise.

## 32. Plugin compatibility

A plugin may depend on:

- specific LSS function names;
- function arguments;
- global variables;
- directory layout;
- return semantics;
- lifecycle order.

An LSS update can therefore break a plugin even when the plugin file is unchanged.

Test plugins after core LSS updates.

## 33. Plugin API stability

Treat internal functions as unstable unless documented as public plugin interfaces.

A plugin built against undocumented globals may work but remain fragile.

Prefer small, explicit interfaces.

## 34. Extensibility boundaries

Plugins are appropriate when:

- behavior is optional;
- implementation is replaceable;
- core should remain small;
- external integration varies;
- fallback exists.

Plugins are less appropriate when:

- behavior is fundamental to package-state correctness;
- ordering cannot be controlled;
- ownership cannot be preserved;
- failures would corrupt core state.

## 35. Common mistakes

### Mistake 1: treating return 2 as failure

It means continue.

### Mistake 2: ignoring registration order

Earlier plugins may shadow later ones.

### Mistake 3: removing a plugin file during active `lin`

Loaded code may remain in memory.

### Mistake 4: installing files outside ownership tracking

This creates unmanaged state.

### Mistake 5: modifying several plugins at once

Failure attribution becomes difficult.

### Mistake 6: relying on undocumented core globals

Compatibility becomes fragile.

### Mistake 7: returning 0 without completing expected state

The caller assumes successful handling.

## 36. Plugin review template

```bash
PLUGIN=/path/to/plugin.plugin

echo '=== SYNTAX ==='
bash -n "$PLUGIN"

echo
echo '=== FUNCTIONS ==='
grep -n -E   '^[[:space:]]*[a-zA-Z_][a-zA-Z0-9_]*[[:space:]]*\(\)'   "$PLUGIN"

echo
echo '=== RETURNS ==='
grep -n -E 'return[[:space:]]+[012]' "$PLUGIN"

echo
echo '=== REGISTRATION ==='
grep -n -E 'register|hook|plugin' "$PLUGIN"

echo
echo '=== EXTERNAL PATHS ==='
grep -n -E '/etc/|/var/|/usr/|https?://' "$PLUGIN"
```

## 37. Plugin troubleshooting model

```text
operation requested
→ registration order
→ plugin eligibility
→ return code
→ next plugin or fallback
→ persistent state
→ runtime result
```

Debug each transition.

## 38. Extensibility and system intent

Plugins should extend system intent, not obscure it.

A good plugin makes clear:

- what it handles;
- when it declines;
- what state it changes;
- what fallback remains;
- how it is removed.

## 39. Summary

LSS plugins provide ordered, replaceable interception.

The central semantics are:

```text
0
→ handled successfully; stop

1
→ handled unsuccessfully; stop

2
→ not handled; continue
```

The operational rule is:

```text
understand registration order
→ preserve ownership and state guarantees
→ test fallback
→ avoid changing plugins during active operations
