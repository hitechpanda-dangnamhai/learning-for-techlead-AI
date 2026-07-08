# Giáo trình: Tạo Collections & Embeddings trong ChromaDB

> Biên soạn theo phong cách "cầm tay chỉ việc" — từ zero đến tư duy Staff Engineer.
> Giảng bằng tiếng Việt, giữ nguyên technical terms tiếng Anh để phục vụ phỏng vấn big tech.
>
> **Đây là giáo trình thứ 5 trong loạt về vector database.** Bốn quyển trước: (1) *vector DB hoạt động thế nào*, (2) *ứng dụng*, (3) *ChromaDB & RAG*, (4) *embeddings & distance metrics*. Quyển này (5) là **thực hành thuần**: làm sao *thực sự tạo một collection và sinh embeddings* trong ChromaDB bằng code. Đây là bài "hands-on" nhất — nên trọng tâm là **code chạy được** và các gotcha khi gõ phím thật.
>
> **Cảnh báo ngay từ đầu:** đoạn code JavaScript trong bài giảng gốc dùng **API đã lỗi thời** (ChromaDB đã viết lại JS client — bản V3, 6/2025). Cụ thể: sai tên property (`embeddings` → phải là `embeddingFunction`), sinh embedding *thủ công* rồi mới add (bản mới **tự embed**), và `DefaultEmbeddingFunction` giờ nằm ở **package riêng**. Bài cũng **không nhắc distance metric** (mặc định là L2, không phải cosine — một cái bẫy), và **chỉ dạy create + add, không có bước query** (thiếu nửa quan trọng). Tôi giữ phần đúng, đính chính code, cập nhật API 2026, và chạy thật bản Python để bạn thấy kết quả. Đánh dấu 🛠️ **[Đính chính]** và ➕ **[Mở rộng ngoài bài gốc]**.

---

## Phần 0 — 🗺️ Bản đồ bài học (Overview)

Ở quyển 3–4 ta đã hiểu *ChromaDB là gì* và *embedding/distance metric hoạt động ra sao*. Quyển này trả lời câu hỏi thực dụng nhất: **"Mở editor lên, tôi gõ gì để có một collection chứa dữ liệu và tìm kiếm được?"** Bài giảng gốc dạy hai khái niệm (**collection** = cái "bảng" chứa embeddings/documents/metadata; **embedding** = dãy số cố định độ dài biểu diễn ý nghĩa) rồi đưa một đoạn code tạo collection và thêm dữ liệu. Ta sẽ đi qua toàn bộ, sửa code cho đúng 2026, **thêm bước query mà bài gốc bỏ quên**, và mổ những gotcha thật khi vận hành.

**Sau khi học xong, bạn sẽ có thể:**
- Giải thích **collection** là gì, giống/khác một relational table ở điểm nào (và giới hạn của analogy đó).
- Hiểu **embedding** như một "điểm trên bản đồ ý nghĩa"; đọc được ví dụ dog/puppy/canine.
- Viết code **chạy được** (Python + JS 2026) để: tạo collection → gắn embedding function → **tự động** sinh embedding khi add → **query** kèm metadata filter.
- Tránh 3 gotcha kinh điển: sai property API, quên chọn distance metric (mặc định L2!), và model bị tải ngầm lúc chạy.
- Thiết kế collection ở quy mô lớn: một collection lớn vs nhiều collection, multi-tenancy, chi phí embed.

**Mạch kiến thức từ basic → staff:**
- 🟢 **Basic:** collection & embedding là gì (analogy "ngăn tủ" + "điểm trên bản đồ") → code hello-world tạo collection.
- 🟡 **Intermediate:** embedding function (default/OpenAI/custom), **auto-embed** vs sinh thủ công, thêm bước **query**, code JS 2026 đúng chuẩn.
- 🔴 **Advanced:** embedding function là một *contract*; default metric L2 vs cosine; hierarchy tenant/database/collection; persistence; batch & ID design; Big-O.
- 🟣 **Staff:** thiết kế collection ở scale lớn, chi phí embed, re-embed khi đổi model, monitoring, system design.
- 🎯 **Cheatsheet:** Keywords, core concepts, code phải thuộc, câu hỏi phỏng vấn + one-liners.

---

## Phần 1 — 🟢 BASIC (Nền tảng)

### 1.1. "Tại sao" — cần một cái "hộp" để chứa vector

Bạn có 10.000 embedding. Vứt rời rạc thì không quản được: không biết cái nào thuộc dự án nào, không gắn được nhãn, không tìm nhanh. Bạn cần một **container** có tên, gom các vector *cùng nhóm* lại, kèm theo dữ liệu gốc và nhãn. Trong ChromaDB, container đó gọi là **collection**.

