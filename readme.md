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
