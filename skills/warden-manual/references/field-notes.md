# Field Notes

Supplementary guidance for the warden. Load this when you need deeper detail on common terrain.

## Common Traps

- **Shared mutable state**: Multiple callers writing to the same object without synchronization. Look for global variables, singletons, and shared caches.
- **Implicit ordering**: Code that depends on operations happening in a specific sequence without enforcing it. Watch for init functions, event handlers, and middleware chains.
- **Silent failures**: Errors swallowed by empty catch blocks, ignored return values, or default fallbacks that mask real problems.

## State Drift Patterns

- **Config vs runtime**: The config file says one thing, but environment variables or flags override it at startup. Always check the effective value, not just the declared one.
- **Stale caches**: A cache that was correct when populated but no longer reflects the source of truth. Look for TTLs, invalidation logic, and race conditions between read and write paths.
- **Database migrations**: The schema in code may not match the live database if migrations were skipped, partially applied, or rolled back.

## Investigation Checklist

1. Reproduce the problem with the simplest possible input.
2. Trace the data path from entry point to the first divergence.
3. Check recent changes to the files in that path.
4. Look for environmental differences between where it works and where it fails.
5. Narrow to a single hypothesis before making changes.
