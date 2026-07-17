# LSS — Lunar Script System User Guide

This repository contains the first complete version of the **Lunar Script System User Guide 0.1**.

LSS is the package management and software build system used by Lunar Linux. The guide covers normal module operations, Moonbase structure, dependencies, persistent state, ownership, troubleshooting, recovery, plugins, policy states, and local module development.

## User Guide chapters

1. [Installing, Rebuilding, and Removing Modules](LSS-User-Guide-Installing-Rebuilding-Removing-Modules-0.1.md)
2. [Inspecting Modules and System State](LSS-User-Guide-Inspecting-Modules-and-System-State-0.1.md)
3. [Understanding Moonbase Modules](LSS-User-Guide-Understanding-Moonbase-Modules-0.1.md)
4. [Managing Dependencies and Optional Features](LSS-User-Guide-Managing-Dependencies-and-Optional-Features-0.1.md)
5. [Configuration, Options, and Reconfiguration](LSS-User-Guide-Configuration-Options-and-Reconfiguration-0.1.md)
6. [Logs, Manifests, and Troubleshooting](LSS-User-Guide-Logs-Manifests-and-Troubleshooting-0.1.md)
7. [Caches, Archives, and Recovery](LSS-User-Guide-Caches-Archives-and-Recovery-0.1.md)
8. [Updating the System Safely](LSS-User-Guide-Updating-the-System-Safely-0.1.md)
9. [Working with Moonbase](LSS-User-Guide-Working-with-Moonbase-0.1.md)
10. [Advanced Module Inspection and Debugging](LSS-User-Guide-Advanced-Module-Inspection-and-Debugging-0.1.md)
11. [Module Removal and Configuration Preservation](LSS-User-Guide-Module-Removal-and-Configuration-Preservation-0.1.md)
12. [Plugins and Extensibility](LSS-User-Guide-Plugins-and-Extensibility-0.1.md)
13. [Policy States: Held, Exiled, and Enforced Modules](LSS-User-Guide-Policy-States-Held-Exiled-Enforced-0.1.md)
14. [Building and Testing a Local Module](LSS-User-Guide-Building-and-Testing-a-Local-Module-0.1.md)
15. [System Recovery and State Repair](LSS-User-Guide-System-Recovery-and-State-Repair-0.1.md)
16. [Reference Commands and File Locations](LSS-User-Guide-Reference-Commands-and-File-Locations-0.1.md)

## Consolidated publication draft

The complete guide is also available as a single document:

- [Lunar Script System User Guide — Website Publication Draft 0.1](Lunar-Script-System-User-Guide-Website-Publication-Draft-0.1.md)

## Supporting material

The repository also contains the evidence checkpoints, lifecycle documentation, and the formal completion checkpoint used to build and validate this guide.

## Operating model

LSS administration follows one consistent discipline:

```text
understand intent
→ inspect current state
→ preserve evidence
→ perform one controlled operation
→ verify ownership
→ verify persistent state
→ verify runtime behavior
```

## Project status

```text
Evidence Foundation 0.1
→ established

Module Lifecycle 0.1
→ established

User Guide 0.1
→ complete

Website publication source
→ prepared
```

The next major documentation milestone is the **LSS Architecture Guide**.