### 1.2. Analogy đời thường

- **Collection = một ngăn tủ hồ sơ có dán nhãn.** Mỗi ngăn (collection) gom các hồ sơ *cùng loại*. Mỗi hồ sơ trong ngăn gồm: **bản gốc** (document — text/ảnh/video), **mã vạch ý nghĩa** (embedding — dãy số), và **giấy dán ghi chú** (metadata — nhãn/tag). Ngăn có tên để bạn mở đúng ngăn.
- **Embedding = một điểm trên bản đồ ý nghĩa.** Bài gốc dùng đúng hình ảnh này: *"tất cả embedding như những điểm trên bản đồ; collection đo được khoảng cách/độ gần giữa các điểm."* Điểm gần nhau = nghĩa giống nhau.

### 1.3. Thuật ngữ nền tảng (giải thích ngay lần đầu)

- **Collection:** container trong ChromaDB chứa một tập **embeddings + documents + metadata**, có tên riêng. Giống (một cách lỏng lẻo) một table trong relational DB.
- **Document:** dữ liệu gốc (text/ảnh/video) đi kèm embedding của nó.
- **Embedding:** *dãy số **cố định độ dài*** biểu diễn ý nghĩa của một data point trong không gian nhiều chiều. Mỗi số = một **dimension** (đặc trưng).
- **Metadata:** nhãn/tag (key–value) đính kèm mỗi item → cho phép **filter** lúc query (vd chỉ lấy `topic="auth"`).
- **Embedding function (EF):** hàm biến document → embedding. ChromaDB có sẵn `DefaultEmbeddingFunction`, hoặc dùng OpenAI, hoặc tự viết.
- **Similarity score:** điểm đo độ giống giữa hai vector; cao = giống hơn (với cosine similarity).

### 1.4. Ví dụ chạy tay — "bản đồ ý nghĩa" của từ *dog*

Bài gốc minh hoạ: vector của **dog** = `[0.123, 0.456, ..., 0.789]` (50–300 chiều). Khi tìm các từ gần **dog** nhất trong embedding space, ta được:

| Từ gần "dog" | Similarity score |
|---|---|
| dogs   | 0.85 |
| puppy  | (cao) |
| pet    | (cao) |
| canine | (cao) |
| puppies| (cao) |

Ý nghĩa: mỗi số trong vector là một *dimension* (đặc trưng); những từ có vector *gần* vector "dog" → **nghĩa liên quan**. Đây chính là "điểm trên bản đồ": *dog, puppy, canine* nằm sát nhau; *dog* và *airplane* nằm xa. (Cách tính "gần" chính là cosine/L2 đã học ở quyển 4.)

### 1.5. Code "hello world" — tạo collection & add (Python, đã chạy thật)

Bài gốc dùng JavaScript; tôi bắt đầu bằng Python cho gọn (và **chạy được thật ngay**), rồi đưa JS 2026 ở Phần 2.

```python
# pip install chromadb
import chromadb

client = chromadb.Client()                       # in-memory, cho prototyping

# Tạo collection. LƯU Ý: tên phải 3-512 ký tự [a-zA-Z0-9._-].
col = client.create_collection(name="my_collection21")

# Add dữ liệu. Nếu collection có embedding function (mặc định có),
# Chroma sẽ TỰ sinh embedding từ `documents` -> không cần tính tay.
col.add(
    ids=["id1", "id2", "id3"],
    documents=["a friendly puppy", "a small kitten", "a fast sports car"],
    metadatas=[{"kind": "animal"}, {"kind": "animal"}, {"kind": "vehicle"}],
)
print("Số item trong collection:", col.count())   # -> 3
```
Giải thích: `create_collection` tạo "ngăn tủ"; `add` bỏ hồ sơ vào — bạn chỉ đưa **documents**, Chroma lo phần biến thành vector. (Mặc định EF sẽ **tải model all-MiniLM-L6-v2** lần đầu — xem gotcha ở Phần 3.)

### 1.6. ✅ Self-check (Basic)

1. Collection chứa những 3 thành phần gì? Metadata dùng để làm gì?
2. "Embedding là *fixed-length list of numbers*" — vì sao "fixed-length" (cùng độ dài) lại quan trọng? (Gợi ý: để so sánh/đo khoảng cách được, mọi vector phải cùng số chiều.)
3. Trong ví dụ dog, vì sao *puppy* và *canine* có similarity score cao với *dog*?

---

## Phần 2 — 🟡 INTERMEDIATE (Vận dụng)

### 2.1. Embedding function — trái tim của việc "biến chữ thành vector"

