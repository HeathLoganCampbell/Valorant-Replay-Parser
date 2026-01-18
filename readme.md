# Valorant Replay Parser
```
ReplayFile
├── Header
├── Info / Metadata
├── ChunkIndex
├── EventIndex
├── CheckpointIndex
└── Chunks[]
```

```
ChunkIndexEntry {
  StartTime
  EndTime
  FileOffset
  CompressedSize
  UncompressedSize
}
```

```
EventEntry {
  Time
  Group
  Metadata
  PayloadOffset
}
```

This is why valorant have to fast forward to a spot cause each chunk is 5 - 15 seconds
```
CheckpointEntry {
  Time
  ChunkIndex
  OffsetInChunk
}
```

```
Chunk
├── [Compressed Block]
│   └── ReplayRecords[]
```

```
ReplayRecord {
  TimeDelta
  RecordType
  PayloadSize
  PayloadBits
}
```
```
ReplicationRecord
└── NetPackets[]
    └── NetBunches[]
```
```
ReplicatedObject
├── NetHandle
├── ReplicationFragments[]
│   ├── FragmentID
│   ├── FieldHandles[]
│   └── StateDeltaBits
```

```
Replay (file)
 └─ Chunk[]
    └─ Record[]
       └─ Packet[]
          └─ Bunch[]
             └─ ReplicationFragment[]
                └─ FieldDeltaBits
```

checkpoints comtain the full game state 
```
[Checkpoint Header]
├── Checkpoint ID/Index
├── Time offset (milliseconds into replay)
├── Size of checkpoint data
└── Checkpoint state data offset

[Checkpoint Data]
├── Actor GUID mappings
├── Deleted actors list
├── Active actors and their state:
│   ├── Actor GUID
│   ├── Actor archetype/class
│   ├── Replicated properties (serialized):
│   │   ├── Location (FVector)
│   │   ├── Rotation (FRotator)
│   │   ├── Velocity
│   │   ├── Health
│   │   └── [Game-specific properties]
│   └── Components and their states
└── Game state snapshot
```

## UE5

```
using System;
using System.Collections.Generic;

/// <summary>
/// Container class for Unreal Engine Replay Data Structures.
/// Converted from C++ structs found in LocalFileNetworkReplayStreaming.h, InMemoryNetworkReplayStreaming.h, and ReplayTypes.h.
/// </summary>
public class UnrealReplayData
{
    // =========================================================================
    // Enums
    // =========================================================================

    public enum ELocalFileChunkType
    {
        Unknown,
        Header,
        ReplayData,
        Checkpoint,
        Event,
        // Add other internal types as needed
    }

    // =========================================================================
    // Common Replay Types (Engine/Source/Runtime/Engine/Public/ReplayTypes.h)
    // =========================================================================

    public class PlaybackPacket
    {
        public byte[] Data { get; set; }
        public float TimeSeconds { get; set; }
        public int LevelIndex { get; set; }
        public uint SeenLevelIndex { get; set; }
    }

    // =========================================================================
    // Local File Streaming (LocalFileNetworkReplayStreaming.h)
    // =========================================================================

    /// <summary>
    /// Struct to hold chunk metadata for file-based replays
    /// </summary>
    public class LocalFileChunkInfo
    {
        public ELocalFileChunkType ChunkType { get; set; }
        public int SizeInBytes { get; set; }
        public long TypeOffset { get; set; }
        public long DataOffset { get; set; }
    }

    /// <summary>
    /// Struct to hold replay data chunk metadata
    /// </summary>
    public class LocalFileReplayDataInfo
    {
        public int ChunkIndex { get; set; }
        public uint Time1 { get; set; }
        public uint Time2 { get; set; }
        public int SizeInBytes { get; set; }
        public int MemorySizeInBytes { get; set; }
        public long ReplayDataOffset { get; set; }
        public long StreamOffset { get; set; }
    }

    /// <summary>
    /// Main header info for a locally stored replay file
    /// </summary>
    public class LocalFileReplayInfo
    {
        public int LengthInMS { get; set; }
        public uint NetworkVersion { get; set; }
        public uint Changelist { get; set; }
        public string FriendlyName { get; set; }
        public DateTime Timestamp { get; set; } // Converted from FDateTime
        public long TotalDataSizeInBytes { get; set; }
        public bool IsLive { get; set; }
        public bool IsValid { get; set; }
        public bool IsCompressed { get; set; }
        public bool IsEncrypted { get; set; }
        public byte[] EncryptionKey { get; set; }

        public int HeaderChunkIndex { get; set; }

        public List<LocalFileChunkInfo> Chunks { get; set; } = new List<LocalFileChunkInfo>();
        public List<LocalFileReplayDataInfo> DataChunks { get; set; } = new List<LocalFileReplayDataInfo>();
        
        // Note: Checkpoints and Events use FLocalFileEventInfo which is structurally similar to chunks 
        // but typically contains metadata strings and time info.
        public List<LocalFileEventInfo> Checkpoints { get; set; } = new List<LocalFileEventInfo>();
        public List<LocalFileEventInfo> Events { get; set; } = new List<LocalFileEventInfo>();
    }

    public class LocalFileEventInfo 
    {
        public string Id { get; set; }
        public string Group { get; set; }
        public string Metadata { get; set; }
        public uint Time1 { get; set; }
        public uint Time2 { get; set; }
        public int SizeInBytes { get; set; }
        public long EventDataOffset { get; set; }
    }

    // =========================================================================
    // In-Memory Streaming (InMemoryNetworkReplayStreaming.h)
    // =========================================================================

    public class InMemoryCheckpoint
    {
        public byte[] Data { get; set; }
        public uint TimeInMS { get; set; }
        public uint StreamByteOffset { get; set; }

        public void Reset()
        {
            Data = null;
            TimeInMS = 0;
            StreamByteOffset = 0;
        }
    }

    /// <summary>
    /// Represents a chunk of replay stream data between two checkpoints.
    /// </summary>
    public class InMemoryStreamChunk
    {
        public int StartIndex { get; set; }
        public uint TimeInMS { get; set; }
        public byte[] Data { get; set; }
    }

    /// <summary>
    /// Main class for a replay held entirely in memory (e.g., Kill Cam)
    /// </summary>
    public class InMemoryReplay
    {
        public byte[] Header { get; set; }
        public List<InMemoryStreamChunk> StreamChunks { get; set; } = new List<InMemoryStreamChunk>();
        public byte[] Metadata { get; set; }
        public List<InMemoryCheckpoint> Checkpoints { get; set; } = new List<InMemoryCheckpoint>();
        
        public NetworkReplayStreamInfo StreamInfo { get; set; }
        public uint NetworkVersion { get; set; }

        public long TotalStreamSize()
        {
            long totalSize = 0; // Base size approximation
            if (Header != null) totalSize += Header.Length;
            if (Metadata != null) totalSize += Metadata.Length;

            foreach (var chunk in StreamChunks)
            {
                if (chunk.Data != null) totalSize += chunk.Data.Length;
            }

            foreach (var cp in Checkpoints)
            {
                if (cp.Data != null) totalSize += cp.Data.Length;
            }

            return totalSize;
        }
    }

    public class NetworkReplayStreamInfo
    {
        public string Name { get; set; }
        public string FriendlyName { get; set; }
        public DateTime Timestamp { get; set; }
        public int LengthInMS { get; set; }
        public int NumViewers { get; set; }
        public bool IsLive { get; set; }
        public int Changelist { get; set; }
    }
}
```


