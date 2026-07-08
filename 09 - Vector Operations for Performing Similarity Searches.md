# Giáo trình: Case Study — Similarity Search cho Công ty Hoa (ChromaDB end-to-end)

> Biên soạn theo phong cách "cầm tay chỉ việc" — từ zero đến tư duy Staff Engineer.
> Giảng bằng tiếng Việt, giữ nguyên technical terms tiếng Anh để phục vụ phỏng vấn big tech.
>
> **Đây là giáo trình thứ 7 của loạt về vector database — quyển "ráp mọi thứ lại thành một sản phẩm thật".** Sáu quyển trước dạy lý thuyết & thành phần (HNSW, embeddings, distance metrics, collections, similarity search, RAG). Quyển này lấy một **case study kinh doanh thật** — một công ty hoa muốn gợi ý hoa thay thế khi hết hàng — và đi **end-to-end**: từ "tại sao doanh nghiệp cần" đến code chạy được đủ 5 bước. Đây là dạng bài hay gặp trong phỏng vấn: *"cho một business problem, hãy dựng similarity search."*
>
> **Cảnh báo ngay từ đầu:** đoạn code trong bài giảng gốc **có nhiều bug và pattern lỗi thời** — đây thực ra là *cơ hội học tốt nhất của cả loạt*, vì mỗi bug tương ứng một khái niệm quan trọng. Cụ thể: `embeddings: default_emd` (sai property, phải là `embeddingFunction`), `collection: collectionName` **thừa và sai** bên trong `query()`, `n: 3` (phải là `nResults`), sinh embedding *thủ công* (bản mới auto-embed), và — quan trọng nhất — gọi các con số **> 1 là "similarity score"** trong khi chúng là **L2 distance** (Chroma mặc định L2, không phải cosine). Tôi giữ phần đúng (business case + khung 5 bước rất tốt), sửa toàn bộ code, và chạy thật. Đánh dấu 🛠️ **[Đính chính]** và ➕ **[Mở rộng ngoài bài gốc]**.

---

## Phần 0 — 🗺️ Bản đồ bài học (Overview)

Đây là quyển "thực chiến hoá". Một công ty hoa nhận phàn nàn vì khách đặt hoa màu này lại nhận màu khác (do hết hàng, phải thay thế tùy tiện). Giải pháp: **lưu hoa dưới dạng vector + similarity search** để nhân viên tìm nhanh **hoa thay thế *giống nhất*** khi hết hàng, chốt với khách *trước khi* cắm. Bài giảng gốc trình bày (a) *tại sao doanh nghiệp cần*, (b) **5 vector operations**, và (c) code 5 bước với ChromaDB. Ta sẽ đi qua tất cả — nhưng sửa hết bug và nâng lên tư duy staff.

**Sau khi học xong, bạn sẽ có thể:**
- Trình bày **business case** cho vector DB (không chỉ "vì nó ngầu") — bằng ngôn ngữ chi phí & sự hài lòng khách hàng.
- Nắm **5 vector operations** làm khung tư duy cho *mọi* dự án similarity search.
- Viết code ChromaDB **chạy được, đúng API 2026** cho toàn bộ vòng đời: setup → prepare → add → fetch → search.
- Nhận ra & sửa **5 bug kinh điển** trong code kiểu bài giảng (property sai, param thừa, `n` vs `nResults`, generate thủ công, distance vs score).
- Hiểu vì sao "score" trong bài gốc **> 1** (L2 distance), và khi nào nên dùng **metadata filter** thay vì để embedding "đoán màu".
- Thiết kế hệ này cho một florist thật với hàng triệu SKU, tồn kho thay đổi liên tục.

**Mạch kiến thức từ basic → staff:**
- 🟢 **Basic:** business problem → analogy → 5 vector operations → hello-world.
- 🟡 **Intermediate:** 5 bước code (setup→prepare→add→fetch→search) với API đúng 2026 → sửa bug bài gốc → lỗi thường gặp.
- 🔴 **Advanced:** distance vs score (vì sao > 1?), L2 vs cosine cho hoa, khi nào metadata filter thắng embedding, auto-embed vs manual, Big-O.
- 🟣 **Staff:** florist thật ở scale (triệu SKU, tồn kho realtime, cold-start hoa mới, substitution = recommendation), cost, business metrics, system design.
- 🎯 **Cheatsheet:** Keywords, core concepts, code phải thuộc, câu hỏi phỏng vấn + one-liners.

---

## Phần 1 — 🟢 BASIC (Nền tảng)

### 1.1. "Tại sao" — business problem có thật