Collection cần biết *dùng model nào* để biến document thành embedding. Đó là **embedding function (EF)**. Ba lựa chọn:

- **DefaultEmbeddingFunction:** dùng **Sentence Transformers `all-MiniLM-L6-v2`** (384 chiều), chạy **local** (tải model tự động lần đầu). Tốt để bắt đầu, miễn phí.
- **OpenAIEmbeddingFunction / các provider:** gọi API (OpenAI `text-embedding-3-small`, Cohere, Google...). Chất lượng cao, **trả phí mỗi lần embed**, gửi dữ liệu ra ngoài.
- **Custom EF:** tự viết (implement interface `EmbeddingFunction`) khi cần model riêng.

🛠️ **[Đính chính — code JS của bài gốc đã lỗi thời]**
Bài gốc mô tả: `client.createCollection` với property **`embeddings`** = default_ef, rồi gọi **`default_ef.generate(...)`** để sinh embedding *thủ công*, rồi `collection.add` trong `.then()`. Ba điểm sai/cũ so với **ChromaDB JS V3 (2026)**:
1. Property đúng là **`embeddingFunction`**, không phải `embeddings`.
2. **Không cần** `generate` thủ công — gắn EF vào collection rồi chỉ cần đưa `documents`, Chroma **tự embed**.
3. `DefaultEmbeddingFunction` giờ **không còn bundle sẵn** trong `chromadb`; phải cài package riêng **`@chroma-core/default-embed`** (để giảm bundle size).

### 2.2. Auto-embed vs sinh thủ công — pattern hiện đại

**Cách bài gốc (dated):** `generate` embedding → rồi `add` cả embedding lẫn document. Rườm rà, dễ sai (query sau này quên embed cùng model → rác).

**Cách hiện đại (khuyến nghị):** gắn EF vào collection *một lần*, sau đó:
- `add(documents=...)` → Chroma tự embed document.
- `query(query_texts=...)` → Chroma tự embed *cả câu query* **bằng đúng EF đó** → đảm bảo query và document cùng không gian (nhắc lại nguyên tắc nhất quán ở quyển 4).

### 2.3. Code JS 2026 đúng chuẩn (thay cho code lỗi thời của bài gốc)

```javascript
// npm install chromadb @chroma-core/default-embed
import { ChromaClient } from "chromadb";
import { DefaultEmbeddingFunction } from "@chroma-core/default-embed"; // package RIÊNG

const client = new ChromaClient();                     // mặc định nối localhost:8000
const embedder = new DefaultEmbeddingFunction();       // all-MiniLM-L6-v2, 384 chiều

async function generateAndAdd() {
  // property đúng là `embeddingFunction` (KHÔNG phải `embeddings`)
  const collection = await client.createCollection({
    name: "my_collection21",
    embeddingFunction: embedder,
  });

  // MODERN: chỉ đưa documents -> Chroma TỰ embed (không gọi generate thủ công)
  await collection.add({
    ids: ["id1", "id2", "id3"],
    documents: ["a friendly puppy", "a small kitten", "a fast sports car"],
    metadatas: [{ kind: "animal" }, { kind: "animal" }, { kind: "vehicle" }],
  });

  // BƯỚC QUERY (bài gốc thiếu!): tự embed query, tìm 2 kết quả gần nhất
  const res = await collection.query({ queryTexts: ["cute doggo"], nResults: 2 });
  console.log(res.documents); // -> [["a friendly puppy", "a small kitten"]]
}
generateAndAdd();
```

### 2.4. Bản Python đầy đủ có query + custom EF (đã chạy thật, offline)

Để chạy được *ngay* mà không cần tải model, tôi dùng một custom EF đơn giản — đồng thời minh hoạ đúng interface EF:

