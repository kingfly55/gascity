---
title: "Agent Patterns"
---

Common patterns for structuring agents in a Gas City workspace.

## Scout: discovering work and creating beads

A scout agent proactively examines a repository (or any external source),
creates work beads for what it finds, and then exits. Other agents — polecats,
reviewers, etc. — pick up those beads through normal pool routing. This
separates discovery from execution and lets you bootstrap work without any
manual bead creation.

### How it works

Configure the scout as a city-scoped agent with `mode = "always"` so the
controller keeps it alive at city level:

```toml
[[agent]]
name = "scout"
scope = "city"
prompt_template = "prompts/scout.md"

[[named_session]]
template = "scout"
mode = "always"
```

The scout's prompt instructs it to inspect repositories, identify work, and
create beads — then signal it's done:

```markdown
# Scout

You discover work that needs doing and file it as beads.

1. Run `gc rig list` to see registered rigs.
2. For each rig, inspect the repository for issues:
   - Failing tests, open TODOs, stale dependencies, etc.
3. For each issue found, create a bead and route it to the appropriate agent:
   ```bash
   bd create "Fix failing auth tests" --rig my-project
   bd update <id> --set-metadata gc.routed_to=my-project/polecat
   ```
4. When discovery is complete, signal done:
   ```bash
   gc runtime drain-ack
   exit
   ```
```

### What happens after drain-ack

The scout calls `gc runtime drain-ack` and the controller stops the session on
its next tick. The named session has `mode = "always"`, so the controller will
restart it on the following tick — unless you want the scout to only run once,
in which case use `mode = "on_demand"` and suspend it after the first run.

The beads the scout created are now in the store with `gc.routed_to` metadata.
Pool agents see them on their next `scale_check` and spin up to handle them.
The scout and the workers never interact directly.

### Triggering the scout on demand

If you want the scout to run on a schedule or in response to an event rather
than continuously, suspend it by default and nudge it when you want a
discovery pass:

```toml
[[agent]]
name = "scout"
scope = "city"
prompt_template = "prompts/scout.md"
suspended = true         # starts dormant

[[named_session]]
template = "scout"
mode = "always"
```

```shell
# Resume the scout, let it run, and it will drain-ack when done
gc agent resume scout
```

The controller wakes the scout, it does its pass, drain-acks, and goes back to
sleep. The session bead stays in `asleep` state until you resume it again.

### Variation: scout as a one-shot polecat

If discovery is triggered by a specific event — a webhook, a human request, a
convoy step — you can sling directly to the scout instead of keeping it as a
named session:

```shell
gc sling scout "Scan my-project for open TODOs"
```

Gas City creates a transient polecat session for the scout agent, it runs the
discovery pass, drain-acks, and the session is cleaned up. This approach works
well when discovery is infrequent or event-driven rather than continuous.
