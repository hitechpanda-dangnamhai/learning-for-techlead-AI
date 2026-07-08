# Chain gối đầu — Tạo Collections & Embeddings trong ChromaDB

> Nén theo chain gối đầu: đuôi node trước = đầu node sau. Thuật ngữ giữ nguyên tiếng Anh,
> giải thích tiếng Việt ngắn. Bảng so sánh / đính chính giữ dạng bảng.
> Bài gốc dùng JS API lỗi thời + không nhắc distance metric (mặc định L2!) + thiếu bước query
> → phần đính chính giữ nguyên như mắt xích trong chain.

---

## MẠCH CHÍNH — collection & embedding là gì
có 10.000 embedding vứt rời rạc thì không quản được (không biết cái nào thuộc nhóm nào, không gắn nhãn, không tìm nhanh) => nên cần một container có tên gom các vector cùng nhóm => trong ChromaDB container đó gọi là collection => collection chứa một tập embeddings + documents + metadata, có tên riêng => document là dữ liệu gốc (text/ảnh/video) đi kèm embedding của nó => embedding là dãy số *cố định độ dài* (fixed-length) biểu diễn ý nghĩa của một data point trong không gian nhiều chiều => "cố định độ dài" quan trọng vì mọi vector phải cùng số chiều mới so sánh/đo khoảng cách được => mỗi số trong vector là một dimension (đặc trưng) => metadata là nhãn/tag key–value đính kèm mỗi item => metadata cho phép filter lúc query (vd chỉ lấy topic="auth") => còn hàm biến document → embedding gọi là embedding function (EF) => điểm đo độ giống giữa hai vector gọi là similarity score (cao = giống, với cosine)

## CHAIN — Ví dụ "dog" trên bản đồ ý nghĩa (rẽ từ "embedding")
embedding đặt mỗi từ như một điểm trên bản đồ ý nghĩa => vd vector của "dog" là [0.123, 0.456, …, 0.789] (50–300 chiều) => tìm các từ gần "dog" nhất trong embedding space => ra dogs (0.85), rồi puppy / pet / canine / puppies (đều cao) => vì từ có vector *gần* vector "dog" thì nghĩa liên quan => nên dog / puppy / canine nằm sát nhau, còn dog / airplane nằm xa

## CHAIN — Auto-embed vs sinh thủ công (rẽ từ "EF")
collection cần biết dùng model nào để biến document thành embedding => đó chính là embedding function (EF) => cách cũ (dated) của bài gốc: generate embedding thủ công rồi add cả embedding lẫn document => cách này rườm rà, dễ sai (query sau này quên embed cùng model → rác) => cách hiện đại: gắn EF vào collection một lần => rồi `add(documents=...)` → Chroma tự embed document => và `query(query_texts=...)` → Chroma tự embed *cả câu query* bằng đúng EF đó => nhờ vậy query và document luôn cùng không gian => đây gọi là auto-embed, an toàn hơn generate thủ công

## CHAIN — EF là CONTRACT bất biến (rẽ từ "EF")
EF không phải một tham số nhỏ, nó là contract (hợp đồng) cố định của collection => nên mọi vector trong collection phải được sinh bởi cùng EF (cùng model, cùng số chiều) => hệ quả 1: số chiều bị khoá bởi EF đầu tiên => collection tạo với EF 384 chiều thì không thể add vector 1536 chiều → lỗi => hệ quả 2: đổi EF = vô hiệu toàn bộ collection => vì vector từ model A và model B khác không gian, so sánh vô nghĩa => nên muốn đổi model phải tạo collection mới + re-embed toàn bộ + rebuild index (đắt, khó đảo ngược) => hệ quả 3: query phải cùng EF với lúc add => đó chính là lý do pattern auto-embed an toàn