Công ty hoa nhận **đánh giá xấu** vì khách đặt "hoa hồng đỏ" nhưng hết hàng, nhân viên thay bằng hoa *tùy tiện* (sai màu, sai loại). Hậu quả: khách bực, **rework/redelivery tốn tiền**, rating tụt. 

**Giải pháp:** lưu toàn bộ hoa trong kho dưới dạng vector (theo *đặc điểm*: màu, loại, hình dáng...). Khi hết một loại, nhân viên **similarity search** để tìm hoa *giống nhất còn hàng*, đề xuất cho khách **trước khi cắm**. Kết quả kỳ vọng: khách hài lòng hơn, rating tăng, **chi phí mỗi đơn giảm**. → Đây là *lý do kinh doanh*, không phải "vì công nghệ hay". Một staff luôn bắt đầu từ đây.

Bài gốc nêu 3 lợi ích, tôi giữ: **cải thiện trải nghiệm khách**, **thay thế nhanh khi hết hàng**, **tối ưu tồn kho** (biết hoa nào giống nhau → gom nhóm, giảm lãng phí).

### 1.2. Analogy đời thường

Giống một **nhân viên bán hoa lão luyện**: khách muốn "bó hồng đỏ rực" nhưng hết, cô ấy *tức thì* nghĩ "à, thược dược đỏ hoặc mẫu đơn đỏ trông tương tự đấy". Cô ấy làm điều đó nhờ *hiểu đặc điểm hoa* và *nhớ hoa nào giống hoa nào*. Vector database = "bộ não" mã hoá đặc điểm hoa thành toạ độ; similarity search = động tác "nghĩ ra hoa tương tự". Ta đang *scale* trực giác của một nhân viên giỏi thành hệ thống chạy cho cả nghìn cửa hàng.

### 1.3. Năm vector operations — khung tư duy cho MỌI dự án similarity search

Bài gốc nêu 5 operation rất chuẩn — đây là *checklist vàng* cho bất kỳ hệ similarity search nào:

1. **Data → Vectors:** biến dữ liệu (mô tả text, ảnh, thuộc tính) thành vector số. Text → word embedding; ảnh → feature extraction (quyển 4–5).
2. **Similarity metric:** chọn cách đo gần/xa (cosine, Euclidean...) (quyển 4).
3. **Query → Vector:** biến câu truy vấn của user thành vector *bằng đúng phép biến đổi đã dùng cho data* (nguyên tắc nhất quán — quyển 5).
4. **Search:** tìm các vector trong collection gần nhất với query vector (quyển 1 & 6).
5. **Retrieve & Present:** lấy document + metadata (category, similarity score) trình bày cho user.

→ Nhớ 5 bước này là nhớ *xương sống* của mọi ứng dụng vector. Mỗi lần dựng hệ mới, chạy qua checklist này.

### 1.4. Ví dụ chạy tay — "yellow flowers" tìm hoa gì?

Kho 3 hoa, embed theo màu (đỏ/vàng/trắng) đơn giản:

| ID | Hoa | Vector [đỏ, vàng, trắng] |
|---|---|---|
| flower_1 | "A vibrant red rose" | `[1, 0, 0]` |
| flower_2 | "A sunny yellow tulip" | `[0, 1, 0]` |
| flower_3 | "A pure white lily" | `[0, 0, 1]` |

Query "yellow flowers" → vector `[0, 1, 0]`. Đo khoảng cách (L2):
- tới flower_2 (vàng): **0** → gần nhất 🥇
- tới flower_1 (đỏ): `√(1+1) ≈ 1.41`
- tới flower_3 (trắng): `√(1+1) ≈ 1.41`

→ Kết quả: flower_2 (tulip vàng) khớp nhất. Đúng trực giác. Chú ý: khoảng cách tới hoa *không* vàng **> 1** — ghi nhớ chi tiết này, nó giải thích output "score" của bài gốc ở Phần 3.

### 1.5. Code "hello world" — florist search (Python, đã chạy thật)

Bài gốc dùng JavaScript (và có bug); tôi khởi động bằng Python chạy được ngay, JS đúng chuẩn ở Phần 2.

