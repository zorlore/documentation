# OTBM2 Binary Spec (Current Runtime)

## Purpose

Defines the on-disk formats used by runtime and tooling for:
- map pack payload blobs (`OTB2`, version `1.2`) used in worldpack sections.
- named content packs (`OTB2`, version `2.0`) for actions/spells/monsters/npc/houses/objects/cities/raids.
- world container packs (`OTB2WPK`, version `1.0`): `data/world.otbm2` and `data/liveworld.otbm2`.
- users store (`OTB2USR`, version `1.0`): `data/users.otbm2`.

## Endianness

All multi-byte integers are little-endian.

## File Layout

1. `OTB2FileHeader`
2. `OTB2SectorIndexEntry[sector_index_entry_count]`
3. sector blobs (variable length)

## Header

```c
struct OTB2FileHeader {
  char magic[4];              // "OTB2"
  uint16_t version_major;     // 1
  uint16_t version_minor;     // 2 (authoritative current format)
  uint32_t header_size;       // 36
  uint32_t sector_index_entry_count;
  uint32_t offset_sector_index;
  uint32_t offset_sector_data;
  uint32_t offset_end;
  uint32_t reserved0;         // 0
  uint32_t reserved1;         // 0
};
```

Validation:
- `magic == "OTB2"`
- `header_size == sizeof(OTB2FileHeader)`
- offsets monotonic and in-file
- reserved fields are zero

## Index Entry

```c
struct OTB2SectorIndexEntry {
  int32_t sector_x;
  int32_t sector_y;
  int32_t sector_z;
  uint32_t sector_blob_offset;
  uint32_t sector_blob_length;
};
```

Rules:
- sector keys unique
- blob range inside `[offset_sector_data, offset_end)`
- no overlapping ranges

## Sector Blob (`version_minor = 2`)

```c
u32 point_count;
repeat point_count times {
  u8 offset_x;          // 0..31
  u8 offset_y;          // 0..31
  u8 tile_flags;        // bit0=Refresh, bit1=NoLogout, bit2=ProtectionZone
  u8 has_content;       // 0 or 1
  u32 content_length;   // bytes of legacy object-stream payload for this tile
  u8 content[content_length];
}
```

Notes:
- Tile order in blob is deterministic and preserved by converter.
- `content` payload uses legacy object stream encoding used by map loader logic.
- Full-sector packs store all non-empty/flagged tile points.

## Runtime Semantics

- Worldpack runtime is required.
- Startup enables runtime from `data/liveworld.otbm2` and fails if baseline `data/world.otbm2` is missing.
- If `data/liveworld.otbm2` is missing, runtime copies `data/world.otbm2` to `data/liveworld.otbm2`.
- Runtime always loads map state from worldpack section `2` (`MAP_LIVE`).
- Runtime reads pending patch keys from baseline worldpack section `3` (`MAP_DIRTY`) and applies those sectors from baseline section `1` (`MAP_ORIG`) into live map state.
- After patch apply, runtime clears `MAP_DIRTY` in both `data/liveworld.otbm2` and `data/world.otbm2`.
- Runtime map save writes only section `2` (`MAP_LIVE`) to `data/liveworld.otbm2`.
- On startup, all non-map sections are mirrored from `data/world.otbm2` into `data/liveworld.otbm2` (map sections `1`/`2` are excluded from mirroring).

## Compatibility Notes

Production map artifacts in worldpack sections should be written as `OTB2` `1.2`.

## Non-Map Named Packs (`version_major=2`, `version_minor=0`)

Used now for:
- `data/actions.otbm2`
- `data/spells.otbm2`
- `data/monsters.otbm2`
- `data/npc.otbm2`
- `data/houses.otbm2`
- `data/objects.otbm2`
- `data/cities.otbm2`
- `data/raids.otbm2`

Pack kind values:
- `1` = actions
- `2` = spells
- `3` = monsters
- `4` = npcs
- `5` = houses
- `6` = objects
- `7` = cities
- `8` = raids

