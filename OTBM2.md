# OTBM2 Format Reference (For Third-Party Developers)

This document describes the current OTBM2 wire formats used across ZorLore runtime and tools.

It is written for developers implementing readers/writers, converters, validators, or custom pipeline tooling.

## Scope

ZorLore uses multiple binary containers under the OTBM2 family:

- `OTB2` map pack payloads (version `1.2`)
- `OTB2` named content packs (version `2.0`)
- `OTB2WPK` world container packs (version `1.0`)
- `OTB2USR` users store (version `1.0`)

All multi-byte integers are little-endian.

## Primitive Types

- `u8`, `u16`, `u32`, `u64`: unsigned little-endian integers
- `i32`: signed little-endian integer (two’s complement)
- `string`: `u32 byte_length` followed by raw bytes (no trailing NUL)

---

## 1) `OTB2` Map Pack (`version 1.2`)

### Header

```c
struct OTB2FileHeader {
  char magic[4];              // "OTB2"
  uint16_t version_major;     // 1
  uint16_t version_minor;     // 2
  uint32_t header_size;       // 36
  uint32_t sector_index_entry_count;
  uint32_t offset_sector_index;
  uint32_t offset_sector_data;
  uint32_t offset_end;
  uint32_t reserved0;         // 0
  uint32_t reserved1;         // 0
};
```

### Index Entry

```c
struct OTB2SectorIndexEntry {
  int32_t sector_x;
  int32_t sector_y;
  int32_t sector_z;
  uint32_t sector_blob_offset;
  uint32_t sector_blob_length;
};
```

### Sector Blob (`v1.2`)

```c
u32 point_count;
repeat point_count times {
  u8 offset_x;          // 0..31
  u8 offset_y;          // 0..31
  u8 tile_flags;        // bit0=Refresh, bit1=NoLogout, bit2=ProtectionZone
  u8 has_content;       // 0|1
  u32 content_length;
  u8 content[content_length]; // legacy object-stream bytes
}
```

### Validation Rules

- `magic == "OTB2"`
- `header_size == 36`
- offsets monotonic and in-file
- reserved fields are zero
- sector keys are unique
- index blob ranges are in `[offset_sector_data, offset_end)`
- sector blob ranges are non-overlapping

---

## 2) `OTB2` Named Pack (`version 2.0`)

Named packs are used for actions/spells/monsters/npcs/houses/objects/cities/raids.

### Header

```c
struct OTB2NamedPackHeader {
  char magic[4];              // "OTB2"
  uint16_t version_major;     // 2
  uint16_t version_minor;     // 0
  uint32_t header_size;       // 40
  uint32_t pack_kind;
  uint32_t entry_count;
  uint32_t offset_index;
  uint32_t offset_data;
  uint32_t offset_end;
  uint32_t reserved0;         // 0
  uint32_t reserved1;         // 0
};
```

### Index Entry

```c
struct OTB2NamedPackIndexEntry {
  uint32_t key_offset;        // absolute offset
  uint32_t key_length;
  uint32_t blob_offset;       // absolute offset
  uint32_t blob_length;
};
```

### Pack Kinds

- `1` = actions
- `2` = spells
- `3` = monsters
- `4` = npcs
- `5` = houses
- `6` = objects
- `7` = cities
- `8` = raids

### Validation Rules

- `magic == "OTB2"`
- `version_major == 2 && version_minor == 0`
- `header_size == 40`
- `pack_kind` matches expected loader kind
- reserved fields are zero
- key/blob ranges are in `[offset_data, offset_end)`
- no overlapping key/blob ranges
- keys are unique and non-empty

### Entry Payloads

Each entry has:

- key bytes at `[key_offset, key_offset + key_length)`
- blob bytes at `[blob_offset, blob_offset + blob_length)`

Blob schema is pack-specific (monsters, npcs, raids, etc).  
For full field-level schemas, use the canonical spec:

- `documentation/OTBM2_BINARY_SPEC.md`

---

## 3) `OTB2WPK` World Container (`version 1.0`)

World containers bundle map + named pack payloads into a single file:

- `data/world.otbm2` (baseline)
- `data/liveworld.otbm2` (runtime mutable)

### Header

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

### Index Entry

```c
struct TOtb2WorldPackIndexEntry {
  uint32_t section_id;
  uint32_t flags;             // currently 0
  uint64_t offset;            // absolute
  uint64_t size;
  uint32_t crc32;             // currently 0
  uint32_t reserved;          // 0
};
```

### World Section IDs

- `1`  = `MAP_ORIG` (`OTB2` map blob)
- `2`  = `MAP_LIVE` (`OTB2` map blob)
- `3`  = `MAP_DIRTY` manifest
- `11` = actions (`OTB2` named pack kind 1)
- `12` = spells (`OTB2` named pack kind 2)
- `13` = monsters (`OTB2` named pack kind 3)
- `14` = npcs (`OTB2` named pack kind 4)
- `15` = houses (`OTB2` named pack kind 5)
- `16` = objects (`OTB2` named pack kind 6)
- `17` = cities (`OTB2` named pack kind 7)
- `18` = raids (`OTB2` named pack kind 8)
- `99` = map metadata block (`OTB2M99\0`, format below)

### Section 3 (`MAP_DIRTY`) Payload

```c
u32 count;
repeat count times {
  i32 sector_x;
  i32 sector_y;
  i32 sector_z;
}
```

### Section 99 (`MAP_METADATA`) Payload

```c
char magic[8] = "OTB2M99\0";
u32 version;          // 1
u32 entry_count;
repeat entry_count times {
  u32 key_len;        // >= 1
  u32 value_len;
  u8 key[key_len];
  u8 value[value_len];
}
```

Validation:

- magic must match
- version must be `1`
- no empty keys
- all reads must stay within blob bounds
- no trailing bytes after final entry

---

## 4) `OTB2USR` Users Store (`version 1.0`)

`data/users.otbm2` stores user records keyed by character ID.

### Header

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

### Index Entry

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

---

## Implementation Notes

### Determinism

For build tooling, deterministic output is recommended:

- stable ordering
- strict offset/index regeneration
- byte-identical rebuilds for unchanged input

### Bounds Safety

Always validate:

- offset + size overflow
- range inclusion in payload area
- index uniqueness (keys/sections/sectors)

### Runtime Behavior (World Container)

Current runtime semantics:

- startup map state is loaded from section `2` (`MAP_LIVE`)
- section `3` drives selective orig->live patching from section `1`
- section `2` is what runtime save updates

---

## Canonical Low-Level Spec

For exhaustive blob schemas of all named packs (actions/spells/monsters/npcs/houses/objects/cities/raids), use:

- `documentation/OTBM2_BINARY_SPEC.md`

This file (`documentation/OTBM2.md`) is the practical implementer overview for third-party integrations.
