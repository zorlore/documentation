# ZorLore Editor

ZorLore Editor is the world-building workspace of the platform.  
It lets MMO builders shape gameplay and visual identity directly in a live map/content pipeline.

## What It Is

The editor is a full world editor for building MMORPG content, including:

- map editing and terrain shaping
- monster and NPC placement/configuration
- raids and spawn management
- items and object metadata editing
- sprite editing (including multi-part appearances and sheet import workflows)
- actions/spells script editing
- cities/houses/depot tooling

In short: you can design both the mechanics and the look-and-feel of your world from one interface.

## Download

Download editor releases from:

- `https://github.com/zorlore/zorlore-editor/releases`

## Default Startup Behavior

If you start the editor without a `--file` argument, it uses the default world path:

- `./data/world.otbm2`

If required assets are missing, the bootstrap flow fetches missing OTBM2 assets/world data and starts from the provided example world dataset.

## Run

From the editor project/release directory:

```bash
./zorlore-editor --mode gui --file ./data/world.otbm2
```

You can also omit `--file` to use default startup behavior:

```bash
./zorlore-editor --mode gui
```

## Notes

- The editor is designed for iterative world creation: edit, preview, save, and immediately continue.
- It is intended to work alongside Bridge + Game Server in a production flow.

