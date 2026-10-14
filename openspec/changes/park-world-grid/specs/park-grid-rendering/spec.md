# Spec Delta

## Purpose

Defines how the park grid is drawn with Three.js and how a pointer position is turned into a tile: chunked terrain and path meshes, instanced props, analytic picking, ghost and highlight visuals, camera limits and performance budgets.

## ADDED Requirements

### Requirement: Chunked terrain and path meshes

The renderer SHALL draw terrain and paths in chunks of 16 x 16 tiles, one terrain mesh and one path mesh per chunk that contains any path. Terrain geometry SHALL come from the vertex levels (two triangles per tile, split along the diagonal that keeps Flat and Ramp tiles planar). On a grid change only chunks containing changed vertices or tiles SHALL be rebuilt. Path surfaces SHALL sit 0.02 m above the terrain and SHALL draw a kerb strip 0.1 m wide along every closed edge. Normal and Queue paths SHALL differ by a colour and a pattern.

#### Scenario: Only dirty chunks rebuild

- **WHEN** a delta changes one path tile in chunk `(2, 3)`
- **THEN** only that chunk's path mesh (and neighbours' meshes if the tile touches a chunk edge) is rebuilt

#### Scenario: Flat and ramp tiles are planar

- **WHEN** a tile is a Ramp rising East
- **THEN** its two triangles lie in one plane

#### Scenario: Kerb on closed edges

- **WHEN** a path tile has only its East and West edges open
- **THEN** kerb strips are drawn on its North and South edges only

### Requirement: Instanced props and placeholders

Objects without art SHALL be drawn as placeholder boxes sized to footprint and clearance and coloured by category, using one instanced mesh per definition id. Path additions SHALL be drawn as small instanced meshes at the tile centre. Instanced meshes SHALL grow by doubling capacity and SHALL update only changed instances. Articulated objects (scene nodes) SHALL be drawn through the scene mirror and SHALL NOT also appear as instanced placeholders.

#### Scenario: Ten benches, one draw call

- **WHEN** ten benches are placed
- **THEN** they are drawn by one instanced mesh with a count of 10

#### Scenario: Removal updates one instance

- **WHEN** one of those benches is removed
- **THEN** the count becomes 9 and no geometry is re-created

#### Scenario: No double drawing

- **WHEN** a definition marked as a scene node is placed
- **THEN** it appears once, through the scene mirror

### Requirement: Analytic grid picking

Picking SHALL convert a pointer ray into a tile by walking the grid cells the ray crosses (a digital differential analyser) and testing the two triangles of each tile against the terrain heights, returning the first hit tile, the hit point and the tile's corner levels. It SHALL NOT raycast against terrain meshes. A ray that misses the map, runs parallel above the terrain, or starts below it SHALL return no hit. Picking SHALL be exact for rays through tile corners and edges (a deterministic tie-break: the tile with the lower x, then lower y).

#### Scenario: Straight down

- **WHEN** a ray starts at world `(7.0, 50.0, 11.0)` pointing straight down over a flat park at level 8
- **THEN** it hits tile `(3, 5)` at height 2.0 m

#### Scenario: Oblique across a hill

- **WHEN** a ray passes over a raised vertex and descends behind it
- **THEN** the hit is on the near slope, not the hidden far tile

#### Scenario: Outside the map

- **WHEN** a ray points away from the park
- **THEN** no hit is returned

#### Scenario: Corner tie-break

- **WHEN** a vertical ray passes exactly through the shared corner of four tiles
- **THEN** the returned tile is the one with the lower x then lower y

### Requirement: Ghost and highlight visuals

The ghost SHALL show the footprint as an outline on the terrain surface plus a translucent box of the object's clearance, in a valid colour, an invalid colour or a neutral colour, with a hatched pattern for invalid. Hover and selection highlights SHALL outline the tile or object. The ghost and highlight objects SHALL be pooled and reused: moving the pointer SHALL not create geometry or materials.

#### Scenario: No allocation on move

- **WHEN** the pointer moves across 100 tiles
- **THEN** the number of geometries and materials in the scene is unchanged

#### Scenario: Footprint on slopes

- **WHEN** the ghost covers a Ramp tile
- **THEN** its outline follows the terrain surface rather than floating at one height

### Requirement: Camera limits and following the cursor

The park camera SHALL orbit a ground target with yaw free, pitch from 20 to 80 degrees and distance from 10 to 200 m. Mouse: middle-drag pans, right-drag orbits, wheel zooms. When the keyboard cursor leaves the visible region, the camera target SHALL move smoothly to bring it back, without motion when the user prefers reduced motion.

#### Scenario: Pitch clamp

- **WHEN** the user drags to pitch the camera to 5 degrees
- **THEN** the pitch is clamped to 20 degrees

#### Scenario: Cursor off screen

- **WHEN** the keyboard cursor is moved eight tiles beyond the view
- **THEN** the camera target moves so the cursor tile is visible

### Requirement: Rendering budgets

On a 128 x 128 park with 4000 path tiles, 200 props and 3 articulated rides, the renderer SHALL use at most 250 draw calls for the grid, at most 70 000 triangles for terrain plus paths, a chunk rebuild of at most 2 ms, and picking of at most 0.2 ms per ray. Per pointer move, renderer work SHALL be at most 1 ms. These are targets measured in a recorded run; Playwright or an in-page probe SHALL report them and a budget test SHALL fail when a target is exceeded by more than a factor of 2.

#### Scenario: Budget report

- **WHEN** the budget probe runs on the reference scene
- **THEN** it reports draw calls, triangles, chunk rebuild time and pick time and the report is stored in `docs/park/01-park-world-grid.md`

#### Scenario: Disposal

- **WHEN** the park view is destroyed
- **THEN** all geometries, materials and textures created by the renderer are disposed