```python
import chromadb
from chromadb import Documents, EmbeddingFunction, Embeddings

# EF offline mô phỏng "màu hoa" -> vector (thực tế dùng model thật, quyển 5)
class FlowerEF(EmbeddingFunction):
    def __call__(self, input: Documents) -> Embeddings:
        out = []
        for d in input:
            d = d.lower()
            out.append([float("red" in d), float("yellow" in d), float("white" in d)])
        return out
    @staticmethod
    def name(): return "flower-ef"
    def get_config(self): return {}
    @staticmethod
    def build_from_config(c): return FlowerEF()

client = chromadb.Client()
col = client.get_or_create_collection("flowers", embedding_function=FlowerEF())
col.add(
    ids=["flower_1", "flower_2", "flower_3"],
    documents=["A vibrant red rose", "A sunny yellow tulip", "A pure white lily"],
)   # auto-embed từ documents

res = col.query(query_texts=["yellow flowers"], n_results=3)
for i in range(len(res["ids"][0])):
    print(res["ids"][0][i], round(res["distances"][0][i], 3), res["documents"][0][i])
# flower_2 0.0  'A sunny yellow tulip'    <- khớp nhất (distance nhỏ nhất)
# flower_3 ...  'A pure white lily'
# flower_1 ...  'A vibrant red rose'
```

### 1.6. ✅ Self-check (Basic)

1. Vì sao công ty hoa cần vector DB — nêu bằng ngôn ngữ *kinh doanh*, không phải kỹ thuật?
2. Kể lại 5 vector operations. Bước nào đảm bảo query và data "so được với nhau"?
3. Trong ví dụ, vì sao khoảng cách từ "yellow" tới hoa đỏ/trắng lớn hơn 1?

---

## Phần 2 — 🟡 INTERMEDIATE (Vận dụng)

### 2.1. Năm bước triển khai — và sửa bug bài gốc

Bài gốc chia 5 bước: **setup → prepare data → add → fetch all → search**. Khung đúng, nhưng code có bug. Ta đi từng bước với **API đúng 2026**.

🛠️ **[Đính chính — tổng hợp 5 bug/pattern lỗi thời trong code bài gốc]**

| # | Code bài gốc (sai/cũ) | Sửa đúng 2026 | Vì sao |
|---|---|---|---|
| 1 | `getOrCreateCollection({ embeddings: default_emd })` | `embeddingFunction: embedder` | Property đúng là **`embeddingFunction`**, không phải `embeddings` (quyển 5) |
| 2 | `collection.query({ collection: collectionName, ... })` | Bỏ `collection:` | `query()` gọi **trên collection object**; không truyền tên collection vào |
| 3 | `n: 3` | `nResults: 3` | Tham số top-k tên là **`nResults`** (JS) / `n_results` (Python) (quyển 6) |
| 4 | `default_emd.generate(...)` rồi `add({ embeddings })` | chỉ `add({ documents })` | Gắn EF vào collection → **auto-embed**, khỏi generate thủ công (quyển 5) |
| 5 | `require('chromadb')` lấy `DefaultEmbeddingFunction` | `@chroma-core/default-embed` | EF **tách package riêng** từ JS V3 (quyển 5) |

### 2.2. Code JS 2026 đúng chuẩn (thay code lỗi của bài gốc)

```javascript
// npm install chromadb @chroma-core/default-embed
const { ChromaClient } = require("chromadb");
const { DefaultEmbeddingFunction } = require("@chroma-core/default-embed"); // package RIÊNG

const client = new ChromaClient();
const embedder = new DefaultEmbeddingFunction();
const collectionName = "my_flower_collection";

async function main() {
  try {
    // BƯỚC 1+2: tạo collection với embeddingFunction (KHÔNG phải `embeddings`)
    const collection = await client.getOrCreateCollection({
      name: collectionName,
      embeddingFunction: embedder,        // ĐÚNG property
    });

    // BƯỚC 2: chuẩn bị dữ liệu hoa
    const flowers = ["A vibrant red rose", "A sunny yellow tulip", "A pure white lily"];
    const ids = flowers.map((_, i) => `flower_${i + 1}`);

    // BƯỚC 3: add — chỉ đưa documents, Chroma TỰ embed (không generate thủ công)
    await collection.add({ ids, documents: flowers });

    // BƯỚC 4: fetch all để kiểm tra
    const allItems = await collection.get();
    console.log(allItems);

    // BƯỚC 5: similarity search
    await performSimilaritySearch(collection);
  } catch (err) {
    console.error("Error:", err);            // error handling (bài gốc đúng ý này)
  }
}

async function performSimilaritySearch(collection) {
  const results = await collection.query({
    queryTexts: ["yellow flowers"],          // Chroma tự embed query
    nResults: 3,                             // ĐÚNG: nResults, KHÔNG `n`, KHÔNG `collection:`
  });
  // query đã trả documents & distances trực tiếp -> không cần dò allItems
  const { ids, documents, distances } = results;
  for (let i = 0; i < ids[0].length; i++) {
    console.log(`ID: ${ids[0][i]}, Text: '${documents[0][i]}', Distance: ${distances[0][i]}`);
  }
}
main();
```

### 2.3. Bản Python đầy đủ (đã chạy thật)