Header:

```c
struct OTB2NamedPackHeader {
  char magic[4];              // "OTB2"
  uint16_t version_major;     // 2
  uint16_t version_minor;     // 0
  uint32_t header_size;       // 40
  uint32_t pack_kind;         // see values above
  uint32_t entry_count;
  uint32_t offset_index;
  uint32_t offset_data;
  uint32_t offset_end;
  uint32_t reserved0;         // 0
  uint32_t reserved1;         // 0
};
```

Index entry:

```c
struct OTB2NamedPackIndexEntry {
  uint32_t key_offset;        // absolute file offset
  uint32_t key_length;        // bytes, no trailing NUL
  uint32_t blob_offset;       // absolute file offset
  uint32_t blob_length;       // bytes
};
```

Envelope validation:
- `magic == "OTB2"`
- `version_major == 2`, `version_minor == 0`
- `header_size == sizeof(OTB2NamedPackHeader)`
- `pack_kind` matches loader expectation
- `reserved0 == 0`, `reserved1 == 0`
- offsets are monotonic and in-file
- every key/blob range is inside `[offset_data, offset_end)`
- all key and blob ranges are non-overlapping
- key length must be > 0
- keys must be unique
- pack must contain at least one entry

Primitive field encoding used by non-map blobs:
- `u8`, `u16`, `u32`: little-endian unsigned integers
- `i32`: two's complement little-endian signed integer
- `string`: `u32 byte_length` + raw bytes, no trailing NUL

## World Container Pack (`world.otbm2` / `liveworld.otbm2`)

The runtime can load a single world container file instead of individual map/named-pack files.

Container files:
- `data/world.otbm2`: baseline snapshot
- `data/liveworld.otbm2`: mutable live snapshot used at runtime

If `data/liveworld.otbm2` is missing but `data/world.otbm2` exists, runtime copies
`world.otbm2` to `liveworld.otbm2` and runs from `liveworld.otbm2`.

Header:

```c
struct TOtb2WorldPackHeader {
  char magic[8];              // "OTB2WPK\0"
  uint16_t version_major;     // 1
  uint16_t version_minor;     // 0
  uint32_t header_size;       // 40
  uint32_t entry_count;
  uint32_t reserved_a;        // 0
  uint64_t index_offset;
  uint64_t data_offset;
};
```

Index entry:

```c
struct TOtb2WorldPackIndexEntry {
  uint32_t section_id;
  uint32_t flags;             // currently 0
  uint64_t offset;            // absolute file offset
  uint64_t size;
  uint32_t crc32;             // currently 0
  uint32_t reserved;          // 0
};
```

Section IDs:
- `1`: map orig blob (`data/world/map.otbm2`)
- `2`: map live blob
- `3`: map dirty manifest (`u32 count` + repeated `i32 x,y,z`)
- `11`: actions named pack blob
- `12`: spells named pack blob
- `13`: monsters named pack blob
- `14`: NPC named pack blob
- `15`: houses named pack blob
- `16`: objects named pack blob
- `17`: cities named pack blob
- `18`: raids named pack blob
- `99`: map metadata key/value store (`MAP_METADATA`)

Rules:
- each section payload is the exact raw bytes of its original artifact.
- `build_worldpack` generation policy:
  - `world.otbm2`: section `2` is intentionally cloned from section `1`.
- `liveworld.otbm2`: section `2` uses `data/world/livemap.otbm2` when present (fallback to section `1`).
  - section `3` defaults to empty and is used only as explicit orig->live pending patch list.
- map sector index/offset layout is preserved because map blobs are stored verbatim.
- section IDs must be unique.
- runtime startup map load reads `MAP_LIVE`, then applies only sectors listed in section `3` from baseline `MAP_ORIG`.
- after startup patch application, runtime clears section `3`.
- runtime map save updates section `2` and flushes `data/liveworld.otbm2`.
- owners are standalone in `data/owners.sqlite` and are not embedded in worldpack sections.

