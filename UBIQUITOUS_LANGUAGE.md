# Ubiquitous Language

The words this product is about. Every other doc (DESIGN.md, TESTING.md) uses these and only these.

| Term | Definition | Aliases to avoid | Named in code at |
| --- | --- | --- | --- |
| **Transfer** | One sending of one file from a Sender to a Receiver. Everything a Receiver accepts belongs to exactly one Transfer. | session, job, upload | `src/lib/assembler.js:46` (`this.transferId`) |
| **Transfer ID** | Short uppercase base36 id that names a Transfer; carried in every Frame. | session id, tx id | `src/lib/protocol.js:60` (`generateTransferId`) |
| **Sender** | The device that shows Frames on its screen. Needs no camera. | transmitter, server, host | `src/pages/Sender.jsx` |
| **Receiver** | The device that reads Frames with its camera and rebuilds the file. Needs no screen output to the Sender. | scanner, client | `src/pages/Receiver.jsx` |
| **File hash** | SHA-256 of the original file, computed by the Sender and carried in the Metadata frame. | checksum, sender hash, signature | `src/lib/protocol.js:155` (`computeSHA256`), `src/hooks/useHashedFile.js:26` |
| **Verified** | The rebuilt file's SHA-256 equals the File hash. Only a Verified file can be downloaded. | valid, matched, ok | `src/lib/assembler.js:158` |
| **Chunk** | A fixed-size slice of the file's bytes, numbered from 0. The last one is zero-padded to full size. | block, piece, segment | `src/lib/fountain.js:65` (`splitChunks`) |
| **Chunk size** | Bytes per Chunk (500 / 800 / 1200 / 1600). Chosen by the Sender. | packet size, payload size | `src/options.js:2` (`CHUNK_SIZE_OPTIONS`) |
| **Packet** | The bytes that travel in one Data frame: the XOR of one or more Chunks. Always one Chunk size long. | chunk (see ambiguity 4), payload, block | `src/lib/fountain.js:77` (`encodePacket`) |
| **Seed** | Number that names a Packet. Sender and Receiver both turn a Seed into the same list of Chunks. | chunk index, sequence number | `src/lib/fountain.js:46` (`pickChunks`) |
| **Degree** | How many Chunks are XORed into a Packet. | weight, size | `src/lib/fountain.js:34` (`degreeFor`) |
| **First pass** | Seeds `0 .. totalChunks-1`: each Packet is exactly one Chunk, in order. A loss-free Transfer ends here. | systematic pass, first cycle, round | `src/lib/fountain.js:47` |
| **Repair packet** | Any Packet whose Seed is `totalChunks` or higher. It mixes random Chunks and fills in whatever the Receiver missed. | parity packet, retransmission | `src/lib/fountain.js:49-61` |
| **Pass** | One run of `totalChunks` Seeds. Pass 1 is the First pass; every later Pass is a repair pass. | cycle, loop, round | `src/lib/protocol.js:144` (`framesPerPass`), `src/pages/Sender.jsx:44` |
| **Frame** | The text inside one QR code. Either a Data frame or a Metadata frame. | QR, code, screen (see ambiguity 1) | `src/lib/protocol.js:145` (`frameAt`), `src/lib/protocol.js:74` (`parseFrame`) |
| **Data frame** | `OT3:<transferId>:<seed>:<totalChunks>:<Base45 Packet>`. Pure QR-alphanumeric text. | chunk frame, data QR | `src/lib/protocol.js:64` (`encodeDataFrame`) |
| **Metadata frame** | JSON Frame with file name, type, size, File hash and chunk count. Shown once every 10 Data frames. | meta, header, info frame | `src/lib/protocol.js:129` (`metaFrame`), `src/lib/protocol.js:15` (`META_EVERY`) |
| **Screen** | One image on the Sender's display. It holds 1, 2 or 4 Frames side by side (Codes per screen). | frame (see ambiguity 1) | `src/lib/qrGrid.js:10` (`drawQrGrid`), `src/pages/Sender.jsx:24` (`n`) |
| **Codes per screen** | How many Frames one Screen shows (1 / 2 / 4). | codes per frame, `perFrame` (see ambiguity 1) | `src/options.js:15` (`PER_FRAME_OPTIONS`) |
| **Solved chunk** | A Chunk whose bytes the Receiver knows for certain. | received chunk, decoded chunk | `src/lib/assembler.js:20` (`this.chunks`) |
| **Pending packet** | A received Packet that still mixes two or more unsolved Chunks. Kept until peeling reduces it to one. | buffered packet, queue | `src/lib/assembler.js:21` (`this.pending`) |
| **Missing chunk** | A Chunk index not yet solved. | lost chunk, gap | `src/lib/assembler.js:133` (`missing`) |
| **Duplicate** | A Data frame whose Seed the Receiver has already seen. Counted, never applied. | repeat, re-scan (see ambiguity 2) | `src/lib/assembler.js:65` |
| **Ignored code** | A decoded QR code that is not a Frame of the current Transfer: a foreign QR, a corrupt Frame, or a Frame from another Transfer. | foreign scan, invalid, `scans` (see ambiguity 6) | `src/lib/assembler.js:26` (`this.ignored`) |
| **Protocol version** | Which wire format a Frame uses. The current one is `OFFLINE_TRANSFER_V3`, shortened to `OT3` on Data frames. | build, revision | `src/lib/protocol.js:14` (`PROTOCOL_ID`), `src/lib/protocol.js:17` (`FRAME_PREFIX`) |
| **Reconstruct** | Join every Solved chunk in order, trim the padding to the file size, then check the File hash. | assemble, rebuild, merge | `src/lib/assembler.js:148` (`reconstruct`) |