```python
import chromadb
from chromadb import Documents, EmbeddingFunction, Embeddings

class FlowerEF(EmbeddingFunction):
    def __call__(self, input: Documents) -> Embeddings:
        return [[float("red" in d.lower()), float("yellow" in d.lower()),
                 float("white" in d.lower())] for d in input]
    @staticmethod
    def name(): return "flower-ef"
    def get_config(self): return {}
    @staticmethod
    def build_from_config(c): return FlowerEF()

client = chromadb.Client()
col = client.get_or_create_collection("my_flower_collection",
                                      embedding_function=FlowerEF())
flowers = ["A vibrant red rose", "A sunny yellow tulip", "A pure white lily"]
col.add(ids=[f"flower_{i+1}" for i in range(len(flowers))], documents=flowers)

res = col.query(query_texts=["yellow flowers"], n_results=3)
for i in range(len(res["ids"][0])):
    print(f"ID: {res['ids'][0][i]}, Text: {res['documents'][0][i]!r}, "
          f"Distance: {res['distances'][0][i]:.3f}")
# ID: flower_2, Text: 'A sunny yellow tulip', Distance: 0.040   <- best (nhỏ nhất)
# ID: flower_3, Text: 'A pure white lily',   Distance: 2.010
# ID: flower_1, Text: 'A vibrant red rose',  Distance: 2.018
```
**Kết quả thật:** tulip vàng khớp nhất (distance 0.040), hai hoa còn lại xa (≈2.0). Khớp narrative bài gốc — nhưng chú ý **distance ≈ 2 > 1** (giải thích ở Phần 3).

### 2.4. 3 lỗi thường gặp (và cách tránh)

1. **Copy code có `embeddings:` / `n:` / `collection:` từ tài liệu cũ.** → Dùng `embeddingFunction`, `nResults`/`n_results`, bỏ `collection:`.
2. **Gọi "distance" là "similarity score" rồi hiểu ngược chiều.** → Chroma trả **distance** (thấp = giống). "Best match = lowest distance", không phải highest score.
3. **Để embedding "đoán màu" trong khi màu là thuộc tính rõ ràng.** → Màu/loại hoa nên là **metadata** để *filter chính xác*, không phó mặc embedding (xem Phần 3).

### 2.5. ✅ Self-check (Intermediate)

1. Chỉ ra 3 bug trong dòng `collection.query({ collection: collectionName, queryTexts: [q], n: 3 })` và sửa.
2. Vì sao không cần `default_emd.generate(...)` nữa?
3. Trong kết quả, "best match" là distance cao hay thấp?

---

## Phần 3 — 🔴 ADVANCED (Chuyên sâu)

### 3.1. 🛠️ "Score" trong bài gốc THỰC RA là L2 distance — vì sao > 1?

Output bài gốc: flower_2 = **0.926**, flower_1 = **1.040**, flower_3 = **1.264**, và nói *"lowest similarity score = best match"*. Có hai vấn đề:

**(a) Đây là *distance*, không phải *similarity score*.** Bài lấy giá trị từ `results.distances` nhưng gọi là "score". Distance **thấp = giống** (nên "lowest = best" mới đúng). Còn "similarity score" theo nghĩa thông thường thì **cao = giống**. Gọi distance là "similarity score" là mâu thuẫn nội tại (quyển 6).

**(b) Vì sao giá trị > 1?** Cosine similarity ∈ [0,1] (thực ra [−1,1], quyển 4) — *không thể* ra 1.264. Vậy đây **không phải cosine**. 🛠️ **ChromaDB mặc định dùng L2 (Euclidean distance squared)** khi bạn không set metric (quyển 5). Bài gốc *không* set `space`, nên nhận **L2 squared distance** — và L2 dễ dàng > 1. **Kiểm chứng bằng code (đã chạy):**

```python
# space="l2" (MẶC ĐỊNH) -> distance có thể > 1  (giống output bài gốc)
#   flower_2 (khớp): 0.040   flower_3: 2.010   flower_1: 2.018
# space="cosine"    -> distance trong [0, 2]
#   flower_2: 0.007   flower_1: 0.441   flower_3: 0.443
```
→ Cùng thứ hạng (tulip vàng nhất), nhưng **thang đo khác hẳn**. Bài học: **luôn biết mình đang dùng metric gì**; đừng gọi distance là "similarity". Và nhớ (quyển 4): với vector normalize, L2 và cosine cho *cùng thứ hạng* — nên thứ hạng đúng, chỉ con số khác.

### 3.2. Khi nào metadata filter THẮNG embedding — bài học "yellow flowers"

