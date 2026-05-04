# Lộ trình học Blockchain qua NEO source code

> Dành cho .NET web developer — học từ "user của blockchain" → "người xây và phát triển blockchain".
> Tick `[x]` mỗi mục khi hoàn thành. Mỗi phase có **mục tiêu**, **file quan trọng**, **bài tập**, và một **câu hỏi checkpoint** để tự kiểm tra hiểu bài.

---

## Phase 0 — Nền tảng & môi trường (1–2 buổi)

**Mục tiêu**: có sandbox để chạy/debug, có mental model về blockchain liên hệ với web dev.

**Mental model cần có** (liên hệ với web/.NET):
- Block ≈ append-only log entry, immutable sau khi commit
- Transaction ≈ một "command" được ký bởi user (giống signed JWT request)
- Blockchain ≈ một state machine replicated trên nhiều node, mọi node đều phải compute ra cùng một state (deterministic)
- Consensus ≈ phiên bản distributed của "two-phase commit" — các node phải đồng ý về thứ tự transaction
- Smart contract ≈ stored procedure chạy trên VM, có storage riêng
- Native contract ≈ system stored procedure, được compile sẵn vào core
- Gas ≈ "billing" cho mỗi opcode VM thực thi (chống infinite loop & DoS)

**Checklist:**
- [ ] Build solution: `dotnet build /workspaces/neo/neo.sln`
- [ ] Chạy thử test: `dotnet test /workspaces/neo/tests/Neo.UnitTests`
- [ ] Đọc [README.md](../README.md) và [CONTRIBUTING.md](../CONTRIBUTING.md)
- [ ] Đọc trang chính của doc chính thức: <https://docs.neo.org/>
- [ ] Cài **neo-express** (private blockchain dev tool) hoặc dùng **neo-cli** để có node chạy local — quan trọng để có cảm giác "blockchain đang chạy" trước khi đụng code
- [ ] Tạo wallet, gửi 1 transaction trên private net, xem block được tạo

**Checkpoint**: Bạn có thể giải thích bằng tiếng nói thường (không thuật ngữ) cho một người không biết blockchain rằng "khi tôi gửi 1 GAS, điều gì thực sự xảy ra giữa các node?"

---

## Phase 1 — Cấu trúc dữ liệu cốt lõi (3–5 buổi)

**Mục tiêu**: hiểu Block, Transaction, Hash, Serialization. Đây là phần *dễ tiếp cận nhất* với dev — chỉ là data structures + (de)serialization.

**File quan trọng:**
- [src/Neo/UInt160.cs](../src/Neo/UInt160.cs), [src/Neo/UInt256.cs](../src/Neo/UInt256.cs) — fixed-size hashes (160-bit cho address, 256-bit cho block/tx hash)
- [src/Neo/Network/P2P/Payloads/](../src/Neo/Network/P2P/Payloads/) — Block, Header, Transaction, Witness định nghĩa ở đây
- [src/Neo.IO/](../src/Neo.IO/) — interface `ISerializable` và helper
- [docs/serialization-format.md](../docs/serialization-format.md) — đọc kỹ
- [src/Neo/Cryptography/MerkleTree.cs](../src/Neo/Cryptography/MerkleTree.cs) — MerkleRoot trong header được tính từ đây

**Checklist:**
- [ ] Mở `Block.cs`, list ra các field của 1 block và **giải thích vì sao mỗi field tồn tại** (PrevHash → để link, MerkleRoot → để verify integrity tx, NextConsensus → ai sẽ sign block tiếp theo, …)
- [ ] Mở `Transaction.cs`, list field và phân biệt: Signers, Attributes, Script, Witnesses
- [ ] Đọc `ISerializable.Serialize/Deserialize` của Block và Transaction — hiểu format binary
- [ ] **Bài tập**: viết unit test tự serialize 1 Transaction giả lập, hash nó bằng `Transaction.Hash`, so sánh với hash decode lại
- [ ] Hiểu Merkle tree: tự vẽ tay 1 cây Merkle với 4 transactions, đối chiếu với `MerkleTree.ComputeRoot`

