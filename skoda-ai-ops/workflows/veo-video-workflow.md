# Workflow: Veo video workflow

> **Status:** outline draft.

## When to use Veo vs reuse a bed
- New variant launch, new model, new feature — generate.
- Routine "deal of the week" — reuse an existing bed (e.g. `*-short-raw.mp4`) with overlay text.

## Generation parameters
- Social Shorts: 9:16, native_audio=true, 8s base loop extended to 30s via composer.
- Website hero: 16:9, native_audio=true, single-take 8s.

## Interpolation (before/after)
- For "dirty car → clean car" or "old config → new config" — use `veo_interpolate_video` between two stills.

## Cost discipline
- Veo renders are the most expensive line item per minute. Re-use beds aggressively.
- When the Veo MCP returns 403/404, fall back to existing beds and overlay rather than retry.