➕ **[Mở rộng — điểm staff tinh tế nhất của bài này]** Bài gốc để **embedding "đoán" màu vàng** từ text "yellow flowers". Nhưng với một florist thật, **màu là thuộc tính có cấu trúc rõ ràng** — nên xử lý bằng **metadata filter**, không phó mặc embedding:

```python
# Thay vì hy vọng embedding bắt được "vàng", lưu màu làm metadata:
col.add(ids=[...], documents=flowers,
        metadatas=[{"color":"red","type":"rose"},
                   {"color":"yellow","type":"tulip"},
                   {"color":"white","type":"lily"}])
# Query: hoa GIỐNG (embedding) NHƯNG chỉ màu vàng (filter chính xác)
res = col.query(query_texts=["bright cheerful bouquet"], n_results=3,
                where={"color":"yellow"})
```
Vì sao quan trọng? Embedding có thể *nhầm* "yellow" nếu mô tả không chứa chữ đó, hoặc lẫn "golden/amber". Filter `where={"color":"yellow"}` là **chính xác tuyệt đối**. → Nguyên tắc staff: **thuộc tính rời rạc, rõ ràng (màu, loại, giá, mùa) → metadata filter; điểm mờ, ngữ nghĩa (phong cách, cảm xúc bó hoa) → embedding.** Kết hợp cả hai = **hybrid** (quyển 3 & 6). Đây là điều case study "3 bông hoa" của bài gốc quá nhỏ để lộ ra.

### 3.3. Auto-embed vs manual generate — và cái bẫy nhất quán

➕ Bài gốc `generate` embedding *thủ công* rồi truyền `embeddings:` vào `add`. Vấn đề: nếu lúc **query** bạn quên embed câu query *bằng đúng model đó*, kết quả là rác (quyển 5). Auto-embed (gắn EF vào collection) loại bỏ rủi ro này — Chroma đảm bảo **add và query dùng cùng EF**. Đây là lý do pattern manual bị coi là dated. *Ngoại lệ hợp lệ:* khi bạn embed offline bằng pipeline riêng (GPU batch) rồi nạp vector sẵn — lúc đó truyền `embeddings` là cố ý, nhưng vẫn phải tự lo query dùng cùng model.

### 3.4. Big-O & cấu trúc

- **add:** embed (O(số token) mỗi doc — phần đắt, nhất là API EF) + chèn HNSW ~O(log N) (quyển 1 & 5).
- **query:** embed query + HNSW search ~O(log N) (quyển 6). Kết quả **approximate** (ANN), không phải exact — bài gốc không nhắc.
- **get() (fetch all):** O(N) — bài gốc dùng `collection.get()` để "kiểm tra data"; ổn với 3 hoa, nhưng **với triệu SKU, `get()` không filter là O(N) tốn kém** → chỉ dùng để debug, đừng dùng trong hot path.

### 3.5. Edge cases phải xử lý

- **top-3 luôn trả 3 kể cả rác** (quyển 6): hoa đỏ/trắng vẫn lọt top-3 cho query "vàng" → cần **threshold** distance để chỉ đề xuất hoa *đủ giống*.
- **Hết sạch loại tương tự:** nếu không hoa nào dưới ngưỡng → nói "không có thay thế phù hợp", đừng ép đề xuất hoa lệch.
- **Mô tả nghèo nàn:** "A pure white lily" ít đặc trưng → embedding yếu. → Làm giàu mô tả (thêm hình dáng, mùa, dịp) hoặc thêm metadata.
- **Đơn vị/định dạng không nhất quán** giữa các mô tả hoa → embedding lệch.

---

## Phần 4 — 🟣 STAFF LEVEL (Tư duy hệ thống & lãnh đạo kỹ thuật)

### 4.1. Từ "3 bông hoa" tới florist thật — scale & bottleneck

➕ Case study có 3 hoa; florist thật có **hàng chục nghìn–triệu SKU** (loại × màu × kích cỡ × nhà cung cấp), nhiều cửa hàng, tồn kho đổi *từng giờ*. Những gì bài gốc giấu:

- **Tồn kho realtime là bài toán khó nhất:** "hoa thay thế" chỉ hữu ích nếu *còn hàng ở cửa hàng đó ngay bây giờ*. → Kết hợp similarity search với **filter `where={"in_stock": true, "store_id": ...}`** và **freshness pipeline** (cập nhật stock liên tục — nhắc log-structured/freshness quyển 3). Vector "giống" mà hết hàng thì vô dụng.
- **Substitution = recommendation:** đề xuất hoa thay thế *chính là* một recommendation problem (quyển 2) — có thể nâng lên **two-tower** (khách này + hoa này → gợi ý), thêm tín hiệu "khách thường chấp nhận thay thế nào".
- **Cold-start:** hoa/nhà cung cấp mới chưa có mô tả tốt → embedding yếu → fallback theo metadata (màu/loại) cho tới khi đủ dữ liệu.
- **Bottleneck:** không phải vector search (vài triệu SKU thì Chroma/Qdrant thừa sức), mà là **chất lượng mô tả hoa (data quality)** + **đồng bộ tồn kho**. Rác vào → gợi ý rác ra.