## CHAIN — Bẫy L2 mặc định (rẽ từ "distance metric")
ChromaDB mặc định dùng L2 (Euclidean squared) cho collection mới => KHÔNG phải cosine (đây là cái bẫy bài gốc bỏ sót) => nhưng đa số text embedding được thiết kế cho cosine => nên để mặc định L2 mà không normalize thì xếp hạng có thể lệch => phải chọn metric rõ ràng lúc tạo: `configuration={"hnsw":{"space":"cosine"}}` (l2 | cosine | ip inner product) => với space="cosine", Chroma trả về cosine *distance* = 1 − cosine similarity => nên giá trị gần 0 = giống nhau (không phải gần 1) => nhầm chiều so sánh là lỗi kinh điển => lưu ý (quyển 4): vector đã normalize thì L2, cosine, ip cho cùng thứ hạng → metric ít quan trọng bằng việc có normalize hay không

## CHAIN — Hierarchy tenant → database → collection (rẽ)
bài gốc coi collection là tầng cao nhất => nhưng ChromaDB thực ra có 3 tầng: tenant → database → collection => 3 tầng này quan trọng cho multi-tenancy (nhiều khách hàng/nhóm dùng chung một cụm nhưng cách ly dữ liệu) => nên dùng tenant/database để cách ly & phân quyền => thay vì nhồi mọi thứ vào một collection

## CHAIN — Collection ≠ SQL table (rẽ)
bài gốc nói collection "giống một relational table" => đúng ở mức trực giác (có tên, chứa nhiều "hàng") nhưng có giới hạn => collection là schema-less (không enforce kiểu cột) => không có JOIN nhiều bảng => không có ACID transaction => nghiệp vụ chính của collection là similarity search, không phải exact/range query => nên đừng mang tư duy SQL (JOIN, transaction) áp vào collection — nó là kho vector có gắn nhãn, không phải RDBMS

## CHAIN — Persistence: chọn client đúng (rẽ)
client mặc định `chromadb.Client()` là in-memory => nên dữ liệu mất khi tắt chương trình (bài gốc không cảnh báo) => vì vậy phải chọn client đúng theo nhu cầu lưu trữ => Ephemeral (RAM, test) → Persistent (đĩa, SQLite+HNSW, app nhỏ) → Http (client-server, nhiều app) → Cloud (serverless, distributed) — chi tiết ở bảng

## CHAIN — Batch, ID design, idempotency (rẽ)
batch embedding: embed cả list một lần nhanh hơn từng cái => vì tận dụng vectorization/GPU, và với API EF còn giảm số request → giảm cost => về ID: `ids` phải unique => trùng id thì hành vi phụ thuộc add vs upsert => `upsert` là idempotent (cập nhật item cũ theo id) => còn `add` trùng id có thể lỗi/ghi đè tùy version => nên thiết kế id ổn định (vd `docid:chunk_index`) để re-embed không tạo bản trùng

## CHAIN — Big-O của add & query (rẽ)
`add` = bước embed (O(số token) mỗi doc, phần đắt) + chèn HNSW (~O(log N) mỗi vector) => `query` = embed query + HNSW search ~O(log N) => nên bottleneck của `add` thường là bước embed, không phải chèn index => nhất là với API EF (latency mạng)

## CHAIN — Edge cases (rẽ độc lập)
model tải ngầm lần đầu: DefaultEmbeddingFunction tải all-MiniLM lần đầu → chậm/cần mạng, môi trường air-gapped sẽ *fail* → pre-download model, dùng API EF, hoặc custom EF offline => document rỗng → vector vô nghĩa; document quá dài → vượt context của model → bị cắt (chunking quyển 3) → validate độ dài trước khi add => add embedding thủ công khác số chiều EF → lỗi → để Chroma tự embed => metadata cardinality cao (mỗi item một giá trị unique) → filter kém hiệu quả → thiết kế metadata để filter có ý nghĩa

## CHAIN — 3 lỗi thường gặp khi code (rẽ độc lập)
lỗi 1: dùng property/pattern JS cũ (`embeddings:`, `.generate()` thủ công, `.then()`) → phải dùng `embeddingFunction:`, auto-embed từ `documents`, async/await, và cài `@chroma-core/default-embed` => lỗi 2: tên collection không hợp lệ (quá ngắn/ký tự lạ) → phải 3–512 ký tự `[a-zA-Z0-9._-]`, bắt đầu & kết thúc bằng chữ/số (vd `"kb"` sẽ lỗi) => lỗi 3: query bằng EF khác lúc add → luôn để collection tự embed cả add lẫn query bằng cùng một EF