**Checkpoint**: Bạn có thể giải thích vì sao `Transaction.Hash` không bao gồm `Witnesses` (gợi ý: malleability — tránh attacker thay đổi signature mà không thay hash)

---

## Phase 2 — Cryptography (2–3 buổi)

**Mục tiêu**: hiểu chữ ký số, address derivation, và vai trò của hash trong blockchain. Không cần hiểu math sâu — chỉ cần biết **cái gì đảm bảo cái gì**.

**File quan trọng:**
- [src/Neo/Cryptography/Crypto.cs](../src/Neo/Cryptography/Crypto.cs) — entrypoint cho ECDSA verify
- [src/Neo/Cryptography/ECC/](../src/Neo/Cryptography/ECC/) — elliptic curve math (đọc lướt, không cần hiểu sâu)
- [src/Neo/Cryptography/Ed25519.cs](../src/Neo/Cryptography/Ed25519.cs) — chữ ký Ed25519
- [src/Neo/Cryptography/Helper.cs](../src/Neo/Cryptography/Helper.cs) — Hash160/Hash256, RIPEMD160, SHA256
- [src/Neo/Wallets/](../src/Neo/Wallets/) — KeyPair, address generation

**Checklist:**
- [ ] Hiểu công thức address: `Address = Base58Check( 0x35 || Hash160( verification_script ) )` (hoặc tương tự — kiểm tra trong `Contract.CreateSignatureContract` + `Wallets/Helper.cs`)
- [ ] Phân biệt 3 loại hash trong codebase: SHA256, Hash160 (= RIPEMD160(SHA256(x))), Hash256 (= SHA256(SHA256(x)))
- [ ] Hiểu **tại sao** mỗi cái dùng ở đâu (Hash256 cho tx/block hash, Hash160 cho address)
- [ ] **Bài tập**: cho 1 private key bất kỳ, tính ra public key → script_hash → address bằng tay (gọi các helper trong codebase)
- [ ] Đọc `Crypto.VerifySignature` — hiểu input/output, không cần hiểu math bên trong

**Checkpoint**: Nếu bạn có private key của tôi và 1 transaction tôi đã ký, bạn có thể tạo 1 transaction *mới* khác mà chữ ký vẫn valid không? Tại sao?

---

## Phase 3 — Persistence & State (3–4 buổi)

**Mục tiêu**: hiểu blockchain "lưu state ở đâu và như thế nào". Phần này có doc tốt nhất trong repo.

**File quan trọng:**
- [docs/persistence-architecture.md](../docs/persistence-architecture.md) — **đọc đầu tiên**
- [src/Neo/Persistence/IStore.cs](../src/Neo/Persistence/IStore.cs), [IStoreSnapshot.cs](../src/Neo/Persistence/IStoreSnapshot.cs) — abstraction trên storage backend (LevelDB/RocksDB/Memory)
- [src/Neo/Persistence/DataCache.cs](../src/Neo/Persistence/DataCache.cs) — write-through cache layer — **trái tim của state management**
- [src/Neo/Persistence/Providers/](../src/Neo/Persistence/Providers/)
- [src/Neo/SmartContract/StorageKey.cs](../src/Neo/SmartContract/StorageKey.cs), [StorageItem.cs](../src/Neo/SmartContract/StorageItem.cs)

**Mental model**: state = giant key-value store. Mỗi smart contract có "namespace" riêng (prefix bằng contract id). Block apply = một batch các thay đổi key-value, commit atomic.

**Checklist:**
- [ ] Đọc xong `persistence-architecture.md`
- [ ] Hiểu vì sao có 2 layer: `IStore` (backend thật) và `DataCache` (in-memory, có thể rollback)
- [ ] Hiểu `Snapshot` model: 1 block đang execute thấy snapshot tại t=0 + thay đổi của riêng nó, các block khác không thấy
- [ ] **Bài tập**: trace 1 lần `LedgerContract` lưu một block — từ `Blockchain.OnNewBlock` → DataCache → Commit → IStore
- [ ] So sánh với khái niệm bạn đã biết: DataCache giống `DbContext` của EF Core (track changes, commit), IStore giống database thật