### 4.2. Trade-off kiến trúc: khi nào KHÔNG cần vector DB ở đây

Staff biết nói "không":
- **Nếu thay thế chỉ theo luật rõ** ("hồng đỏ hết → luôn thay bằng thược dược đỏ") → một **bảng mapping/quy tắc** đơn giản rẻ và minh bạch hơn; không cần embedding.
- **Nếu chỉ lọc theo màu/loại/giá** (thuộc tính rời rạc) → **SQL/metadata filter** là đủ; embedding chỉ thêm giá trị khi cần "giống về *phong cách/cảm giác*" (bó hoa lãng mạn vs rực rỡ) — điểm mờ khó mô tả bằng luật.
- **Dữ liệu nhỏ (vài trăm SKU tĩnh)** → không cần hạ tầng vector; brute-force cosine trong bộ nhớ là xong (quyển 1).
→ Vector DB *đáng giá* khi cần **similarity ngữ nghĩa mờ + quy mô lớn + kết hợp filter**; đừng "AI hoá" một bài toán mapping.

### 4.3. Chi phí, vận hành, monitoring — nối với business metric

- **Cost:** với API EF, embed mỗi SKU/mỗi query tốn tiền; ở triệu SKU nên **self-host EF** hoặc batch offline (quyển 2 & 5). Số chiều = đòn bẩy storage/latency.
- **Monitoring phải nối với business:** không chỉ recall@k/latency, mà **tỉ lệ khách chấp nhận đề xuất thay thế**, **chi phí rework/redelivery**, **rating/satisfaction** — vì đó mới là lý do dự án tồn tại (bài gốc nêu đúng mục tiêu này). Một staff *đóng vòng* từ chỉ số hệ thống tới chỉ số kinh doanh.
- **Failure modes:** stock lỗi thời (đề xuất hoa đã hết), mô tả nghèo → gợi ý lệch, EF tải model fail (quyển 5), threshold sai → đề xuất rác.

### 4.4. Ảnh hưởng tổ chức & 🎤 System design mẫu

**Cho stakeholder non-technical:** *"Ta dạy hệ thống 'hiểu' hoa nào giống hoa nào theo đặc điểm, để khi hết hàng, nhân viên đề xuất thay thế hợp gu khách *trước khi cắm* — bớt giao sai, bớt làm lại, khách hài lòng hơn. Chi phí lớn nằm ở việc mô tả hoa cho chuẩn và đồng bộ tồn kho, không ở khâu tìm kiếm."*

> **Đề: "Thiết kế hệ gợi ý hoa thay thế cho chuỗi 500 cửa hàng, 200k SKU, tồn kho cập nhật mỗi 5 phút, phải tôn trọng tồn kho từng cửa hàng."**

Hướng trả lời staff:
1. **Làm rõ trước:** "giống" theo màu/loại (rời rạc) hay phong cách (mờ)? → quyết định embedding vs filter. Tồn kho theo *từng cửa hàng* → filter bắt buộc.
2. **Data model:** mỗi SKU = document (mô tả giàu) + metadata (`color`, `type`, `price`, `season`, `store_id`, `in_stock`, `updated_at`).
3. **Pipeline:** mô tả → embed (batch, EF nhất quán) → collection; **hybrid query**: embedding (giống phong cách) + `where` (màu nếu khách chỉ định + `in_stock` + `store_id`).
4. **Freshness:** stock cập nhật 5 phút → upsert metadata; đề xuất *phải* lọc `in_stock=true` tại `store_id`.
5. **Threshold + fallback:** dưới ngưỡng giống → "không có thay thế phù hợp" thay vì ép.
6. **Scale/cost:** 200k SKU nhỏ với vector DB; bottleneck là data quality + sync tồn kho; self-host EF nếu cost cao.
7. **Monitoring:** tỉ lệ chấp nhận thay thế, rework cost, rating — nối business.
8. **Kết bằng trade-off:** embedding vs rule-based mapping; hybrid; threshold chặt/lỏng. → *đánh đổi*.

---

## Phần 5 — 🎯 CHỐT LẠI ĐỂ ĐI PHỎNG VẤN (Interview Cheatsheet)

### 5.1. Keywords bắt buộc nhớ (định nghĩa 1 dòng)