```python
import chromadb
from chromadb import Documents, EmbeddingFunction, Embeddings

class ToyEmbeddingFunction(EmbeddingFunction):
    def __call__(self, input: Documents) -> Embeddings:
        out = []
        for doc in input:
            d = doc.lower()
            out.append([                                   # "embed" giả -> vector 3 chiều
                float(d.count("dog")+d.count("puppy")+d.count("canine")),  # độ "chó"
                float(d.count("cat")+d.count("kitten")),                    # độ "mèo"
                float(len(d))/20.0,                                         # độ dài
            ])
        return out
    @staticmethod
    def name() -> str: return "toy-ef"
    def get_config(self): return {}
    @staticmethod
    def build_from_config(config): return ToyEmbeddingFunction()

client = chromadb.Client()
col = client.create_collection(
    name="animal_docs",
    embedding_function=ToyEmbeddingFunction(),
    configuration={"hnsw": {"space": "cosine"}},   # CHỌN metric (xem Phần 3)
)
col.add(ids=["a","b","c"],
        documents=["a puppy and a dog", "a canine pet dog", "a cat and a kitten"],
        metadatas=[{"kind":"dog"},{"kind":"dog"},{"kind":"cat"}])   # auto-embed
print("count:", col.count())                                       # -> 3

res = col.query(query_texts=["cute doggo"], n_results=2)            # auto-embed query
print("query docs:", res["documents"][0])   # -> ['a puppy and a dog','a canine pet dog']

res2 = col.query(query_texts=["kitten"], n_results=1, where={"kind":"cat"})
print("filtered :", res2["documents"][0])   # -> ['a cat and a kitten']
```
**Kết quả thật:** `count: 3` → query "cute doggo" trả về đúng hai document về chó → filter `kind=cat` trả về document mèo. Đây là vòng đời **create → add(auto-embed) → query(+filter)** hoàn chỉnh (bài gốc dừng ở add).

### 2.5. 3 lỗi thường gặp (và cách tránh)

1. **Dùng property/pattern JS cũ** (`embeddings:`, `.generate()` thủ công, `.then()`). → Dùng `embeddingFunction:`, auto-embed từ `documents`, async/await; cài `@chroma-core/default-embed`.
2. **Tên collection không hợp lệ** (quá ngắn/ký tự lạ). → 3–512 ký tự `[a-zA-Z0-9._-]`, bắt đầu & kết thúc bằng chữ/số. (Ví dụ `"kb"` sẽ lỗi.)
3. **Query bằng EF khác lúc add.** → Luôn để collection tự embed cả add lẫn query bằng *cùng một EF*; đừng embed câu query bằng model khác.

### 2.6. ✅ Self-check (Intermediate)

1. Trong ChromaDB JS 2026, property nào truyền embedding function vào `createCollection`? `DefaultEmbeddingFunction` cài ở đâu?
2. Vì sao "auto-embed từ documents" an toàn hơn "generate thủ công rồi add"?
3. Viết một lệnh query lấy 3 kết quả gần "puppy" nhất nhưng chỉ trong metadata `kind="dog"`.

---

## Phần 3 — 🔴 ADVANCED (Chuyên sâu)

### 3.1. Embedding function là một *CONTRACT* bất biến

➕ **[Mở rộng — điểm staff quan trọng nhất của bài này]** EF không phải "một tham số nhỏ" — nó là **hợp đồng (contract) cố định của collection**. Mọi vector trong collection *phải* được sinh bởi cùng EF (cùng model, cùng số chiều). Hệ quả:
- **Số chiều phải khớp:** nếu collection tạo với EF 384 chiều, bạn *không thể* add vector 1536 chiều → lỗi. Số chiều bị "khoá" bởi EF đầu tiên.
- **Đổi EF = vô hiệu toàn bộ collection:** vector từ model A và model B khác không gian, so sánh vô nghĩa (nhắc lại quyển 4). Muốn đổi model → **tạo collection mới + re-embed toàn bộ + rebuild index** — thao tác đắt, khó đảo ngược.
- **Query phải cùng EF:** đó là lý do pattern auto-embed an toàn — Chroma đảm bảo query dùng đúng EF của collection.

### 3.2. 🛠️ Bẫy ngầm: distance metric mặc định là L2, KHÔNG phải cosine

Bài gốc **không hề nhắc distance metric**. Đây là thiếu sót nguy hiểm:

➕ **ChromaDB mặc định dùng L2 (Euclidean squared)** cho collection mới, **không phải cosine**. Nhưng đa số **text embedding được thiết kế cho cosine**. Nếu bạn để mặc định L2 mà không normalize, xếp hạng có thể lệch. → Chọn metric rõ ràng lúc tạo:
```python
col = client.create_collection(
    name="docs",
    configuration={"hnsw": {"space": "cosine"}},   # l2 | cosine | ip (inner product)
)
```
**Lưu ý (nối quyển 4):** với `space="cosine"`, Chroma trả về **cosine *distance* = 1 − cosine similarity** → **giá trị gần 0 = giống nhau** (không phải gần 1!). Nhầm chiều so sánh là lỗi kinh điển. Còn nhớ quyển 4: nếu vector đã normalize thì L2, cosine, ip cho *cùng thứ hạng* — nên nhiều khi metric ít quan trọng bằng việc *có normalize hay không*.

### 3.3. Hierarchy: Tenant → Database → Collection (multi-tenancy)