## CHAIN — Một hay nhiều collection ở scale (rẽ, staff)
câu hỏi thiết kế: gom mọi thứ vào một collection khổng lồ hay chia nhiều collection => một collection lớn + metadata filter thì đơn giản, nhưng filter trên collection tỷ-item chậm nếu cardinality cao, và mọi tenant chung một index (rủi ro cách ly) => nhiều collection (theo tenant/loại dữ liệu) thì cách ly tốt, filter tự nhiên (mỗi tenant một namespace), nhưng nhiều collection nhỏ → overhead quản lý + khó query xuyên collection => nguyên tắc staff: chia theo ranh giới cách ly & phân quyền (tenant, độ nhạy dữ liệu), không chia bừa

## CHAIN — Chi phí nằm ở sinh embedding (rẽ, staff)
bottleneck chi phí thật nằm ở *sinh embedding*, không phải lưu/tìm => với API EF (OpenAI/Cohere): trả tiền mỗi document embed + latency mạng mỗi add/query → embed 100M chunk = hoá đơn lớn → batch, cache, cân nhắc self-host => với Local/Custom EF: cần CPU/GPU, embed 100M doc bằng GPU tốn thời gian & tiền → thường đắt hơn cả chi phí lưu/tìm vector => số chiều là đòn bẩy cost kép: chiều lớn → embed chậm hơn VÀ storage/latency search cao hơn → cân nhắc model chiều vừa hoặc Matryoshka (quyển 4)

## CHAIN — Đổi model / re-embed (rẽ, staff)
EF là contract nên đổi embedding model → re-embed toàn bộ + tạo collection mới + swap => chiến lược blue-green: dựng collection mới song song, embed nền, cắt traffic khi sẵn sàng => version hoá tên collection (vd `docs_v2_bge`) => giữ id ổn định để đối chiếu => và lên kế hoạch cost/thời gian re-embed TRƯỚC khi commit chọn model

## CHAIN — Reliability & monitoring (rẽ, staff)
idempotency: dùng `upsert` + id ổn định để pipeline re-run không tạo bản trùng => freshness: add → searchable có độ trễ (index nền, log-structured quyển 3) → giám sát độ trễ này nếu cần near-real-time => giám sát thêm: phân bố distance của query (drift?), tỉ lệ query rỗng/dưới ngưỡng, latency embed (API EF hay nghẽn), dung lượng collection, số chiều => failure mode: model tải ngầm fail (air-gapped), API EF rate-limit/đổi giá, số chiều lệch khi add thủ công, collection phình quá RAM

## CHAIN — System design SaaS RAG 1000 tenant (rẽ, khung staff)
đề: lớp lưu trữ vector cho SaaS RAG đa khách hàng (1000 tenant, mỗi tenant vài trăm nghìn tài liệu), cách ly dữ liệu tuyệt đối => bước 1 ràng buộc trước: cách ly tuyệt đối → KHÔNG dùng một collection chung với filter (rủi ro rò rỉ), dùng tenant/database riêng hoặc collection-per-tenant => bước 2 cấu trúc: mỗi tenant → database/collection riêng, id dạng `docid:chunk`, metadata gồm source/acl/updated_at => bước 3 EF & metric: chọn một EF chuẩn (BGE-M3 self-host cho dữ liệu nhạy cảm), space="cosine", normalize, khoá số chiều => bước 4 ingestion: chunk → auto-embed (batch) → upsert idempotent, blue-green khi đổi model => bước 5 query: auto-embed query bằng đúng EF, giới hạn trong collection của tenant (cách ly by design, không post-filter) => bước 6 scale/cost: bottleneck = embed → batch + self-host GPU, giám sát drift & latency, kế hoạch re-embed => bước 7 kết bằng trade-off (collection-per-tenant cách ly tốt nhưng overhead quản lý vs shared+filter đơn giản nhưng rủi ro rò rỉ)