- **5 vector operations:** data→vector, similarity metric, query→vector, search, retrieve&present — checklist mọi dự án.
- **Similarity search:** tìm item giống nhất với query (quyển 6).
- **Embedding function (EF):** biến document→vector; contract của collection (quyển 5).
- **DefaultEmbeddingFunction:** all-MiniLM-L6-v2, 384 chiều; JS 2026 ở `@chroma-core/default-embed`.
- **`embeddingFunction`** (property đúng) vs `embeddings` (sai trong bài gốc).
- **`nResults`/`n_results`:** top-k; KHÔNG phải `n`.
- **`where`:** metadata filter trong query.
- **distance vs similarity score:** Chroma trả **distance** (thấp=giống); score nghĩa thường cao=giống.
- **L2 (default) vs cosine:** Chroma mặc định **L2 squared** (distance có thể >1); set `configuration={"hnsw":{"space":"cosine"}}` để đổi.
- **auto-embed:** đưa `documents`/`query_texts`, Chroma tự embed (an toàn hơn generate thủ công).
- **metadata filter vs embedding:** thuộc tính rời rạc rõ ràng → filter; ngữ nghĩa mờ → embedding; kết hợp = hybrid.
- **substitution = recommendation:** gợi ý hoa thay thế là bài toán recommendation (quyển 2).
- **freshness / in_stock:** đề xuất phải tôn trọng tồn kho realtime.

### 5.2. Core concepts (nếu chỉ nhớ mấy điều này)

1. **Bắt đầu từ business case** (giảm rework, tăng satisfaction), không phải từ công nghệ.
2. **5 vector operations là xương sống** của mọi hệ similarity search.
3. **Score trong bài gốc là L2 *distance* (thấp=giống, >1 vì L2 mặc định), KHÔNG phải similarity score.**
4. **API đúng:** `embeddingFunction` (không `embeddings`), `nResults` (không `n`), bỏ `collection:` trong query, auto-embed (không generate thủ công).
5. **Thuộc tính rời rạc (màu/loại/giá) → metadata filter; ngữ nghĩa mờ → embedding; kết hợp = hybrid.**
6. **Bottleneck thật = data quality + tồn kho realtime,** không phải vector search.
7. **top-k luôn trả đủ k → cần threshold** để không đề xuất hoa lệch.
8. **Nối chỉ số hệ thống với business metric** (tỉ lệ chấp nhận thay thế, rework cost, rating).

### 5.3. Ideas / mental models

- **"Nhân viên bán hoa lão luyện"** nghĩ ra hoa tương tự = similarity search được scale.
- **"5 operations là checklist"** — chạy qua mỗi lần dựng hệ mới.
- **"Distance đi xuống, score đi lên"** — bài gốc gọi distance là score là lẫn.
- **"Score > 1 ⇒ không phải cosine"** — dấu hiệu đang dùng L2 mặc định.
- **"Màu là filter, phong cách là embedding"** — đừng bắt embedding đoán thứ đã rõ.
- **"Vector giống mà hết hàng thì vô dụng"** — luôn kèm filter tồn kho.
- **"Rác vào, rác ra"** — chất lượng mô tả quyết định chất lượng gợi ý.

### 5.4. Code cần thuộc lòng

**(1) Vòng đời florist đúng chuẩn — Python:**
```python
col = client.get_or_create_collection("flowers", embedding_function=EF(),
                                      configuration={"hnsw":{"space":"cosine"}})  # chọn metric!
col.add(ids=ids, documents=flowers, metadatas=metas)                # auto-embed + metadata
res = col.query(query_texts=["yellow flowers"], n_results=3, where={"in_stock": True})
best = [(d, dist) for d, dist in zip(res["documents"][0], res["distances"][0]) if dist < THRESH]
```

**(2) Sửa 3 bug trong một dòng (interviewer hay bắt tìm lỗi):**
```javascript
// SAI:  collection.query({ collection: name, queryTexts:[q], n: 3 })
// ĐÚNG: collection.query({ queryTexts:[q], nResults: 3 })   // bỏ collection:, n->nResults
```

**(3) Hybrid: embedding + metadata filter:**
```python
col.query(query_texts=["romantic bright bouquet"],  # ngữ nghĩa mờ -> embedding
          n_results=5, where={"color":"yellow", "in_stock": True})  # rời rạc -> filter
```

### 5.5. Câu hỏi phỏng vấn thường gặp + gợi ý trả lời (có câu bẫy)

1. **"5 bước dựng similarity search là gì?"** → data→vector, chọn metric, query→vector (cùng phép biến đổi!), search, retrieve&present.