➕ Bài gốc coi collection là tầng cao nhất. Thực ra ChromaDB có **3 tầng**: **tenant → database → collection**. Điều này quan trọng cho **multi-tenancy** (nhiều khách hàng/nhóm dùng chung một cụm Chroma nhưng cách ly dữ liệu). Ở tầm ứng dụng thật (nhất là RAG doanh nghiệp ở quyển 3), bạn dùng tenant/database để **cách ly & phân quyền**, không nhồi mọi thứ vào một collection.

### 3.4. "Collection giống relational table" — analogy có giới hạn

🛠️ Bài gốc nói collection "functions like a relational database table." Đúng ở mức trực giác (có tên, chứa nhiều "hàng"), nhưng **một staff phải biết giới hạn**:
| | Relational table | Chroma collection |
|---|---|---|
| Schema cứng (kiểu cột) | ✅ Có, enforce chặt | ❌ Không (metadata linh hoạt, schema-less) |
| JOIN nhiều bảng | ✅ Có | ❌ Không |
| ACID transaction | ✅ Có | ❌ Không (nhắc quyển 1) |
| Nghiệp vụ chính | Exact/range query trên cột | **Similarity search** trên vector |
→ Đừng mang tư duy SQL (JOIN, transaction) áp vào collection. Nó là *kho vector có gắn nhãn*, không phải RDBMS.

### 3.5. Persistence: chọn client đúng

➕ Bài gốc dùng client mặc định (in-memory) mà không cảnh báo **dữ liệu mất khi tắt chương trình**. Bốn kiểu client:
- **`chromadb.Client()` / `EphemeralClient()`** — in-memory, mất khi thoát. Chỉ để test.
- **`PersistentClient(path=...)`** — lưu ra đĩa (SQLite + HNSW). App nhỏ 1 máy.
- **`HttpClient(...)`** (`chroma run`) — client-server, nhiều app dùng chung.
- **`CloudClient(...)`** — Chroma Cloud (serverless, distributed).

### 3.6. Batch, ID design, idempotency & Big-O

- **Batch embedding:** bài gốc nhắc đúng — embed *cả list* một lần nhanh hơn từng cái (tận dụng vectorization/GPU). Với API EF, batch còn giảm số request → giảm cost.
- **ID design:** `ids` phải **unique**. Trùng id → hành vi phụ thuộc `add` vs `upsert`. Dùng **`upsert`** nếu muốn cập nhật item cũ (idempotent); `add` trùng id có thể lỗi/ghi đè tùy version. Thiết kế id ổn định (vd `docid:chunk_index`) để re-embed không tạo bản trùng.
- **Big-O:** `add` = embed (O(số token) mỗi doc, phần đắt) + chèn HNSW (~O(log N) mỗi vector). `query` = embed query + HNSW search ~O(log N) (nhắc quyển 1). **Bottleneck của `add` thường là bước *embed*, không phải chèn index** — nhất là với API EF (latency mạng).

### 3.7. Edge cases phải xử lý

- **Model tải ngầm lần đầu:** `DefaultEmbeddingFunction` tải all-MiniLM lần đầu → chậm/cần mạng; môi trường air-gapped sẽ *fail* (tôi từng gặp lỗi tải model khi không có mạng). → Pre-download model, hoặc dùng API EF, hoặc custom EF offline.
- **Document rỗng / quá dài:** doc rỗng → vector vô nghĩa; doc vượt context của model → bị cắt (nhắc chunking ở quyển 3). → Validate độ dài trước khi add.
- **Số chiều lệch:** add embedding thủ công khác số chiều EF → lỗi. → Để Chroma tự embed.
- **Metadata cardinality cao** (mỗi item một giá trị unique) → filter kém hiệu quả. → Thiết kế metadata để filter có ý nghĩa.

---

## Phần 4 — 🟣 STAFF LEVEL (Tư duy hệ thống & lãnh đạo kỹ thuật)

### 4.1. Thiết kế collection ở quy mô lớn — một hay nhiều collection?

➕ Câu hỏi thiết kế thật mà bài gốc không chạm: **gom mọi thứ vào một collection khổng lồ, hay chia nhiều collection?**
- **Một collection lớn + metadata filter:** đơn giản, nhưng filter trên collection tỷ-item có thể chậm nếu cardinality cao; và mọi tenant chung một index (rủi ro cách ly).
- **Nhiều collection (theo tenant/loại dữ liệu):** cách ly tốt, filter tự nhiên (mỗi tenant một namespace — nhắc kiến trúc RAG doanh nghiệp quyển 3), nhưng nhiều collection nhỏ → overhead quản lý, khó query xuyên collection.
- **Nguyên tắc staff:** chia theo **ranh giới cách ly & phân quyền** (tenant, độ nhạy dữ liệu), không chia bừa. Đây là quyết định kiến trúc, không phải chi tiết code.