---

## BẢNG — 3 lựa chọn Embedding Function (EF)
| EF | Model / cách chạy | Ưu / nhược |
|---|---|---|
| **DefaultEmbeddingFunction** | Sentence Transformers all-MiniLM-L6-v2 (384 chiều), chạy local, tải model tự động lần đầu | miễn phí, tốt để bắt đầu; cần tải model lần đầu (fail nếu air-gapped) |
| **OpenAIEmbeddingFunction / providers** | gọi API (OpenAI text-embedding-3-small, Cohere, Google…) | chất lượng cao; trả phí mỗi lần embed, gửi dữ liệu ra ngoài |
| **Custom EF** | tự viết (implement interface `EmbeddingFunction`) | khi cần model riêng / offline; tự lo hạ tầng |

> JS 2026: `DefaultEmbeddingFunction` nằm ở package riêng `@chroma-core/default-embed` (giảm bundle size).

## BẢNG — Collection vs Relational table (analogy có giới hạn)
| | Relational table | Chroma collection |
|---|---|---|
| Schema cứng (kiểu cột) | ✅ có, enforce chặt | ❌ không (metadata linh hoạt, schema-less) |
| JOIN nhiều bảng | ✅ có | ❌ không |
| ACID transaction | ✅ có | ❌ không |
| Nghiệp vụ chính | exact/range query trên cột | **similarity search** trên vector |

## BẢNG — 4 kiểu persistence client
| Client | Lưu trữ | Khi nào dùng |
|---|---|---|
| `Client()` / `EphemeralClient()` | in-memory (mất khi thoát) | chỉ để test/prototype |
| `PersistentClient(path=...)` | đĩa (SQLite + HNSW) | app nhỏ, 1 máy |
| `HttpClient(...)` (`chroma run`) | server | nhiều app dùng chung |
| `CloudClient(...)` | Chroma Cloud | serverless, distributed |

## BẢNG — Đính chính bài gốc (code JS lỗi thời / thiếu)
| Bài gốc | Đúng ra là (2026) |
|---|---|
| property `embeddings` | 🚨 phải là `embeddingFunction` |
| `default_ef.generate(...)` thủ công rồi add | bản mới **auto-embed** từ `documents`/`query_texts` |
| `DefaultEmbeddingFunction` bundle trong `chromadb` | nay ở package riêng `@chroma-core/default-embed` |
| `.then()` | async/await |
| (không nhắc) distance metric | mặc định **L2**, không phải cosine → set `configuration={"hnsw":{"space":"cosine"}}` |
| chỉ create + add | thiếu bước **query** — nửa quan trọng nhất |
| "collection = relational table" | thiếu giới hạn: không schema/JOIN/ACID |
| dùng client mặc định | không cảnh báo in-memory → **mất dữ liệu khi thoát** |
| (bỏ qua) | tenant/database hierarchy, EF-as-contract (số chiều khoá, re-embed khi đổi model), chi phí embed, model tải ngầm, upsert/idempotency |

---

## TỪ KHÓA MỒI
- Mạch chính: **collection = embeddings + documents + metadata**
- Ví dụ dog: **dog/puppy/canine gần, airplane xa**
- Auto-embed vs thủ công: **cùng EF cho add & query**
- EF là contract: **số chiều bị khoá**
- Bẫy L2 mặc định: **L2 không phải cosine**
- Hierarchy: **tenant → database → collection**
- Collection ≠ SQL table: **không JOIN/ACID**
- Persistence: **in-memory mất khi thoát**
- Batch/ID/idempotency: **upsert + docid:chunk**
- Big-O: **bottleneck của add là embed**
- Edge cases: **model tải ngầm air-gapped fail**
- 3 lỗi code: **embeddingFunction không phải embeddings**
- Một hay nhiều collection: **chia theo ranh giới cách ly**
- Chi phí: **tiền ở bộ não embed**
- Đổi model: **blue-green re-embed**
- Reliability & monitoring: **idempotency + freshness**
- System design: **1000 tenant cách ly by design**