2. **"Output score 1.264 nghĩa là gì? Sao lại > 1?"** *(câu bẫy cosine)* → Đó là **L2 distance** (mặc định của Chroma), không phải cosine similarity (∈[0,1]). Là **distance** → thấp = giống. Muốn cosine thì set `space="cosine"`.

3. **"Tìm 3 bug trong `query({ collection: name, queryTexts:[q], n:3 })`."** *(câu bẫy code)* → (1) `collection:` thừa (query gọi trên object); (2) `n` phải là `nResults`; (3) [nếu collection tạo bằng `embeddings:`] property sai, phải `embeddingFunction`.

4. **"Query 'yellow flowers' — nên dùng embedding hay metadata filter cho màu?"** *(câu bẫy thiết kế)* → Màu là thuộc tính **rời rạc rõ ràng** → **metadata filter** chính xác hơn; embedding cho phần *phong cách/cảm giác*. Kết hợp = hybrid.

5. **"Hệ gợi ý được hoa giống nhưng khách phàn nàn — vì sao?"** *(câu bẫy vận hành)* → Có thể hoa đề xuất **đã hết hàng** (thiếu filter `in_stock`), hoặc **mô tả nghèo** (data quality), hoặc **không threshold** nên đề xuất hoa lệch. Debug retrieval + stock trước.

6. **"Khi nào KHÔNG nên dùng vector DB cho florist?"** → Thay thế theo **luật rõ** (mapping table), chỉ lọc **thuộc tính rời rạc** (SQL), hoặc **data nhỏ tĩnh** (brute-force). Vector DB đáng giá khi cần similarity *mờ* + scale + hybrid.

7. **"Vì sao auto-embed an toàn hơn generate thủ công?"** → Đảm bảo query & document **cùng EF/cùng không gian**; generate thủ công dễ khiến query dùng model khác → rác.

### 5.6. One-liner đắt giá

- *"Tôi bắt đầu từ **business case** — giảm rework, tăng satisfaction — rồi mới tới vector; công nghệ phục vụ chỉ số kinh doanh."*
- *"Cái bài gốc gọi 'similarity score 1.264' thực ra là **L2 distance** — Chroma mặc định L2, và distance thì **thấp mới là giống**."*
- *"Màu hoa là **metadata filter**, phong cách bó hoa là **embedding** — đừng bắt embedding đoán thứ đã rõ ràng."*
- *"Vector 'giống' mà **hết hàng thì vô dụng** — mọi đề xuất phải lọc tồn kho realtime của đúng cửa hàng."*
- *"Bottleneck của hệ này là **chất lượng mô tả + đồng bộ tồn kho**, không phải tốc độ vector search."*
- *"Ba lỗi trong một dòng query: `collection:` thừa, `n` phải là `nResults`, và collection phải tạo bằng `embeddingFunction` — đó là dấu hiệu code chép từ tài liệu cũ."*

---

### 📌 Phụ lục: những chỗ code bài giảng gốc sai / lỗi thời

1. **🚨 `getOrCreateCollection({ embeddings: default_emd })`** → property đúng là **`embeddingFunction`**.
2. **🚨 `collection.query({ collection: collectionName, ... })`** → **bỏ `collection:`**; `query()` gọi trên collection object.
3. **🚨 `n: 3`** → phải là **`nResults: 3`** (JS) / `n_results=3` (Python).
4. **🚨 Gọi `results.distances` là "similarity score"** → là **distance** (thấp=giống); và **>1 vì mặc định L2**, không phải cosine.
5. **`default_emd.generate()` + `add({ embeddings })`** → pattern cũ; nên **auto-embed** từ `documents`.
6. **`require('chromadb')` lấy `DefaultEmbeddingFunction`** → nay ở package **`@chroma-core/default-embed`**.
7. **`allItems.documents[allItems.ids.indexOf(id)]`** → thừa; `query()` đã trả `documents` trực tiếp.
8. **Không set distance metric & không có threshold** → dễ hiểu nhầm score và đề xuất rác.

> **Phần đúng & giá trị của bài gốc:** **business framing** (giảm rework, thay thế nhanh, tối ưu tồn kho) rất chuẩn và là cách mở đầu đúng của một staff; **5 vector operations** là checklist xuất sắc cho mọi dự án similarity search; khung **5 bước triển khai** (setup→prepare→add→fetch→search) đúng trình tự; **error handling bằng try/catch** là thói quen tốt. Giữ những phần đó, sửa toàn bộ code sang API 2026, làm rõ distance-vs-score + L2-vs-cosine, và bổ sung metadata-filter/tồn-kho/threshold + tư duy scale mà case "3 bông hoa" chưa lộ ra.