## Riot's changes maybe?

This document describes a **streaming-first, memory-efficient architecture** for parsing VALORANT replay (`.vrf`) files, modeled after Riot’s likely internal Unreal Engine replay pipeline.

The design prioritizes fast startup, lazy decoding, scalability, and accurate timeline seeking.

---

## Design Goals

- **Fast startup**: Minimal header parsing (<1 ms)
- **Low memory usage**: Stream chunks instead of loading entire files
- **Random access**: Optional chunk and checkpoint indexing
- **Scalable**: Handles large replays (50–100 MB+)
- **Seekable**: Timeline scrubbing via checkpoints
- **Incremental enrichment**: Indexes built during playback

---

## High-Level Architecture

```text
Replay File
 ├─ Header (small, parsed eagerly)
 ├─ Chunk Stream (lazy)
 │   ├─ Oodle-compressed replay data
 │   ├─ Event blocks
 │   ├─ Checkpoints (full state snapshots)
 │   └─ Metadata / padding
 └─ Optional indexes
     ├─ Chunk index
     ├─ Event index
     └─ Checkpoint index
```

---

## Core Parser Entry Point

```csharp
public class ValorantReplayParser
{
    public ValorantReplay Parse(string filePath)
    {
        using var stream = File.OpenRead(filePath);
        using var reader = new BinaryReader(stream);

        var replay = new ValorantReplay();

        replay.Header = ReadValorantHeader(reader);
        replay.ChunkStream = new LazyChunkStream(stream, replay.Header);

        if (needsRandomAccess)
            replay.ChunkStream.BuildChunkIndex();

        if (needsEvents)
            replay.Events = new EventIndexBuilder().BuildEventIndex(stream);

        if (needsCheckpoints)
            replay.Checkpoints = new CheckpointSeeker(replay.ChunkStream)
                .BuildCheckpointIndex();

        return replay;
    }
}
```

---

## 1. Minimal Header Parsing (Fast Path)

Only essential metadata is parsed up-front to enable streaming.

Parsed fields:

- Magic (`0x43F4EFDD`)
- File version
- Custom version table
- Match duration (ms)
- Network session GUID
- Network version
- Changelist
- Friendly name (FString)
- Chunk stream offset

No replay payload data is loaded during this phase.

---

## 2. Lazy Chunk Streaming

Chunks are streamed sequentially on demand.