## ĐÃ PHỦ
- **Vấn đề & nền tảng:** vector rời rạc không quản được → collection (container có tên) ✓
- **Thuật ngữ:** collection (embeddings+documents+metadata), document, embedding (fixed-length, vì sao cùng độ dài mới so được), dimension, metadata (filter), embedding function, similarity score ✓
- **Ví dụ dog:** vector [0.123…], dogs 0.85, puppy/pet/canine/puppies cao, gần=liên quan, dog/airplane xa ✓
- **Auto-embed vs thủ công:** cách cũ generate+add rườm rà/dễ rác, cách mới gắn EF 1 lần, add/query tự embed cùng không gian ✓
- **3 lựa chọn EF** (bảng): DefaultEmbeddingFunction (all-MiniLM 384 chiều local miễn phí), OpenAI/providers (API, phí, gửi ra ngoài), Custom EF ✓
- **EF là contract:** mọi vector cùng EF, số chiều khoá bởi EF đầu (384 vs 1536 lỗi), đổi EF = vô hiệu → collection mới + re-embed + rebuild, query phải cùng EF ✓
- **Bẫy L2 mặc định:** L2 (Euclidean squared) không phải cosine, text muốn cosine, set configuration hnsw space (l2|cosine|ip), cosine distance = 1−similarity gần 0=giống, normalize → 3 metric cùng thứ hạng ✓
- **Hierarchy:** tenant → database → collection, multi-tenancy, cách ly & phân quyền ✓
- **Collection ≠ SQL table** (bảng): schema-less, không JOIN, không ACID, nghiệp vụ similarity search ✓
- **Persistence** (bảng): Ephemeral (RAM, mất khi thoát), Persistent (SQLite+HNSW đĩa), Http (client-server), Cloud (serverless) ✓
- **Batch/ID/idempotency:** batch embed nhanh hơn (vectorization/GPU, giảm request), ids unique, upsert idempotent vs add trùng lỗi, id ổn định docid:chunk_index ✓
- **Big-O:** add = embed O(token) + chèn HNSW O(log N), query = embed + HNSW O(log N), bottleneck add là embed (API EF latency) ✓
- **Edge cases:** model tải ngầm (air-gapped fail → pre-download/API/offline), doc rỗng/quá dài (validate), số chiều lệch khi add thủ công, metadata cardinality cao → filter kém ✓
- **3 lỗi code:** JS cũ (embeddings:/.generate()/.then()), tên collection không hợp lệ (3–512 ký tự, "kb" lỗi), query khác EF ✓
- **Một hay nhiều collection:** một lớn + filter (đơn giản, chậm/rủi ro cách ly) vs nhiều (cách ly tốt, overhead/khó xuyên collection), chia theo ranh giới cách ly ✓
- **Chi phí embed:** API EF (phí mỗi doc + latency, 100M = hoá đơn lớn), Local/Custom (GPU, đắt hơn lưu/tìm), số chiều đòn bẩy cost kép, Matryoshka ✓
- **Đổi model/re-embed:** re-embed toàn bộ + collection mới + swap, blue-green, version hoá tên, giữ id, lên kế hoạch trước ✓
- **Reliability & monitoring:** upsert+id ổn định, freshness add→searchable trễ, giám sát distance drift/query rỗng/latency embed/dung lượng/số chiều, failure modes (air-gapped, rate-limit, số chiều lệch, phình RAM) ✓
- **System design 1000 tenant:** ràng buộc cách ly → tenant/collection riêng, cấu trúc id/metadata, EF+cosine+normalize, ingestion auto-embed+upsert+blue-green, query trong collection tenant, cost, trade-off ✓
- **Bảng đính chính bài gốc** (9 điểm) ✓
- *Bỏ có chủ đích (không phải kiến thức):* analogy "ngăn tủ"/"bản đồ ý nghĩa", self-check, one-liner tiếng Anh (chỉ diễn đạt lại ý đã phủ).