### 4.2. Chi phí — bottleneck thật nằm ở *sinh embedding*

➕ (Nhắc lại xuyên suốt loạt) Bài gốc coi "generate embeddings" là một dòng code. Ở scale lớn đó là **khoản tốn kém nhất**:
- **API EF (OpenAI/Cohere):** trả tiền **mỗi document** embed + latency mạng mỗi lần add/query. Embed 100M chunk = hoá đơn lớn. → Batch, cache, cân nhắc self-host.
- **Local/Custom EF:** cần **CPU/GPU**; embed 100M doc bằng GPU tốn thời gian & tiền hạ tầng. → Đây thường đắt hơn cả chi phí lưu/tìm vector.
- **Số chiều = đòn bẩy cost kép:** chiều lớn → embed chậm hơn *và* storage/latency search cao hơn. Cân nhắc model chiều vừa hoặc Matryoshka (quyển 4).

### 4.3. Đổi model / re-embed — quyết định khó đảo ngược

➕ EF là contract (Phần 3.1). Đổi embedding model (nâng cấp chất lượng) → **re-embed toàn bộ + tạo collection mới + swap**. Chiến lược staff: **blue-green** (dựng collection mới song song, embed nền, cắt traffic khi sẵn sàng), **version hoá** tên collection (`docs_v2_bge`), và **giữ id ổn định** để đối chiếu. Lên kế hoạch cost/thời gian re-embed *trước khi* commit chọn model.

### 4.4. Reliability & monitoring

- **Idempotency:** dùng `upsert` + id ổn định để pipeline re-run không tạo bản trùng.
- **Freshness:** add → searchable có độ trễ (index nền, nhắc kiến trúc log-structured quyển 3). Giám sát độ trễ này nếu cần near-real-time.
- **Giám sát:** phân bố distance của query (drift?), tỉ lệ query rỗng/ dưới ngưỡng, latency embed (API EF hay nghẽn), dung lượng collection, số chiều.
- **Failure mode:** model tải ngầm fail (air-gapped), API EF rate-limit/đổi giá, số chiều lệch khi add thủ công, collection phình quá RAM.

### 4.5. Giải thích cho stakeholder & 🎤 System design mẫu

**Cho stakeholder non-technical:** *"Collection giống một ngăn tủ có dán nhãn: nó cất bản gốc, một 'mã ý nghĩa' của tài liệu, và các nhãn để lọc. Chi phí lớn nhất không phải cái tủ, mà là 'bộ não' biến tài liệu thành mã ý nghĩa — chạy nó trên hàng triệu tài liệu tốn GPU/API. Và đổi bộ não đó thì phải làm lại toàn bộ mã, nên ta chọn kỹ từ đầu."*

> **Đề: "Thiết kế lớp lưu trữ vector cho một SaaS RAG đa khách hàng (1000 tenant, mỗi tenant vài trăm nghìn tài liệu), phải cách ly dữ liệu tuyệt đối."**

Hướng trả lời staff:
1. **Ràng buộc trước:** cách ly tuyệt đối → **không** dùng một collection chung với filter (rủi ro rò rỉ); dùng **tenant/database** riêng hoặc collection-per-tenant.
2. **Cấu trúc:** mỗi tenant → database/collection riêng; id dạng `docid:chunk`; metadata gồm `source`, `acl`, `updated_at`.
3. **EF & metric:** chọn **một EF chuẩn** (vd BGE-M3 self-host cho dữ liệu nhạy cảm — nhắc quyển 4), `space="cosine"`, normalize; khoá số chiều.
4. **Ingestion:** chunk → auto-embed (batch) → upsert idempotent; blue-green khi đổi model.
5. **Query:** auto-embed query bằng đúng EF; giới hạn trong collection của tenant (cách ly by design, không post-filter).
6. **Scale/cost:** bottleneck = embed → batch + self-host GPU; giám sát drift & latency; kế hoạch re-embed.
7. **Kết bằng trade-off:** collection-per-tenant (cách ly tốt, overhead quản lý) vs shared+filter (đơn giản, rủi ro rò rỉ). → *đánh đổi*, chọn theo yêu cầu compliance.

---

## Phần 5 — 🎯 CHỐT LẠI ĐỂ ĐI PHỎNG VẤN (Interview Cheatsheet)

### 5.1. Keywords bắt buộc nhớ (định nghĩa 1 dòng)

