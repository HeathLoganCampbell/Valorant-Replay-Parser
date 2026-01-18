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