Section `99` (`MAP_METADATA`) payload format:

```c
struct TMapMetadataHeader {
  char magic[8];              // "OTB2M99\0"
  uint32_t version;           // 1
  uint32_t entry_count;
};

// repeated entry_count times
struct TMapMetadataEntry {
  uint32_t key_len;           // >= 1
  uint32_t value_len;
  uint8_t key_bytes[key_len];     // raw bytes, not null-terminated
  uint8_t value_bytes[value_len]; // raw bytes, not null-terminated
};
```

Validation:
- `magic == "OTB2M99\0"`.
- `version == 1`.
- `key_len >= 1`.
- all reads are bounds-checked (`offset + len <= blob_size`).
- no trailing bytes are allowed after the final entry.

## Users Store (`data/users.otbm2`)

Player persistence is stored in `data/users.otbm2`, keyed by `CharacterID`.
Each entry payload is the exact legacy `.usr` blob bytes.

Header:

```c
struct TUsersHeader {
  char magic[8];              // "OTB2USR\0"
  uint16_t version_major;     // 1
  uint16_t version_minor;     // 0
  uint32_t header_size;       // 40
  uint32_t entry_count;
  uint32_t reserved_a;        // 0
  uint64_t index_offset;
  uint64_t data_offset;
};
```

Index entry:

```c
struct TUsersIndexEntry {
  uint32_t character_id;
  uint32_t flags;             // 0
  uint64_t offset;
  uint64_t size;
  uint32_t crc32;             // 0
  uint32_t reserved;          // 0
};
```

Initialization behavior:
- runtime: if `users.otbm2` is missing, server initializes it as empty.

### actions.otbm2 (kind=1)

Entry shape:
- `key = <relative action lua path from actions/actions.yaml>`
- `blob = <raw lua source bytes>`

Ordering:
- preserves `actions/actions.yaml` list order.

### spells.otbm2 (kind=2)

Entry shape:
- `key = <spell lua file name>`
- `blob = <raw lua source bytes>`

Ordering:
- lexicographically sorted file names from `data/spells/*.lua`.

### monsters.otbm2 (kind=3)

Entry shape:
- `key = <normalized monster name>`
  - normalized to lowercase/trimmed name.
  - if normalized-name collision occurs, converter uses `<name>#<race_id>`.
- `blob`:

```c
u32 version;                  // 1
u32 race_id;
u32 home_count;
repeat home_count times {
  u32 x;
  u32 y;
  u32 z;
  u32 radius;
  u32 max_monsters;
  u32 regeneration_time;
}
u32 lua_len;
u8 lua_bytes[lua_len];
```

Additional validation:
- `race_id` must be valid.
- `lua_len > 0`.
- no trailing blob bytes.
- `race_id` must be unique across entries.
- decoded home runtime fields `act_monsters` and `timer` initialize to `0`.

### npc.otbm2 (kind=4)

Pack entry types:
- `npc:<npc_key>`: NPC definition blob.
- `module:<module_name>`: module Lua blob.
- `helper:<helper_name>`: helper Lua blob.

At least one `npc:` entry is required.

NPC definition blob (`npc:*`):

```c
u32 version;                  // 1
string name;
string sex;                   // "male" | "female" (validated later by runtime)
u32 race_id;
string outfit_literal;        // canonical literal string
string home_literal;          // canonical literal string
u32 radius;
u32 go_strength;
u8 source_mode;               // 1=lua_blob, 2=lua_file_ref
string source_payload;        // blob bytes or relative path string
```

`source_mode` values:
- `1` (`DSL_NPC_SCRIPT_BLOB`) => `source_payload` is Lua source.
- `2` (`DSL_NPC_SCRIPT_FILE_REF`) => `source_payload` is relative script path.

