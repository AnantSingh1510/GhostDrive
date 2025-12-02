# Architecture

GhostDrive is built as a layered system that transforms a standard video file into a block-addressable, encrypted, error-corrected virtual filesystem. Each layer has a clear responsibility, and data flows top-to-bottom from user operations down to raw video frames.

The architecture is organized into **eight major layers**, from user space all the way down to physical video storage.

---

## **L1 — User Space**

This layer includes everything that users directly interact with:

- Standard POSIX commands (`cp`, `ls`, `cat`, `mkdir`, `rm`)
- The GhostDrive CLI (`ghostdrive create`, `ghostdrive mount`, `ghostdrive verify`)
- The mount point (e.g., `/mnt/ghost`)

From the user's perspective, GhostDrive behaves exactly like a normal filesystem.

---

## **L2 — FUSE Interface Layer**

GhostDrive implements a complete FUSE3 filesystem, enabling the OS to treat the mounted video file as a real disk.

Implemented filesystem operations include:

- `open`
- `read`
- `write`
- `mkdir`
- `readdir`
- `unlink`

These operations are routed to the GhostDrive Core Engine for processing.

---

## **L3 — GhostDrive Core Engine**

This is the heart of GhostDrive. It orchestrates metadata, block mapping, caching, and all read/write logic.

### Components:

#### **GhostDrive Manager**  
Coordinates all filesystem operations, block translations, and metadata updates.

#### **Frame Cache (LRU)**  
A lightweight cache (~100 frames) that speeds up reads and reduces redundant frame decodes.

#### **Metadata Manager**  
Handles filesystem metadata including directory structure, file entries, block allocation, timestamps, etc.

#### **Metadata Structures** (stored in early frames):

- **Frame 0** — Superblock (magic, version, configs)
- **Frames 1–5** — Block bitmap (free/used block tracking)
- **Frames 6–50** — Inode table (files + directories)

---

## **L4 — Block Device Layer**

This layer abstracts the concept of “blocks” and maps them 1:1 to video frames.

- Block size: **64 KB**
- Block N maps to **Frame (51 + N)**

Operations include:

- `read_block`
- `write_block`
- `allocate`
- `free`

---

## **L5 — Data Transformation Pipeline**

This layer converts data blocks into video-safe frames and back.

### **Write Pipeline**
Raw 64KB Block
- Encrypt (ChaCha20-Poly1305)
- Add ECC (Reed–Solomon)
- Encode bytes → RGB pixels
- Produce video-ready frame


### **Read Pipeline**
Frame
- Decode pixels
- ECC recovery
- Decrypt
- Raw 64KB Block


This pipeline ensures integrity, confidentiality, and resilience against video compression or corruption.

---

## **L6 — Security & Reliability Layer**

GhostDrive uses modern cryptography and error correction:

- **ChaCha20-Poly1305** — authenticated encryption of each block  
- **Reed–Solomon ECC** — protects against corrupted or partially damaged frames  
- **Argon2id** — secure password-to-key derivation  

These features ensure that even if the video is recompressed or altered, data can still be recovered.

---

## **L7 — Video Encoding Layer**

GhostDrive uses **FFmpeg** (libavcodec + libavformat) to convert frames into a playable video.

Operations:

- `extract_frame(n)`
- `inject_frame(n)`
- `encode` (MJPEG / GOP=1 intra-frame)
- `decode`

Using I-frame-only encoding ensures that each frame is independently readable — essential for random block access.

---

## **L8 — Physical Storage**

This is the actual output file, e.g.:
ghostdrive.mp4


Video layout:
Frame 0 → Superblock
Frames 1–5 → Block Bitmap
Frames 6–50 → Inode Table
Frame 51+N → Data Blocks


The file plays normally in media players, but internally it represents a fully functional filesystem.

---

## Summary

GhostDrive’s architecture combines:

- FUSE filesystem semantics  
- a block device abstraction  
- encryption + ECC  
- pixel encoding  
- video encoding  
- and a structured metadata layout  

…to create a filesystem hidden inside a standard video.

This layered design keeps components modular, testable, and easy to extend.