**Checkpoint**: Nếu 2 transaction trong cùng 1 block đều update cùng 1 storage key, transaction nào "thắng"? Vì sao thiết kế thế?

---

## Phase 4 — Networking / P2P (3–4 buổi)

**Mục tiêu**: hiểu các node nói chuyện với nhau ra sao, block và transaction lan truyền thế nào.

**File quan trọng:**
- [src/Neo/Network/P2P/LocalNode.cs](../src/Neo/Network/P2P/LocalNode.cs) — node của chính mình
- [src/Neo/Network/P2P/RemoteNode.cs](../src/Neo/Network/P2P/RemoteNode.cs), [RemoteNode.ProtocolHandler.cs](../src/Neo/Network/P2P/RemoteNode.ProtocolHandler.cs) — đại diện cho 1 peer
- [src/Neo/Network/P2P/Message.cs](../src/Neo/Network/P2P/Message.cs), [MessageCommand.cs](../src/Neo/Network/P2P/MessageCommand.cs) — wire protocol
- [src/Neo/Network/P2P/TaskManager.cs](../src/Neo/Network/P2P/TaskManager.cs) — quản lý "tôi đang xin block từ ai"

**Mental model**: P2P giống pub/sub mesh. Mỗi node giữ 1 list peer. Khi nhận tx mới → broadcast cho neighbors. Khi behind block → `getheaders/getdata` để sync.

NEO dùng **Akka.NET** (actor model) — nếu chưa biết, hãy đọc qua Akka actor cơ bản. Mọi component (LocalNode, Blockchain, ConsensusService, …) đều là Actor → giao tiếp qua message asynchronous.

**Checklist:**
- [ ] Đọc qua các `MessageCommand` enum — list các loại message và mục đích
- [ ] Trace flow: 1 transaction được user submit → vào `LocalNode` → broadcast → `RemoteNode` của peer khác nhận → đi đâu?
- [ ] Trace flow: node mới khởi động, làm sao biết block hiện tại đến đâu để sync? (`getheaders` → `getdata` → `block`)
- [ ] Đọc qua Akka.NET cơ bản (15 phút): Actor, Tell, Ask, mailbox
- [ ] **Bài tập**: chạy 2 instance neo-cli local, dùng wireshark/tcpdump xem chúng gửi gì cho nhau

**Checkpoint**: Tại sao P2P broadcast lại dùng `inv` (inventory — chỉ gửi hash) trước, rồi peer mới `getdata` để xin nội dung — thay vì broadcast luôn cả tx?

---

## Phase 5 — Smart Contract & VM (5–7 buổi — phần lớn nhất)

**Mục tiêu**: hiểu cách 1 transaction script được thực thi.

**File quan trọng:**
- [src/Neo/SmartContract/ApplicationEngine.cs](../src/Neo/SmartContract/ApplicationEngine.cs) — **entry point thực thi script**
- [src/Neo/SmartContract/ApplicationEngine.*.cs](../src/Neo/SmartContract/) — các partial class cho từng nhóm syscall (Storage, Runtime, Crypto, Contract, Iterator)
- [src/Neo/SmartContract/InteropDescriptor.cs](../src/Neo/SmartContract/InteropDescriptor.cs) — đăng ký system call
- [src/Neo/SmartContract/NefFile.cs](../src/Neo/SmartContract/NefFile.cs) — format file contract sau khi compile
- [src/Neo/SmartContract/Manifest/](../src/Neo/SmartContract/Manifest/) — metadata contract (ABI)
- Repo riêng: `neo-vm` (subrepo của neo-project) — opcode definitions, ExecutionEngine

**Mental model**:
- Neo VM = stack-based VM (giống PostScript / Forth, không phải register-based như x86)
- 1 transaction có 1 `Script` (bytecode) → VM execute → có thể đọc/ghi storage, gọi contract khác, emit notification
- `ApplicationEngine` extend ExecutionEngine để thêm: gas metering, syscall, storage access, blockchain context

