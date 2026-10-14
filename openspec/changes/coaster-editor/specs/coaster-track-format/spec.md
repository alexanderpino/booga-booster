# Spec Delta

## Purpose

Defines the file format that stores a coaster design: a small, versioned, byte-for-byte deterministic document with a content hash, the rules for reading old and damaged files, and the starter fixtures that workshops and tests share.

## ADDED Requirements

### Requirement: A design is stored as canonical JSON (Tier 1)

A design file SHALL be a UTF-8 JSON object without byte order mark and without whitespace between tokens, with the members in this order: `format` ("bb-coaster"), `version` (integer), `name`, `rideType`, `anchor` (`x`, `z`, `h`, `heading`, all integers), `train` (`cars`, `seatsPerCar`), `pieces` (array of piece id strings in design order), `hash`. It SHALL contain no floating-point number and no timestamp. Strings SHALL escape only `"`, `\` and control characters. Writing the same design SHALL give identical bytes on every run, culture and platform.

#### Scenario: Canonical bytes

- **WHEN** the same design is written twice under different current cultures
- **THEN** the byte sequences are identical

#### Scenario: Member order

- **WHEN** a file is written
- **THEN** its bytes begin with `{"format":"bb-coaster","version":1,"name":` and the members `rideType`, `anchor`, `train`, `pieces`, `hash` follow in that order

### Requirement: The hash identifies the content (Tier 1)

The `hash` member SHALL be the lower-case hexadecimal SHA-256 of the canonical bytes of the document without the `hash` member. When reading, a file with no hash SHALL be accepted and hashed; a file whose hash does not match SHALL be rejected with `HashMismatch` unless the caller passes the explicit allow-edited option, in which case the hash is recomputed. The design hash SHALL be the same value the test run, ratings and track versions refer to.

#### Scenario: Tampering

- **WHEN** one piece id is changed in a file that keeps its old hash
- **THEN** reading fails with `HashMismatch`

#### Scenario: Hand-written file

- **WHEN** a file without a `hash` member is read
- **THEN** the design loads and its hash is the SHA-256 of its canonical bytes

### Requirement: Reading is strict and atomic (Tier 1)

Reading SHALL reject a file larger than 262,144 bytes, with more than 256 pieces, with an unknown `format`, with a malformed anchor or train configuration, with an unknown or disabled piece id, or with a version newer than the supported one, each with an error that names the member or piece index. A rejected file SHALL NOT create or change any design. Piece connection problems SHALL load and be reported by validation, not rejected.

#### Scenario: Unknown piece

- **WHEN** a file lists the piece `loop:vertical` while the maximum tier is 1
- **THEN** reading fails naming index and id and stating that the piece's tier is not enabled

#### Scenario: Newer version

- **WHEN** a file has `version` 2 and the supported version is 1
- **THEN** reading fails with a message naming both versions

#### Scenario: Oversize

- **WHEN** a file is 300,000 bytes
- **THEN** it is rejected before parsing

### Requirement: Older versions are migrated by a chain (Tier 1)

Each format version change SHALL ship a pure migration from the previous version, applied in order from the file's version to the current one, followed by a recomputation of the hash. A migration SHALL be covered by a fixture of the old format whose migrated output is checked in. Version 1 is the first version; the chain machinery SHALL be tested with a test-only version 0 that lacks the `train` member and migrates to 4 cars of 2 seats.

#### Scenario: Migration chain

- **WHEN** the test-only version 0 file is read
- **THEN** it becomes a version 1 design with 4 cars of 2 seats and a computed hash

### Requirement: Round trips are lossless (Tier 1)

For every valid design, reading the written file SHALL give a design equal to the original in anchor, train configuration, name and piece list, and writing it again SHALL give identical bytes. A property test SHALL check 500 seeded random designs.

#### Scenario: Property

- **WHEN** 500 seeded random designs are written, read and written again
- **THEN** the two byte sequences of each design are identical

### Requirement: The starter coaster closes and is the reference (Tier 1)

The project SHALL ship the fixture `starter-coaster.v1.json`: anchor (0, 0, 0, heading 0), 4 cars of 2 seats, and 51 pieces: `station`; `pitch:0>+2+lift`; `slope:+2+lift` x4; `pitch:+2>-2+lift`; `slope:-2` x4; `pitch:-2>0`; `curve:L4`; `straight` x2; `curve:L4`; `straight` x24; `curve:L4`; `straight` x2; `curve:L4`; `straight` x3; `brake:trim` x4. Compiled, it SHALL close (end frame equals start frame) and have length 313.27 m within 0.01 m.

#### Scenario: Closure by arithmetic

- **WHEN** the starter is compiled
- **THEN** the end position is (0, 0, 0) within 1e-9 m with heading 0 and slope class 0, and the highest centreline point is 11.0 m

#### Scenario: Validates clean

- **WHEN** the starter is validated against an empty park of 40 x 30 tiles with the anchor at tile (12, 12)
- **THEN** there are no error findings and no warning

### Requirement: Workshop variants share the starter's closure (Tier 1)

The project SHALL ship `starter-tall-hill.v1.json` (the 24 `straight` pieces of the return leg replaced by `straight` x8 followed by `pitch:0>+2`, `slope:+2` x5, `pitch:+2>-2`, `slope:-2` x5, `pitch:-2>0`) and `starter-tight-curves.v1.json` (every `curve:L4` replaced by `curve:L2`). Both SHALL close and validate without error.

#### Scenario: Tall hill closes

- **WHEN** `starter-tall-hill` is compiled
- **THEN** it closes within 1e-9 m and its second hill has an apex of 13.0 m

#### Scenario: Tight curves close

- **WHEN** `starter-tight-curves` is compiled
- **THEN** it closes within 1e-9 m because each curve contributes the same offset to opposite legs

### Requirement: The file is the only persistent form of a design (Tier 1)

Designs live in memory in the server; export SHALL write the file and import SHALL read it. The system SHALL state in the design response that designs are not persisted across restarts unless exported or built into a park that is itself saved. A park save SHALL embed the design file of each built coaster, so loading a park rebuilds the same track hash.

#### Scenario: Park load

- **WHEN** a park containing a built coaster is saved and loaded
- **THEN** the loaded coaster has the same track hash as before the save