- **Collection:** container chứa embeddings + documents + metadata, có tên; ~table nhưng schema-less, không JOIN/ACID.
- **Document:** dữ liệu gốc (text/ảnh/video) đi kèm embedding.
- **Embedding:** dãy số **cố định độ dài** biểu diễn ý nghĩa; mỗi số = một dimension.
- **Metadata:** nhãn key–value để **filter** lúc query.
- **Embedding function (EF):** hàm biến document → vector; là **contract cố định** của collection.
- **DefaultEmbeddingFunction:** EF mặc định = all-MiniLM-L6-v2 (384 chiều), local, tải model lần đầu; JS 2026 ở package `@chroma-core/default-embed`.
- **Auto-embed:** đưa `documents`/`query_texts`, Chroma tự sinh embedding bằng EF của collection.
- **`embeddingFunction`** (JS) / **`embedding_function`** (Python): property gắn EF vào collection (KHÔNG phải `embeddings`).
- **Distance metric mặc định = L2** (không phải cosine); đổi bằng `configuration={"hnsw":{"space":"cosine"}}`.
- **cosine distance:** = 1 − cosine similarity; **gần 0 = giống** (Chroma trả cái này).
- **Tenant → Database → Collection:** hierarchy 3 tầng cho multi-tenancy/cách ly.
- **Persistence:** Ephemeral (RAM) / Persistent (đĩa) / Http (server) / Cloud client.
- **upsert vs add:** upsert idempotent (cập nhật theo id); add trùng id dễ lỗi.

### 5.2. Core concepts (nếu chỉ nhớ mấy điều này)

1. **Collection = ngăn tủ chứa (embedding + document + metadata) có tên.** Giống table nhưng schema-less, không JOIN/ACID.
2. **Embedding cố định độ dài; số chiều bị khoá bởi EF.** Mỗi số = một dimension.
3. **EF là contract:** đổi EF = vô hiệu collection → phải re-embed toàn bộ. Query phải cùng EF với add.
4. **Pattern hiện đại: auto-embed** (đưa documents/query_texts) — an toàn hơn generate thủ công (bài gốc dùng cách cũ).
5. **Distance mặc định là L2, không phải cosine** — chọn metric rõ ràng; cosine distance gần 0 = giống.
6. **Bottleneck & chi phí nằm ở *sinh embedding* (GPU/API), không phải lưu/tìm.**
7. **Ở scale: chia collection theo ranh giới cách ly/phân quyền** (tenant), không nhồi một collection.
8. **JS API 2026 đổi:** `embeddingFunction` (không phải `embeddings`), EF tách package, async/await, auto-embed.

### 5.3. Ideas / mental models

- **"Ngăn tủ dán nhãn"** = collection (bản gốc + mã ý nghĩa + nhãn lọc).
- **"Điểm trên bản đồ ý nghĩa"** = embedding; gần = giống.
- **"EF là hợp đồng đóng dấu"** — đổi nó là phải làm lại toàn bộ.
- **"Auto-embed = để cùng một cái cân cân cả hàng lẫn query"** — nhất quán không gian.
- **"Collection ≠ SQL table"** — không JOIN, không transaction, không schema cứng.
- **"Tiền nằm ở bộ não embed, không ở cái tủ"** — cost chủ yếu là sinh embedding.

### 5.4. Code cần thuộc lòng

**(1) Vòng đời tối thiểu — Python (interviewer hay bắt viết):**
```python
import chromadb
client = chromadb.Client()
col = client.create_collection("docs", configuration={"hnsw": {"space": "cosine"}})
col.add(ids=["1","2"], documents=["a puppy","a car"], metadatas=[{"k":"animal"},{"k":"vehicle"}])
res = col.query(query_texts=["dog"], n_results=1, where={"k":"animal"})  # auto-embed + filter
```
*Khi nào dùng:* dựng nhanh knowledge base; nhớ chọn metric + có bước query.

**(2) JS 2026 đúng chuẩn (thay code lỗi thời của bài gốc):**
```javascript
import { ChromaClient } from "chromadb";
import { DefaultEmbeddingFunction } from "@chroma-core/default-embed";
const client = new ChromaClient();
const col = await client.createCollection({
  name: "docs", embeddingFunction: new DefaultEmbeddingFunction(),   // KHÔNG phải `embeddings`
});
await col.add({ ids: ["1"], documents: ["a puppy"] });               // auto-embed
const res = await col.query({ queryTexts: ["dog"], nResults: 1 });
```

**(3) Custom EF (khi cần model riêng / offline):**
```python
from chromadb import Documents, EmbeddingFunction, Embeddings
class MyEF(EmbeddingFunction):
    def __call__(self, input: Documents) -> Embeddings:
        return [my_model.encode(d).tolist() for d in input]   # cùng model cho add & query!
    @staticmethod
    def name(): return "my-ef"
    def get_config(self): return {}
    @staticmethod
    def build_from_config(c): return MyEF()
```