**Checklist:**
- [ ] Hiểu stack machine cơ bản: PUSH, POP, ADD, JMP, CALL — tự simulate trên giấy
- [ ] Đọc `ApplicationEngine.cs` constructor: `Trigger`, `ScriptContainer`, `Snapshot`, `gas` — hiểu mỗi field cho gì
- [ ] Đọc 1 syscall đơn giản: `System.Runtime.Log` trong `ApplicationEngine.Runtime.cs`
- [ ] Đọc 1 syscall phức tạp: `System.Storage.Put` trong `ApplicationEngine.Storage.cs`
- [ ] **Bài tập**: viết test gọi `ApplicationEngine` với 1 script đơn giản (ví dụ: PUSH 1, PUSH 2, ADD, RET) — verify result stack
- [ ] Hiểu gas metering: ai trả gas, khi nào fail, gas refund không? (Đọc `ApplicationEngine.AddFee` và `OpCodePrices`)
- [ ] Hiểu `TriggerType`: Application vs Verification — khác nhau ra sao và vì sao cần 2 mode

**Checkpoint**: Khi 1 contract A gọi contract B, stack frame của A có còn không? Storage context khi B chạy là của ai? Gas tính cho ai?

---

## Phase 6 — Native contracts (2–3 buổi)

**Mục tiêu**: hiểu các "system contract" được hardcode trong core, đặc biệt là NEO/GAS token.

**File quan trọng:**
- [src/Neo/SmartContract/Native/NativeContract.cs](../src/Neo/SmartContract/Native/NativeContract.cs) — base class
- [src/Neo/SmartContract/Native/NeoToken.cs](../src/Neo/SmartContract/Native/NeoToken.cs) — governance token, có voting & committee
- [src/Neo/SmartContract/Native/GasToken.cs](../src/Neo/SmartContract/Native/GasToken.cs) — utility token
- [src/Neo/SmartContract/Native/PolicyContract.cs](../src/Neo/SmartContract/Native/PolicyContract.cs) — fee policy
- [src/Neo/SmartContract/Native/RoleManagement.cs](../src/Neo/SmartContract/Native/RoleManagement.cs) — committee/oracle/state validators
- [src/Neo/SmartContract/Native/ContractManagement.cs](../src/Neo/SmartContract/Native/ContractManagement.cs) — deploy/update/destroy contract
- [src/Neo/SmartContract/Native/LedgerContract.cs](../src/Neo/SmartContract/Native/LedgerContract.cs) — đọc block/tx từ contract
- [src/Neo/SmartContract/Native/CryptoLib.cs](../src/Neo/SmartContract/Native/CryptoLib.cs)
- [src/Neo/SmartContract/Native/StdLib.cs](../src/Neo/SmartContract/Native/StdLib.cs)

**Checklist:**
- [ ] Đọc `NativeContract.cs` — hiểu mechanism `[ContractMethod]` + reflection để expose method ra VM
- [ ] Đọc `NeoToken.cs`: hiểu `Vote`, `Committee`, `NextValidators` — đây là *governance layer* của NEO
- [ ] Hiểu mỗi epoch (mỗi 21 block ở mainnet?) committee được tính lại như thế nào
- [ ] Đọc `GasToken.cs`: hiểu `OnPersist` (mỗi block, distribute gas cho NEO holders) và burn fee
- [ ] **Bài tập**: trace luồng "user transfer NEO" — entry script → ContractManagement → resolve → NeoToken.transfer → state change

**Checkpoint**: Vì sao NEO/GAS dùng dạng "native contract" thay vì "smart contract thường được deploy"? Trade-off?

---

## Phase 7 — Consensus dBFT (3–5 buổi)

**Mục tiêu**: hiểu cách các validator đồng ý về thứ tự block.

**Lưu ý**: dBFT logic không nằm trong repo này — nó là plugin riêng (`DBFTPlugin` trong neo-modules, hoặc subrepo `neo-project/neo-modules`). Repo `neo` core chỉ định nghĩa interface và payload (xem `NeoSystem.cs` chỗ inject consensus service).

