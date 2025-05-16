# BitTorrent Client Project Plan with Magnet Links and Distributed GUI

## Project Overview
**Goal**: Build a Go-based BitTorrent client with magnet link support, capable of downloading and seeding torrents, with a distributed architecture allowing a GUI to connect locally or remotely (via SSH) to a client running on a VPS or local machine.  
**Scope**: Core BitTorrent protocol features (magnet links, .torrent files, peer-to-peer transfers), CLI core, and a separate GUI (local native or SSH-accessible). No advanced tracker protocols like DHT.  
**Tech Stack**: 
- Go (standard library for networking, concurrency).
- [bencode-go](https://github.com/jackpal/bencode-go) for Bencode parsing.
- [Cobra](https://github.com/spf13/cobra) for CLI.
- [Fyne](https://github.com/fyne-io/fyne) for local native GUI (cross-platform, lightweight).
- gRPC or REST for client-GUI communication.
- SSH for remote access to VPS-hosted client.
**Target Duration**: 4-5 months (part-time, 10-15 hours/week).  
**Deliverable**: A CLI BitTorrent client and a separate GUI app, hosted on GitHub with documentation, tests, and deployment instructions.

---

## Module Breakdown

### Module 1: Project Setup, Bencode Parsing, and Magnet Link Support
**Objective**: Set up the Go project, implement Bencode parsing for .torrent files, and support magnet link parsing.  
**Duration**: 3 weeks  
**Tasks**:
1. **Initialize Project** (2 hours):
   - Create a Go module (`go mod init bittorrent-client`).
   - Set up Git repository and folder structure (`cmd/`, `pkg/`, `internal/`, `gui/`).
   - Configure linters ([golangci-lint](https://golangci-lint.run/)) and `gofmt`.
2. **Research Bencode and Magnet Links** (5 hours):
   - Study Bencode specification ([BEP 0003](http://bittorrent.org/beps/bep_0003.html)).
   - Study magnet link format ([BEP 0009](http://bittorrent.org/beps/bep_0009.html)), focusing on `xt` (info hash), `tr` (tracker URLs), and `dn` (display name).
3. **Implement Bencode Parser** (10 hours):
   - Use [github.com/jackpal/bencode-go](https://github.com/jackpal/bencode-go) for decoding .torrent files.
   - Create a `bencode` package to parse into structs (e.g., `TorrentFile` with `Announce`, `InfoHash`, `PieceLength`).
   - Write unit tests using `testing` package.
4. **Parse Magnet Links** (8 hours):
   - Implement a `magnet` package to parse magnet URIs (e.g., `magnet:?xt=urn:btih:<info-hash>&tr=<tracker>`).
   - Extract `info_hash`, tracker URLs, and optional fields (e.g., `dn`).
   - Fetch .torrent metadata from peers (basic metadata exchange per [BEP 0009](http://bittorrent.org/beps/bep_0009.html)).
5. **Parse .torrent Files** (5 hours):
   - Implement .torrent file parsing to extract metadata (tracker URL, file names, piece hashes).
   - Validate with sample .torrent files and magnet links.

**Deliverables**:
- Go module with structured layout.
- Bencode parser, .torrent file reader, and magnet link parser.
- Unit tests for parsing logic.
- Successful parsing of sample .torrent files and magnet links.

**Dependencies**: `github.com/jackpal/bencode-go`.  
**Risks**: Magnet link metadata fetching complexity. Mitigate with fallback to .torrent files and robust error handling.

---

### Module 2: Tracker Communication and Metadata Fetching
**Objective**: Communicate with trackers to discover peers and fetch metadata for magnet links.  
**Duration**: 3 weeks  
**Tasks**:
1. **Study Tracker Protocols** (5 hours):
   - Review [BEP 0015: UDP Tracker Protocol](http://bittorrent.org/beps/bep_0015.html) and HTTP tracker specs.
   - Study metadata extension for magnet links ([BEP 0009](http://bittorrent.org/beps/bep_0009.html)).
2. **Implement HTTP Tracker Client** (12 hours):
   - Use `net/http` to send announce requests (include `info_hash`, `peer_id`, `port`).
   - Parse Bencode-encoded tracker responses for peer lists.
   - Handle errors (e.g., invalid info_hash).
3. **Support Metadata Fetching** (8 hours):
   - Extend tracker client to request metadata for magnet links (if supported by tracker).
   - Store fetched metadata as a .torrent file equivalent.
4. **Optional UDP Tracker Support** (8 hours, stretch goal):
   - Implement UDP tracker protocol using `net` package.
   - Prioritize HTTP for simplicity.
5. **Write Tests** (5 hours):
   - Mock tracker responses using `httptest`.
   - Test peer list and metadata parsing.

**Deliverables**:
- `tracker` package for HTTP (optionally UDP) communication.
- Peer list and metadata fetching from trackers for .torrent files and magnet links.
- Unit tests for tracker logic.

**Dependencies**: Standard library (`net/http`, `net`).  
**Risks**: Tracker unreliability, metadata unavailability. Mitigate by supporting multiple trackers and peer-based metadata fetching.

---

### Module 3: Peer-to-Peer Communication and Metadata Exchange
**Objective**: Implement the BitTorrent peer protocol, including metadata exchange for magnet links.  
**Duration**: 4 weeks  
**Tasks**:
1. **Study Peer Protocol** (8 hours):
   - Review [BEP 0003: Peer Protocol](http://bittorrent.org/beps/bep_0003.html) and [BEP 0009: Metadata Extension](http://bittorrent.org/beps/bep_0009.html).
   - Understand handshake, messages (`choke`, `unchoke`, `piece`, `request`), and metadata messages.
2. **Implement Peer Handshake** (10 hours):
   - Create a `peer` package for TCP connections using `net`.
   - Implement handshake (send/receive `info_hash`, `peer_id`, extension bits for metadata).
3. **Handle Peer Messages** (15 hours):
   - Parse peer protocol messages (fixed and variable-length).
   - Implement core messages: `bitfield`, `have`, `request`, `piece`, `choke`, `unchoke`.
   - Support metadata extension messages (`ut_metadata`).
   - Use goroutines for concurrent peer connections.
4. **Fetch Metadata for Magnet Links** (8 hours):
   - Request metadata from peers supporting `ut_metadata`.
   - Assemble and validate metadata into a `TorrentFile` struct.
5. **Manage Peer State** (8 hours):
   - Track peer state (choked, interested, metadata support) in a `Peer` struct.
   - Prioritize peers with metadata or pieces.
6. **Write Tests** (8 hours):
   - Simulate peer interactions with a mock server.
   - Test handshake, message exchange, and metadata fetching.

**Deliverables**:
- `peer` package with peer protocol and metadata exchange.
- Concurrent peer handling and metadata fetching for magnet links.
- Unit tests for peer logic.

**Dependencies**: Standard library (`net`, `sync`).  
**Risks**: Peer protocol complexity, metadata availability. Mitigate with clear state management and fallback to tracker metadata.

---

### Module 4: File Download and Piece Management
**Objective**: Download files by requesting pieces from peers and verifying integrity.  
**Duration**: 3 weeks  
**Tasks**:
1. **Design Piece Management** (5 hours):
   - Create a `piece` package to track piece availability, download status, and verification.
   - Use a bitfield for piece availability.
2. **Implement Piece Download** (12 hours):
   - Request pieces using `request` messages.
   - Write pieces to temporary files using buffered I/O.
   - Use goroutines for concurrent piece downloads.
3. **Verify Pieces** (5 hours):
   - Compute SHA-1 hashes using `crypto/sha1`.
   - Compare against piece hashes from .torrent or metadata.
4. **Assemble File** (5 hours):
   - Combine verified pieces into final file(s).
   - Support multi-file torrents (optional).
5. **Write Tests** (5 hours):
   - Test piece download, verification, and file assembly.
   - Use small test torrents/magnet links.

**Deliverables**:
- `piece` package for downloading and verifying files.
- Complete file download from peers for .torrent or magnet links.
- Unit tests for piece logic.

**Dependencies**: Standard library (`crypto/sha1`, `io`, `os`).  
**Risks**: Disk I/O bottlenecks, corrupted pieces. Mitigate with buffered writes and strict verification.

---

### Module 5: Seeding and Core CLI
**Objective**: Enable seeding and provide a CLI for the core client.  
**Duration**: 2 weeks  
**Tasks**:
1. **Implement Seeding** (8 hours):
   - Respond to peer `request` messages with `piece` messages.
   - Track upload stats.
   - Reuse piece management for file reads.
2. **Build CLI Interface** (8 hours):
   - Use [Cobra](https://github.com/spf13/cobra) for commands (e.g., `bittorrent-client download <torrent-file|magnet-link>`).
   - Display progress (download/upload speed, peers, percentage).
   - Log errors and status.
3. **Write Tests** (4 hours):
   - Test seeding with mock peer requests.
   - Test CLI commands with mock inputs.

**Deliverables**:
- Seeding functionality in `peer` and `piece` packages.
- CLI for starting downloads/seeding (torrents or magnet links).
- Unit tests for seeding and CLI.

**Dependencies**: `github.com/spf13/cobra`.  
**Risks**: CLI usability, seeding efficiency. Mitigate with clear feedback and optimized reads.

---

### Module 6: Distributed Architecture and API
**Objective**: Implement a gRPC or REST API to allow GUI connections to the client (local or VPS).  
**Duration**: 3 weeks  
**Tasks**:
1. **Design API** (5 hours):
   - Define endpoints (e.g., start/stop download, get progress, list torrents).
   - Choose gRPC for performance or REST for simplicity (gRPC recommended for Go).
2. **Implement API Server** (12 hours):
   - Create a `server` package with gRPC (using [google.golang.org/grpc](https://pkg.go.dev/google.golang.org/grpc)) or REST (`net/http`).
   - Expose methods to control client (e.g., `AddTorrent`, `GetStatus`).
   - Use goroutines to handle API requests concurrently.
3. **Secure API** (5 hours):
   - Add basic authentication (e.g., API key or token).
   - For VPS, configure SSH tunneling for secure access (client connects via `ssh -L`).
4. **Write Tests** (5 hours):
   - Test API endpoints with mock clients.
   - Test SSH tunneling locally.

**Deliverables**:
- gRPC/REST API in `server` package.
- Secure API access with SSH tunneling support.
- Unit tests for API.

**Dependencies**: `google.golang.org/grpc` or `net/http`.  
**Risks**: API complexity, SSH setup issues. Mitigate with clear docs and simple authentication.

---

### Module 7: GUI Implementation
**Objective**: Build a cross-platform GUI using Fyne, connectable to the client locally or via SSH.  
**Duration**: 3 weeks  
**Tasks**:
1. **Research Fyne** (3 hours):
   - Study [Fyne](https://fyne.io/) for cross-platform GUI development in Go.
   - Explore basic widgets (buttons, progress bars, text inputs).
2. **Design GUI** (5 hours):
   - Create a simple layout: torrent list, add torrent/magnet link, start/stop buttons, progress bars, peer stats.
   - Support local and remote (VPS) connections.
3. **Implement GUI Client** (15 hours):
   - Create a `gui` package using Fyne.
   - Connect to client’s gRPC/REST API (local or via SSH tunnel).
   - Implement real-time updates (e.g., progress bars using API polling or gRPC streams).
4. **Write Tests** (5 hours):
   - Test GUI interactions with mock API responses.
   - Test local and SSH connections.

**Deliverables**:
- Fyne-based GUI in `gui` package.
- Local and remote (SSH) connectivity to client.
- Unit tests for GUI logic.

**Dependencies**: `fyne.io/fyne/v2`.  
**Risks**: GUI performance, SSH usability. Mitigate with lightweight design and clear SSH setup guide.

---

### Module 8: Testing, Documentation.Concurrent, and Deployment
**Objective**: Finalize with end-to-end tests, documentation, and public deployment.  
**Duration**: 3 weeks  
**Tasks**:
1. **End-to-End Testing** (10 hours):
   - Test full workflow (download/seed with .torrent and magnet links).
   - Test GUI with local and VPS-hosted client (use Docker for VPS simulation).
2. **Write Documentation** (8 hours):
   - Create `README.md` with setup, usage, VPS deployment, and SSH instructions.
   - Document code with comments and package docs.
   - Include GUI screenshots and architecture diagram.
3. **Polish Code** (5 hours):
   - Refactor for Go idioms (small interfaces, clear errors).
   - Run linters and fix issues.
4. **Deploy to GitHub** (3 hours):
   - Push to public GitHub repository.
   - Add GitHub Actions for CI (linting, testing).
   - Release CLI and GUI binaries with `go build`.

**Deliverables**:
- Fully tested client and GUI.
- Comprehensive documentation with VPS/SSH guide.
- Public GitHub repository with CI pipeline.

**Dependencies**: GitHub Actions, Docker (testing).  
**Risks**: Test flakiness, incomplete docs. Mitigate with stable tests and user-focused docs.

---

## Timeline and Effort
**Total Duration**: 21 weeks (~5 months)  
**Weekly Commitment**: 10-15 hours (2-3 hours weekdays, 5-7 hours weekends)  
**Milestone Schedule**:
- **Week 1-3**: Module 1 (Setup, Bencode, magnet links)
- **Week 4-6**: Module 2 (Tracker communication, metadata)
- **Week 7-10**: Module 3 (Peer communication, metadata exchange)
- **Week 11-13**: Module 4 (File download, piece management)
- **Week 14-15**: Module 5 (Seeding, CLI)
- **Week 16-18**: Module 6 (Distributed API)
- **Week 19-21**: Module 7 (GUI implementation)
- **Week 22-24**: Module 8 (Testing, documentation, deployment)

---

## Technical Considerations
- **Magnet Links**: Require metadata fetching from peers or trackers, adding complexity to peer protocol. Prioritize peers supporting `ut_metadata`.
- **Distributed Architecture**: gRPC is preferred for low-latency GUI-client communication. SSH tunneling ensures secure VPS access (e.g., `ssh -L 8080:localhost:8080 user@vps`).
- **GUI**: Fyne is lightweight and cross-platform, ideal for a simple torrent client. Avoid heavy frameworks to keep the GUI responsive.
- **Concurrency**: Use goroutines for peer connections, piece downloads, and API handling. Use channels for coordination and `sync.Mutex` for shared state.
- **Error Handling**: Propagate errors explicitly per Go conventions. Log clearly for CLI and GUI users.
- **Testing**: Aim for 70-80% coverage. Use table-driven tests for parsing/protocol logic and Docker for end-to-end tests.
- **Dependencies**: Keep minimal (standard library, `bencode-go`, `cobra`, `fyne`, `grpc`). Vet dependencies for stability.

---

## Resources
- **Learning**: [Go by Example](https://gobyexample.com/), [Effective Go](https://golang.org/doc/effective_go.html), [Fyne Docs](https://developer.fyne.io/).
- **Specs**: [BitTorrent BEP Index](http://bittorrent.org/beps/bep_0000.html), especially BEP 0003, 0009, 0015.
- **Inspiration**: [anacrolix/torrent](https://github.com/anacrolix/torrent) for Go BitTorrent code (study, don’t copy).
- **Tools**: [Delve](https://github.com/go-delve/delve) for debugging, [golangci-lint](https://golangci-lint.run/) for linting.

---

## Success Criteria
- Download and seed files using .torrent files and magnet links.
- CLI is intuitive with clear progress output.
- GUI connects locally or to a VPS (via SSH) and displays torrent status.
- Code is well-documented, tested, and follows Go idioms.
- Project is public on GitHub with professional README, including VPS setup guide.