### 5.5. Câu hỏi phỏng vấn thường gặp + gợi ý trả lời (có câu bẫy)

1. **"Collection có phải là table SQL không?"** *(câu bẫy analogy)* → Giống ở mức "container có tên chứa nhiều item", nhưng **schema-less, không JOIN, không ACID**; nghiệp vụ chính là **similarity search**, không phải exact/range.

2. **"Distance metric mặc định của Chroma là gì?"** *(câu bẫy — nhiều người nói cosine)* → **L2 (Euclidean squared)**, không phải cosine. Text embedding thường muốn cosine → phải set `configuration={"hnsw":{"space":"cosine"}}`.

3. **"Đổi embedding model thì dữ liệu đã lưu ra sao?"** *(câu bẫy vận hành)* → EF là contract; vector cũ & mới khác không gian → **re-embed toàn bộ + collection mới** (blue-green). Query cũng phải cùng EF với add.

4. **"Vì sao nên để Chroma auto-embed thay vì generate rồi add?"** → Đảm bảo **query và document cùng EF/cùng không gian**; ít code, ít lỗi. (Code cũ của nhiều tài liệu generate thủ công là dated.)

5. **"Bottleneck khi add hàng triệu document là gì?"** *(câu bẫy — 'chèn index')* → Thường là bước **embed** (GPU/API latency), không phải chèn HNSW. Batch + self-host giúp.

6. **"cosine similarity 0.9 và cosine distance 0.1 — cái nào 'gần'?"** *(bẫy chiều so sánh)* → Cùng nghĩa: similarity **cao** = gần; distance **thấp** = gần. Chroma trả về **distance** (gần 0 = giống).

7. **"1000 khách hàng chung một Chroma, cách ly thế nào?"** → Dùng **tenant/database hoặc collection-per-tenant**, không phải một collection chung + filter (rủi ro rò rỉ). Cách ly by design.

### 5.6. One-liner đắt giá

- *"Collection là **ngăn tủ vector có dán nhãn**, không phải SQL table — không JOIN, không transaction, không schema cứng."*
- *"Embedding function là **hợp đồng đóng dấu của collection**: đổi nó là phải re-embed cả kho, nên tôi chọn model kỹ từ đầu."*
- *"Chroma mặc định **L2 chứ không cosine** — đây là cái bẫy tôi luôn set rõ metric, và nhớ cosine *distance* gần 0 mới là giống."*
- *"Để Chroma **auto-embed** cả documents lẫn query — cùng một cái cân cân cả hai, khỏi lệch không gian."*
- *"Tiền của một hệ vector nằm ở **bộ não sinh embedding (GPU/API)**, không ở cái tủ lưu trữ."*
- *"Cách ly đa khách hàng là **tenant/collection riêng**, không phải một collection chung rồi lọc — cách ly phải by design."*

---

### 📌 Phụ lục: những chỗ bài giảng gốc lỗi thời / thiếu

1. **🚨 Code JS lỗi thời:** property `embeddings` → phải **`embeddingFunction`**; `default_ef.generate` thủ công → bản mới **auto-embed** từ `documents`; `DefaultEmbeddingFunction` giờ ở package **`@chroma-core/default-embed`**; `.then()` → async/await.
2. **🚨 Không nhắc distance metric:** mặc định **L2**, không phải cosine — dễ khiến xếp hạng lệch với text embedding. Phải set `configuration={"hnsw":{"space":"cosine"}}`.
3. **Thiếu bước query:** bài chỉ dạy create + add, bỏ hẳn `query` — nửa quan trọng nhất của vòng đời.
4. **Analogy "collection = relational table" thiếu giới hạn:** không schema/JOIN/ACID.
5. **Không cảnh báo persistence:** client mặc định in-memory, mất dữ liệu khi thoát.
6. **Bỏ qua:** hierarchy tenant/database, EF là contract (số chiều bị khoá, re-embed khi đổi model), chi phí sinh embedding, model tải ngầm lần đầu, upsert/idempotency.

> **Phần đúng & giá trị của bài gốc:** định nghĩa **collection** (container embeddings/documents/metadata) và **embedding** (fixed-length list of numbers, mỗi số một dimension) là chính xác và cô đọng; ví dụ **dog → dogs/puppy/pet/canine/puppies** minh hoạ similarity score rất trực quan; mental model **"embedding như điểm trên bản đồ, collection đo khoảng cách"** rất tốt; có nhắc **batch embedding**. Giữ những phần đó, cập nhật code sang API 2026, thêm bước query + chọn metric, và bổ sung tư duy EF-as-contract + thiết kế ở scale.
