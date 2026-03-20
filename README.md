# PerigonForge Voxel Engine - Reference Guide

## Table of Contents
1. [Project Structure](#project-structure)
2. [Block System](#block-system)
3. [Hotbar System](#hotbar-system)
4. [World & Chunks](#world--chunks)
5. [Rendering](#rendering)
6. [Sky System](#sky-system)
7. [Settings](#settings)
8. [Input Controls](#input-controls)

---

## Project Structure

```
PerigonForge/
├── main/
│   ├── blockregister/
│   │   └── BlockRegistry.cs      # Block definitions & registration
│   ├── player/
│   │   └── Camera.cs              # Player camera & movement
│   ├── shaders/
│   │   └── Shader.cs              # OpenGL shader wrapper
│   ├── sky/
│   │   ├── SkyRenderer.cs         # Sky rendering
│   │   └── SkySystem.cs           # Sky time/weather
│   ├── systems/
│   │   ├── RaycastSystem.cs       # Block picking
│   │   └── SkySystem.cs           # Day/night cycle
│   ├── ui/
│   │   ├── BlockPreviewRenderer.cs # 3D block preview
│   │   ├── CrosshairRenderer.cs   # Crosshair
│   │   ├── FontRenderer.cs        # Text rendering
│   │   ├── TextRenderer.cs        # Text utilities
│   │   └── UIRenderer.cs          # UI elements
│   ├── worldgen/
│   │   ├── TerrainGenerator.cs    # Terrain generation
│   │   └── World.cs               # World management
│   └── src/
│       ├── Chunk.cs               # Chunk data
│       ├── ChunkRenderer.cs       # Chunk rendering
│       ├── Chunkoctree.cs         # Octree storage
│       ├── Game.cs                # Main game loop
│       ├── HotbarSystem.cs        # Inventory hotbar
│       ├── MeshBuilder.cs         # Mesh generation
│       ├── SelectionRenderer.cs   # Block selection
│       ├── Settings.cs            # Game settings
│       └── SVO.cs                 # Sparse voxel octree
├── Program.cs                     # Entry point
└── DOCUMENTATION.md              # This file
```

---

## Block System

### BlockType Enum (Chunk.cs:7)
Located in `main/src/Chunk.cs`

```csharp
public enum BlockType : byte 
{ 
    Air = 0,           // Empty air block
    Grass = 1,         // Grass block
    Dirt = 2,          // Dirt block  
    Stone = 3,         // Stone block
    Water = 4,         // Water (liquid)
    MapleLog = 5,      // Maple wood log
    MapleLeaves = 6,   // Green maple leaves
    YellowMapleLeaves  // Yellow autumn leaves
}
```

**How to add a new block:**
1. Add to `BlockType` enum in `Chunk.cs`
2. Register in `BlockRegistry.cs` Initialize() method

### BlockRegistry (BlockRegistry.cs)
Located in `main/blockregister/BlockRegistry.cs`

#### BlockDefinition Structure
```csharp
public struct BlockDefinition
{
    public string  Name;                   // Display name
    public Vector2i TopAtlasTile;         // Texture tile for top face
    public Vector2i BottomAtlasTile;     // Texture tile for bottom face
    public Vector2i SideAtlasTile;       // Texture tile for side faces
    public bool    UsesFlatColor;        // Use solid color instead of texture
    public Vector4 FlatColor;            // Solid color (RGBA, 0-1 range)
    public bool    IsSolid;              // Block has collision
    public bool    IsTransparent;        // Can see through block
    public bool    IsLiquid;             // Is a liquid (water)
    public bool    CanFall;              // Falls like sand
    public float   Buoyancy;             // How much it floats in water
    public bool    IsVisibleInInventory;  // Shows in hotbar
}
```

#### Key Functions

| Function | Syntax | Description |
|----------|--------|-------------|
| `Get` | `BlockRegistry.Get(int id)` | Get block definition by ID |
| `Get` | `BlockRegistry.Get(BlockType t)` | Get block definition by type |
| `IsSolid` | `BlockRegistry.IsSolid(BlockType t)` | Check if block has collision |
| `IsTransparent` | `BlockRegistry.IsTransparent(BlockType t)` | Check if block is transparent |
| `IsLiquid` | `BlockRegistry.IsLiquid(BlockType t)` | Check if block is liquid |
| `NeedsBlending` | `BlockRegistry.NeedsBlending(BlockType t)` | Needs alpha blending |
| `GetFlatColor` | `BlockRegistry.GetFlatColor(BlockType t)` | Get solid color |
| `GetOpacity` | `BlockRegistry.GetOpacity(BlockType t)` | Get opacity (0-1) |
| `IsVisibleInInventory` | `BlockRegistry.IsVisibleInInventory(BlockType t)` | Shows in hotbar |

#### Registering a New Block
```csharp
// In BlockRegistry.Initialize()
RegisterCore(BlockType.YourNewBlock, new BlockDefinition
{
    Name = "Your Block Name",
    UsesFlatColor = false,           // false = use texture, true = use flat color
    FlatColor = new Vector4(1, 0, 0, 1),  // Red color (if using flat color)
    TopAtlasTile = new Vector2i(0, 0),     // X, Y position in texture atlas
    BottomAtlasTile = new Vector2i(0, 0),
    SideAtlasTile = new Vector2i(0, 0),
    IsSolid = true,                  // Has collision
    IsTransparent = false,           // Can see through
    IsLiquid = false,                // Is water/lava
    CanFall = false,                 // Falls like sand
    Buoyancy = 0f,                   // Float in water (0-1)
    IsVisibleInInventory = true      // Shows in hotbar
});
```

### Texture Atlas
Located in `BlockRegistry.cs`

```csharp
// Atlas is 128x128 pixels with 32x32 tiles
// 4 tiles per row (0-3)
// Tile coordinates: new Vector2i(x, y) where x,y are 0-3
public const int AtlasSize = 128;
public const int TileSize = 32;
public const int TilesPerRow = 4;
```

---

## Hotbar System

### HotbarSystem (HotbarSystem.cs)
Located in `main/src/HotbarSystem.cs`

#### Properties
| Property | Type | Description |
|----------|------|-------------|
| `SlotsPerHotbar` | `int` | Number of slots (9) |
| `CurrentSlot` | `int` | Currently selected slot (0-8) |

#### Methods

| Method | Syntax | Description |
|--------|--------|-------------|
| `SwitchSlot` | `SwitchSlot(int slotIndex)` | Select slot 0-8 |
| `NextSlot` | `NextSlot()` | Cycle to next slot |
| `PreviousSlot` | `PreviousSlot()` | Cycle to previous slot |
| `GetSelectedBlock` | `GetSelectedBlock()` | Get selected block type |
| `GetBlock` | `GetBlock(int slotIndex)` | Get block at slot |
| `SetBlock` | `SetBlock(int slotIndex, BlockType?)` | Set block at slot |

#### Adding Blocks to Hotbar
```csharp
// In HotbarSystem.InitializeSlots()
slots[0] = BlockType.Grass;
slots[1] = BlockType.Dirt;
slots[2] = BlockType.Stone;
slots[3] = BlockType.MapleLog;
slots[4] = BlockType.MapleLeaves;
slots[5] = BlockType.YellowMapleLeaves;
// Slots 6-8 remain null (empty)
```

---

## World & Chunks

### Chunk Constants (Chunk.cs)
```csharp
public const int CHUNK_SIZE = 32;        // 32x32x32 blocks
public const int CHUNK_VOLUME = 32768;   // 32*32*32
```

### Chunk Properties
| Property | Type | Description |
|----------|------|-------------|
| `ChunkPos` | `Vector3i` | Chunk position in chunk coords |
| `WorldPosition` | `Vector3` | World position in blocks |
| `IsGenerated` | `bool` | Has terrain been generated |
| `IsDirty` | `bool` | Needs remeshing |
| `IndexCount3D` | `int` | Number of indices in opaque mesh |
| `VAO3D` | `int` | OpenGL VAO for opaque mesh |
| `HasTransparentMesh` | `bool` | Has transparent geometry |

### World Class (World.cs)
Located in `main/worldgen/World.cs`

#### Properties
| Property | Type | Description |
|----------|------|-------------|
| `RenderDistance` | `int` | How many chunks to load |
| `FullDetailDistance` | `int` | Full detail radius |
| `LoadedChunks` | `int` | Number of loaded chunks |
| `PendingGeneration` | `int` | Chunks waiting to generate |

#### Methods
| Method | Syntax | Description |
|--------|--------|-------------|
| `SetVoxel` | `SetVoxel(int x, int y, int z, BlockType)` | Set block at position |

---

## Rendering

### ChunkRenderer (ChunkRenderer.cs)
Located in `main/src/ChunkRenderer.cs`

#### Vertex Format (28 bytes per vertex)
- Position: 3 floats (x, y, z)
- Tile UV: 2 floats (atlas coordinates)
- UV/Face/AO: 1 packed uint
- Flat Color: 1 packed uint (RGBA)

#### Key Methods
| Method | Syntax | Description |
|--------|--------|-------------|
| `EnsureBuffers` | `EnsureBuffers(Chunk chunk)` | Upload mesh to GPU |
| `RenderChunksOpaque` | `RenderChunksOpaque(List<Chunk>, view, projection, camPos)` | Render solid blocks |
| `RenderChunksTransparent` | `RenderChunksTransparent(List<Chunk>, view, projection, camPos)` | Render water/transparent |
| `SetFog` | `SetFog(float start, float end)` | Set fog distance |

### MeshBuilder (MeshBuilder.cs)
Located in `main/src/MeshBuilder.cs`

Generates chunk meshes using greedy meshing algorithm.

#### Constants
```csharp
public const int STRIDE = 7;  // floats per vertex
// Face indices
public const byte F_PY = 0;   // +Y (top)
public const byte F_NY = 1;   // -Y (bottom)
public const byte F_PZ = 2;   // +Z (south)
public const byte F_NZ = 3;   // -Z (north)
public const byte F_PX = 4;   // +X (east)
public const byte F_NX = 5;   // -X (west)
```

---

## Sky System

### SkySystem (SkySystem.cs)
Located in `main/systems/SkySystem.cs`

#### Properties
| Property | Type | Description |
|----------|------|-------------|
| `CurrentSkyColor` | `Vector4` | RGBA sky color |
| `SunDirection` | `Vector3` | Sun position |
| `CloudTime` | `float` | Cloud animation time |
| `TimeOfDay` | `float` | Current time (0-1) |

#### Methods
| Method | Syntax | Description |
|--------|--------|-------------|
| `SetDayLength` | `SetDayLength(float seconds)` | Set full day duration |
| `UpdateSky` | `UpdateSky(float deltaTime)` | Update sky state |

### SkyRenderer (SkyRenderer.cs)
Located in `main/sky/SkyRenderer.cs`

Renders the skybox gradient.

---

## Settings

### Settings Class (Settings.cs)
Located in `main/src/Settings.cs`

#### Graphics Properties
| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `RenderDistance` | `int` | 7 | Chunk load radius |
| `FullDetailDistance` | `int` | 7 | Full detail radius |
| `VerticalRenderDistance` | `int` | 1 | Vertical chunks |
| `FogStart` | `float` | 50 | Fog start distance |
| `FogEnd` | `float` | 150 | Fog end distance |

#### Constants
```csharp
public const int MAX_RENDER_DISTANCE = 32;
public const int MAX_FULL_DETAIL_DISTANCE = 16;
```

---

## Input Controls

### Default Key Bindings
| Key | Action |
|-----|--------|
| W/S | Move forward/backward |
| A/D | Move left/right |
| Space | Jump (tap) / Fly toggle (double-tap) |
| Shift | Descend (in fly mode) |
| Ctrl | Sprint |
| Left Click | Break block |
| Right Click | Place block |
| 1-9 | Select hotbar slot |
| Mouse Wheel | Cycle hotbar |
| F8 | Toggle debug mode |
| F11 | Toggle fullscreen |
| ESC | Open/close settings |

### Changing Controls
In `Settings.cs`:
```csharp
public Keys KeyForward = Keys.W;
public Keys KeyBackward = Keys.S;
public Keys KeyLeft = Keys.A;
public Keys KeyRight = Keys.D;
public Keys KeyJump = Keys.Space;
public Keys KeyDescend = Keys.LeftShift;
public Keys KeySprint = Keys.LeftControl;
```

---

## Adding New Features

### Adding a New Block Type
1. Add to `BlockType` enum in `main/src/Chunk.cs`:
   ```csharp
   public enum BlockType : byte { ..., NewBlock }
   ```

2. Register in `main/blockregister/BlockRegistry.cs`:
   ```csharp
   RegisterCore(BlockType.NewBlock, new BlockDefinition
   {
       Name = "New Block",
       // ... properties
   });
   ```

3. Add texture to atlas in `Resources/Blocks/Atlas.png`

4. Add to hotbar in `main/src/HotbarSystem.cs`:
   ```csharp
   slots[slotIndex] = BlockType.NewBlock;
   ```

### Modifying Terrain Generation
In `main/worldgen/TerrainGenerator.cs`:
- `GenerateChunk(Chunk chunk)` - Main terrain generation
- `PlantTreesInChunk(Chunk chunk, World world)` - Tree placement

### Changing Fog/Sky Colors
In `main/src/ChunkRenderer.cs`:
```csharp
// Set fog
SetFog(start, end);
shader.SetVector3("uFogColor", new Vector3(r, g, b));
```

In `main/systems/SkySystem.cs`:
- Modify `CurrentSkyColor` getter for custom sky colors
- Adjust `GetCloudCoverage()` for cloud density

---

## Common Issues

### Block Not Showing in Inventory
1. Check `IsVisibleInInventory = true` in BlockDefinition
2. Ensure block is added to `HotbarSystem.InitializeSlots()`

### Block Not Rendering
1. Check `IsSolid` is set correctly
2. Verify texture coordinates in atlas
3. Check mesh generation in `MeshBuilder.cs`

### Game Crash on Start
1. Check shader compilation (console output)
2. Verify texture atlas exists at `Resources/Blocks/Atlas.png`
3. Check OpenGL context settings in `Game.cs`

---

## File Quick Reference

| File | Purpose |
|------|---------|
| `Chunk.cs` | BlockType enum, Chunk class |
| `BlockRegistry.cs` | Block definitions, textures |
| `HotbarSystem.cs` | Inventory slots |
| `World.cs` | Chunk management, generation |
| `ChunkRenderer.cs` | GPU rendering |
| `MeshBuilder.cs` | Mesh generation |
| `Game.cs` | Main loop, rendering |
| `Settings.cs` | Game settings |
| `Camera.cs` | Player movement |
| `SkySystem.cs` | Day/night cycle |
| `SkyRenderer.cs` | Sky rendering |