Additional validation:
- only prefixes `npc:`, `module:`, `helper:` allowed.
- no trailing blob bytes.
- source mode must be `1` or `2`.
- keys after prefixes must be non-empty and unique per prefix class.
- NPC metadata is authoritative from blob fields (`name/sex/race/outfit/home/radius/go_strength`).
- NPC Lua registration payload must not contain `name` or `identity`; only handler registration data is allowed.

### houses.otbm2 (kind=5)

Pack entry types:
- `area:<id>`
- `house:<id>`

At least one `area:` and one `house:` entry are required.

Area blob (`area:*`):

```c
u32 version;                  // 1
u16 area_id;                  // >0
string name;
i32 sqm_price;
i32 depot;
```

House blob (`house:*`):

```c
u32 version;                  // 1
u16 house_id;                 // >0
string name;
string description;
i32 rent_offset;
u16 area_id;                  // >0
u8 guild_house;               // 0|1
i32 exit_x;
i32 exit_y;
i32 exit_z;
i32 center_x;
i32 center_y;
i32 center_z;
u32 field_count;              // >0
repeat field_count times {
  i32 x;
  i32 y;
  i32 z;
}
```

Additional validation:
- key id suffix must parse as positive decimal `<= 65535`.
- key id must equal blob id.
- `guild_house` must be `0` or `1`.
- no trailing blob bytes.
- area and house ids must be unique.

### objects.otbm2 (kind=6)

Entry shape:
- `key = object:<typeid>`
- `blob`:

```c
u32 version;                  // 1
u32 typeid;                   // <= INT_MAX
u8 has_name;                  // 0|1
string name;
u8 has_description;           // 0|1
string description;
u32 flag_count;
repeat flag_count times {
  string flag_name;
}
u32 attribute_count;
repeat attribute_count times {
  string attribute_name;
  u32 attribute_value;
}
```

Additional validation:
- key prefix must be `object:`.
- key typeid must parse as non-negative decimal integer.
- key typeid must equal blob `typeid`.
- `has_name` and `has_description` must be `0` or `1`.
- no trailing blob bytes.
- object typeids must be unique.

### cities.otbm2 (kind=7)

Pack entry types:
- `start:newbie`
- `start:veteran`
- `mark:<zero-padded index>`
- `depot:<id>`

Start blob (`start:*`):

```c
u32 version;                  // 1
i32 x;
i32 y;
i32 z;
```

Mark blob (`mark:*`):

```c
u32 version;                  // 1
string name;
i32 x;
i32 y;
i32 z;
```

Depot blob (`depot:*`):

```c
u32 version;                  // 1
i32 id;
string town;
i32 size;
```

Additional validation:
- exactly one `start:newbie` and one `start:veteran` entry must exist.
- mark names must be non-empty and fit runtime `TMark::Name` (max 19 bytes + NUL).
- depot ids must be unique and satisfy `0 <= id < MAX_DEPOTS`.
- depot town names must be non-empty and fit runtime `TDepotInfo::Town` (max 19 bytes + NUL).
- depot `size` must be `> 0`.
- `depot:<id>` key suffix must match blob `id`.

### raids.otbm2 (kind=8)

Pack entry types:
- `raid:<zero-padded index>`

Raid blob (`raid:*`):

```c
u32 version;                  // 1
string name;
u8 has_type;
u8 big_raid;
u8 has_date;
u8 date_is_holiday;
i32 date;
string holiday;
u8 has_utc_time;
i32 utc_hour;
i32 utc_minute;
i32 interval;
i32 duration;
u32 wave_count;
repeat wave_count times {
  i32 delay;
  i32 x;
  i32 y;
  i32 z;
  i32 spread;
  i32 race;
  i32 min_count;
  i32 max_count;
  i32 radius;
  i32 lifetime;
  string message;
  u32 inventory_count;
  repeat inventory_count times {
    i32 type;
    i32 maximum;
    i32 probability;
  }
}
```

Additional validation:
- key prefix must be `raid:`.
- key suffix must be a non-negative decimal index and unique.
- boolean fields are strictly `0|1`.
- blob version must be `1`.
- no trailing bytes in any blob.
