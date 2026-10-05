# Testing

Run with `npm test` (Node's built-in `node:test`, no browser). All tests are in `test/protocol.test.js`. Rules are numbered as in [DESIGN.md](DESIGN.md).

| Rule | Test file:line | If untested — what stops you |
| --- | --- | --- |
| 1. A file can be downloaded only if it is Verified | `test/protocol.test.js:98` (correct bytes → `verified` true), `test/protocol.test.js:140` (wrong File hash → `verified` false) | **Half tested.** The `verified` flag is tested. The gate that withholds the download link (`src/pages/Receiver.jsx:71`, `:262`) is not: it lives in a React page, and the repo has no DOM test setup (no jsdom or Testing Library). |
| 2. A Receiver follows one Transfer at a time | `test/protocol.test.js:131` (a Data frame from `OTHERID` returns `null`) | — |
| 3. Not finished without every Chunk and the Metadata frame | `test/protocol.test.js:135-138` (all Chunks solved but `isComplete` false; true only after the Metadata frame) | — |
| 4. A Duplicate never changes the file | `test/protocol.test.js:130` (same Seed twice → `'duplicate'`), `test/protocol.test.js:139` (counted once), `test/protocol.test.js:95` (a loss-free Transfer has 0 Duplicates) | — |
| 5. An empty file cannot be sent | none | **Untested.** The check (`src/hooks/useHashedFile.js:15`) sits inside a React hook and calls `alert()`, so it needs a DOM environment we don't have. Fix: move the size check into a plain function in `src/lib/` and test it in Node. |

**Rules tested:** 4 of 5, and rule 1 only at the core level (the `verified` flag, not the download link).

**Weak test:** `test/protocol.test.js:140` claims to show that a hash mismatch is reported. It does that by passing a fake File hash (`'deadbeef'`, line 126) with perfect data, so it only checks the `===` on `src/lib/assembler.js:158`. It never flips a byte in a Packet, so it does not prove that a corrupted Transfer is caught.