```csharp
foreach (var chunk in replay.ChunkStream.StreamChunks())
{
    ProcessChunk(chunk);
}
```

### Rationale

- Replay files are large
- Checkpoints decompress to tens of MB
- Streaming avoids unnecessary allocations and GC pressure

---

## 3. Chunk Indexing (Optional)

Chunk indexing is only built if random access is required.

The index maps:

```text
ChunkOffset → ChunkMetadata
```

Indexing heuristics:

- Oodle Kraken marker (`0x8C`)
- Compression boundaries
- Chunk size patterns

Use cases:

- Timeline scrubbing
- Checkpoint seeking
- Random chunk access

---

## 4. Chunk Type Detection

Chunk type is inferred using **byte-pattern heuristics**.

| Pattern | Meaning |
|------|--------|
| `0x8C` | Oodle Kraken compressed |
| `00 00 80 BF` | Alternate Oodle variant |
| Event string markers | Replay events |
| Large compressed blocks | Likely checkpoints |
| Small blocks | Metadata or padding |

This mirrors Unreal Engine replay streaming behavior.

---

## 5. Oodle Decompression (Critical)

All meaningful replay data is Oodle-compressed.

```csharp
byte[] decompressed = Oodle.OodleLZ_Decompress(...);
```

Notes:

- Riot uses **Oodle Kraken**
- Decompressed size is not always stored explicitly
- Output buffer size is estimated or derived heuristically
- Requires native Oodle DLL integration

---

## 6. Network Packet Parsing

Decompressed replay data is parsed as an Unreal network stream.

Each packet contains:

- Timestamp
- NetGUID exports
- Actor replication bunches
- Property updates
- RPC calls

This phase reconstructs **actual game state**.

---

## 7. Event Extraction

Events are discovered via string scanning.

Example pattern:

```text
EReplayEventGroup::EventName
```

Extracted fields:

- Event name
- Offset
- Timestamp (float near marker)

Used for:

- Kill events
- Round transitions
- Objective updates
- UI-level annotations

---

## 8. Checkpoints & Timeline Seeking

Checkpoints represent **full game state snapshots**.

Heuristics for detection:

- Large compressed size (>50 KB)
- Regular spacing (~30 seconds)
- Contains full actor state

Seek algorithm:

1. Find nearest checkpoint ≤ target time
2. Load checkpoint state
3. Replay incremental packets until target timestamp

This matches Unreal Engine replay seek semantics.

---

## 9. Playback Flow

```csharp
foreach (var chunk in replay.ChunkStream.StreamChunks())
{
    var decompressed = OodleDecompress(chunk.Data);
    var packets = ParseNetworkStream(decompressed);

    foreach (var packet in packets)
    {
        ApplyPacketToGameState(gameState, packet);
        RenderFrame(gameState, packet.TimeSeconds);
    }
}
```

---

## Riot-Style Optimizations

### Lazy Everything
- No eager loading
- No eager decompression

### Chunk Caching
- Cache only frequently accessed chunks (checkpoints)
- Evict via FIFO or LRU

### Incremental Indexing
- Build event and checkpoint indexes during playback
- Zero upfront indexing cost

### Memory Caps
- Hard limit on decompressed cache size
- Prevents runaway memory usage

---

## Summary

This architecture reflects how Riot likely processes VALORANT replays internally:

- Streaming-first design
- Oodle-centric compression
- Network replication decoding
- Checkpoint-driven seeking
- Incremental indexing

It is suitable for replay analysis, visualization tools, and deep reverse-engineering workflows.

## break down
```
OFFSET   | BYTES           | VALUE       | MEANING
---------|-----------------|-------------|---------------------------
0x00-0x03| DD EF F4 43     | 0x43F4EFDD  | Magic Number ✓
0x04-0x07| 07 00 00 00     | 7           | Version Major
0x08-0x0B| 01 00 00 00     | 1           | Version Minor  
0x0C-0x0F| 3E F0 A4 95     | 0x95A4F03E  | File Checksum
---------|-----------------|-------------|---------------------------
         RESERVED SECTION (32 bytes) - Unknown Purpose
---------|-----------------|-------------|---------------------------
0x10-0x13| E4 49 0B 7E     | 0x7E0B49E4  | Unknown metadata
0x14-0x17| 56 D3 43 BA     | 0xBA43D356  | Unknown metadata
0x18-0x1B| D9 87 FF 94     | 0x94FF87D9  | Unknown metadata
0x1C-0x1F| 07 00 00 00     | 7           | Version? (matches major)
0x20-0x23| C9 13 09 00     | 594,889     | Unknown counter/timestamp?
0x24-0x27| E6 EF A7 1C     | 0x1CA7EFE6  | Unknown metadata
0x28-0x2B| CD 6F 3E 00     | 4,091,853   | Unknown counter?
0x2C-0x2F| FF FE FF FF     | 0xFFFFFEFF  | Padding/separator
---------|-----------------|-------------|---------------------------
0x30+    | GUID field starts here (UTF-16LE, 512 bytes)
```