**Pre-reading bắt buộc:**
- [ ] Whitepaper dBFT 2.0: <https://docs.neo.org/docs/n3/Advanced/dBFT.html>
- [ ] Khái niệm chung: PBFT (Practical Byzantine Fault Tolerance) — Castro & Liskov 1999

**Checklist:**
- [ ] Hiểu cơ bản: với `n` validator, dBFT chịu được tối đa `f = (n-1)/3` node Byzantine (malicious/down)
- [ ] Hiểu 3 message: `PrepareRequest` (primary đề xuất block), `PrepareResponse` (backup vote ok), `Commit` (commit khi đủ ≥ 2f+1 vote)
- [ ] Hiểu **view change**: khi primary fail, làm sao rotate sang primary mới
- [ ] Clone neo-modules, đọc `DBFTPlugin/Consensus/ConsensusService.cs`
- [ ] **Bài tập**: chạy private net 4 validators bằng neo-express, kill 1 node, xem network có còn produce block không. Kill thêm 1 → còn produce không? (Trả lời: 4 node tolerate được f=1, nên kill 1 vẫn ok, kill 2 sẽ stop)

**Checkpoint**: So sánh dBFT vs PoW (Bitcoin) vs PoS (Ethereum 2.0) — mỗi cái đánh đổi gì?

---

## Phase 8 — Build / Contribute (mở)

**Mục tiêu**: từ "đọc hiểu" → "viết được". Đây là lúc kiến thức trở thành thực sự của bạn.

**Lựa chọn task tăng dần độ khó:**
- [ ] **Easy**: viết một plugin đơn giản (logger, RPC method mới) — đọc [src/Neo/Plugins/Plugin.cs](../src/Neo/Plugins/Plugin.cs) và xem các plugin có sẵn ở `neo-modules`
- [ ] **Easy**: tìm `good-first-issue` trên <https://github.com/neo-project/neo/issues>
- [ ] **Medium**: viết unit test cho 1 file chưa có coverage tốt
- [ ] **Medium**: trace 1 bug đã được fix trong git log gần đây, **tự reproduce** trước khi xem patch
- [ ] **Hard**: thêm 1 method mới vào 1 native contract (cần hardfork — đọc `Hardfork.cs` để hiểu cơ chế)
- [ ] **Hard**: implement 1 RPC method mới
- [ ] **Stretch**: thử implement 1 BIP/NIP đơn giản (ví dụ: 1 NEP — Neo Enhancement Proposal)

---

## Tài liệu tham khảo chung

- Doc chính thức: <https://docs.neo.org/>
- DeepWiki repo này: <https://deepwiki.com/neo-project/neo>
- Whitepaper Neo N3: <https://docs.neo.org/docs/n3/Basic/whitepaper.html>
- Sách kinh điển (concept blockchain, không phải NEO-specific): *Mastering Bitcoin* (Andreas Antonopoulos), *Mastering Ethereum* (cũng A.A.)
- Akka.NET basics: <https://getakka.net/articles/intro/what-is-akka.html>

---

## Nguyên tắc xuyên suốt

1. **Đọc theo luồng dữ liệu, không đọc theo file**. Chọn 1 use case (vd: "tx được xác minh và vào block") rồi *trace* xuyên suốt — sẽ hiểu kiến trúc nhanh hơn đọc tuần tự.
2. **Gắn breakpoint, debug step-through** — quan trọng hơn đọc code chay. NEO chạy được trong Visual Studio / VS Code với debugger đầy đủ.
3. **Viết test cho mỗi khái niệm bạn nghĩ là đã hiểu** — đó là cách tốt nhất để verify hiểu thật hay hiểu giả.
4. **Đặt câu hỏi "tại sao", không chỉ "cái gì"**. Code trả lời "cái gì". Whitepaper, issues, PR discussion trả lời "tại sao".
5. **Đừng cố hiểu hết một lượt**. Pass đầu lướt nhanh để có map tổng thể, pass 2 mới đào sâu chỗ cần.