## Flagged ambiguities

1. **"Frame" means two things.** In the protocol, a Frame is the text of *one QR code* (`frameAt(n)`, `src/lib/protocol.js:145`). In the Sender UI, a "frame" is *one Screen* holding 1, 2 or 4 QR codes (`n` in `src/pages/Sender.jsx:24`, the "Frame x / y" label at `src/pages/Sender.jsx:101`, "QR Codes per Frame" at `src/pages/Sender.jsx:74`). The clash reaches the code: `framesPerPass` counts Frames in `src/lib/protocol.js:144` but counts Screens in `src/pages/Sender.jsx:43` and `src/pages/Simulator.jsx:42`.
   **Winner: Frame = one QR code.** The display image is a **Screen**, and the setting is **Codes per screen**. The UI labels and the page-level `framesPerPass` still use the old meaning and should be renamed.

2. **"Duplicate" means two things.** `duplicates` goes up for a repeated Seed (`src/lib/assembler.js:65-67`) *and* for a new Seed whose Packet adds nothing because all of its Chunks are already solved (`src/lib/assembler.js:75-77`). The Receiver shows both as "Duplicates skipped" (`src/pages/Receiver.jsx:232`).
   **Winner: Duplicate = repeated Seed only.** A Packet that adds nothing is a *redundant packet*. The code still counts both together.

3. **"Pass" and "Cycle" name one thing.** The earlier vanilla-JS version (in `offline-file-transfer(1).zip`, `js/sender.js:228-236`) looped the Chunks in "cycles". The current code says Pass (`src/pages/Sender.jsx:89`).
   **Winner: Pass.** "Cycle" has been removed from README.md.

4. **"Chunk" and "Packet" get mixed up.** During the First pass a Packet *is* one Chunk, so the code reads Chunk size from a Packet's length (`this.chunkSize = frame.bytes.length`, `src/lib/assembler.js:71`). For Repair packets the two are different.
   **Winner: Chunk = a slice of the file; Packet = what travels in a Data frame.** Progress is counted in Solved chunks, never in Packets (`src/pages/Receiver.jsx:220`).

5. **"Seed" vs "chunk index".** In the First pass, Seed `i` carries Chunk `i` (`src/lib/fountain.js:47`), so it is tempting to call a Seed an index. Above the First pass that stops being true.
   **Winner: Seed.** "Chunk index" is only for positions in the file (`missing()`, `src/lib/assembler.js:133`).

6. **Two names for Ignored codes.** The assembler counts `ignored` (`src/lib/assembler.js:26`). The Receiver page keeps its own counter called `scans` (`src/pages/Receiver.jsx:17`) for the "N QR codes read, but none belong to a transfer" hint.
   **Winner: Ignored code.** `scans` is a misleading name, because it does not count every scan.

7. **One Protocol version, two spellings.** Metadata frames say `OFFLINE_TRANSFER_V3` (`src/lib/protocol.js:14`). Data frames say `OT3` (`src/lib/protocol.js:17`). `describeForeign` reports either one as `version` (`src/lib/protocol.js:104`).
   **Winner: Protocol version.** `OT3` is its short form, used only because Data frames must stay QR-alphanumeric.