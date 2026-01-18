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
