# Giáo trình: Vector Databases — từ Zero đến Staff Engineer

> Tái cấu trúc từ bài giảng gốc "Essential vector database concepts", giảng lại + đào sâu + mở rộng theo tư duy staff-level.
> Mọi thuật ngữ kỹ thuật giữ nguyên tiếng Anh. Code đã được chạy thử thật (numpy). Thông tin thị trường được cập nhật tới 2026.
>
> ⚠️ Bài giảng gốc rất cơ bản (chỉ nói "vector = mảng số" + "similarity search" + ví dụ sách). Phần lớn nội dung từ tầng INTERMEDIATE trở lên là **mở rộng ngoài bài gốc** — được đánh dấu rõ bằng `[MỞ RỘNG]`.

---

## Phần 0 — 🗺️ Bản đồ bài học (Overview)

**Bài này dạy gì.** Vector database là loại database lưu dữ liệu dưới dạng *vector* (mảng số nhiều chiều) và trả lời câu hỏi **"cái gì giống với cái này nhất?"** thay vì "hàng nào bằng đúng giá trị này?". Nó ra đời vì database truyền thống (SQL) chỉ giỏi *exact match* và *range query*, nhưng bất lực trước câu hỏi "tìm ảnh giống ảnh này", "gợi ý sản phẩm tương tự", "tìm đoạn văn có ý nghĩa gần với câu hỏi". Đây là hạ tầng cốt lõi của recommendation systems, semantic search, và đặc biệt là RAG (Retrieval-Augmented Generation) cho các ứng dụng LLM.

**Sau khi học xong bạn sẽ làm được:**
- Giải thích *tại sao* vector DB tồn tại và khi nào KHÔNG cần nó.
- Hiểu vector/embedding đến từ đâu, chọn đúng distance metric (cosine / dot / L2).
- Hiểu cơ chế bên trong: exact search vs ANN, thuật toán HNSW / IVF / PQ, và độ phức tạp Big-O.
- Phân tích trade-off recall ↔ latency ↔ memory ↔ cost.
- Thiết kế một hệ thống semantic search cho hàng trăm triệu bản ghi và bảo vệ thiết kế đó trong phỏng vấn.

**Mạch kiến thức basic → staff:**
- 🟢 **Basic:** Vấn đề exact-match → vector là gì → distance → tìm nearest bằng tay.
- 🟡 **Intermediate:** Embedding từ đâu ra → cosine vs dot vs L2 → normalization → search thật với thư viện.
- 🔴 **Advanced:** Vì sao brute-force O(N·d) không scale → curse of dimensionality → ANN (HNSW/IVF/PQ) → recall@k → tự implement IVF.
- 🟣 **Staff:** Toán bộ nhớ ở tỷ-scale → sharding/quantization → landscape 2026 (pgvector/Pinecone/Qdrant/Weaviate/Milvus) → hybrid search → cost/ops → system design.
- 🎯 **Cheatsheet:** Keywords, core concepts, mental models, code thuộc lòng, Q&A phỏng vấn.

---

## Phần 1 — 🟢 BASIC (Nền tảng)

### 1.1. Vấn đề thực tế trước đã

Bạn có một database SQL chứa 1 triệu cuốn sách. Bạn viết được ngay:

```sql
SELECT * FROM books WHERE title = 'Dune';          -- exact match: OK
SELECT * FROM books WHERE pages BETWEEN 200 AND 300; -- range: OK
```

Nhưng sếp hỏi: **"Tìm cho tôi những cuốn *giống* Dune."** Giống về cái gì? Về không khí, chủ đề, giọng văn — những thứ không nằm gọn trong một cột nào. SQL chịu chết, vì `WHERE` chỉ so sánh được giá trị *bằng nhau* hoặc *lớn/nhỏ hơn*, chứ không so sánh được *mức độ giống nhau* về ngữ nghĩa.

> Đây chính là điểm bài giảng gốc nói: dữ liệu phức tạp (ảnh, âm thanh, quan hệ xã hội, dữ liệu gen) rất khó lưu và phân tích bằng hệ truyền thống, thường phải pre-process rất nhiều.

### 1.2. Analogy đời thường

Hãy tưởng tượng **hai cách sắp xếp thư viện**:

- **Cách SQL:** xếp sách theo mã ISBN. Muốn tìm đúng 1 cuốn thì siêu nhanh. Nhưng muốn tìm "sách hay giống cuốn này" thì vô dụng, vì hai cuốn giống nhau về nội dung lại nằm cách nhau cả cây số trên kệ.
- **Cách vector DB:** xếp sách trong một **không gian nhiều chiều** sao cho *sách nội dung càng giống nhau thì đứng càng gần nhau*. Giờ muốn tìm sách tương tự, bạn chỉ cần đứng ở vị trí cuốn đang xét rồi **với tay ra xung quanh** — hàng xóm gần nhất chính là kết quả.

Đó đúng là định nghĩa trong bài gốc: vector DB *tổ chức các data point trong không gian đa chiều dựa trên độ gần (proximity) của chúng*, và việc tìm hàng xóm gần nhất gọi là **similarity search**.

### 1.3. Định nghĩa đơn giản (giải thích mọi thuật ngữ)

- **Vector:** một **mảng các con số**, ví dụ `[180, 2010, 4.6]`. Về mặt toán, nó là một điểm/mũi tên trong không gian; bài gốc nói vector "được định nghĩa bởi size (độ dài) và direction (hướng)".
- **Dimension (chiều):** mỗi con số trong mảng là một dimension. `[180, 2010, 4.6]` là vector 3 chiều. Embedding thật thường 384–3072 chiều.
- **Feature/attribute:** mỗi chiều tương ứng một đặc trưng của dữ liệu (số trang, năm xuất bản, rating...).
- **Similarity search / Nearest Neighbor search:** cho một vector query, tìm các vector *gần nó nhất* trong DB.
- **Proximity (độ gần):** đo bằng một hàm khoảng cách (distance). Gần = giống.

### 1.4. Ví dụ chạy tay (làm thủ công để "thấy" cơ chế)

Dùng đúng ví dụ sách của bài gốc. Mỗi sách = `[pages, year, rating]` (tôi bỏ cột genre để đơn giản — xem ghi chú bên dưới). Ta muốn tìm sách gần với query `[200 trang, năm 2018, rating 4.8]`.

Dùng **Euclidean distance** — công thức "khoảng cách chim bay":
`d = √( (p₁−p₂)² + (y₁−y₂)² + (r₁−r₂)² )`

Tính tay cho Book4 `[190, 2018, 4.7]`:
`d = √((200−190)² + (2018−2018)² + (4.8−4.7)²) = √(100 + 0 + 0.01) = √100.01 ≈ 10.0`

Tính cho Book5 `[200, 2012, 4.5]`:
`d = √(0 + 6² + 0.3²) = √(36 + 0.09) ≈ 6.01`

→ Theo con số thô, Book5 "gần" hơn Book4. **Nhưng khoan** — đây là cái bẫy quan trọng nhất của tầng basic: cột `year` chênh 6 đơn vị đã lấn át cột `rating` chênh 0.3, còn cột `pages` chênh 10 lại lấn át tất cả. **Khoảng cách bị chi phối bởi feature có thang đo (scale) lớn nhất**, không phản ánh cái ta thực sự quan tâm. Ta sẽ sửa bằng *normalization* ở tầng Intermediate. Nhớ điểm này.

> 🔍 **Ghi chú trung thực về bài gốc:** ví dụ trong video bị **lỗi dữ liệu**. Nó nói "số đầu tiên là genre: 1=fiction, 2=non-fiction, 3=sci-fi", rồi lại liệt kê 5 cuốn "science fiction" mà cuốn nào cũng bắt đầu bằng số `1`. Một staff engineer *phải* để ý những bất nhất kiểu này — dữ liệu sai âm thầm là nguồn bug tốn kém nhất trong ML. Tôi đã bỏ cột genre để tránh nhân bản lỗi đó.

### 1.5. Code "hello world"

Đoạn này **đã chạy thật** bằng numpy:

```python
import numpy as np

# Mỗi sách là 1 vector: [pages, year, rating]
books = {
    'Book1': np.array([180, 2010, 4.6]),
    'Book2': np.array([220, 2005, 4.8]),
    'Book3': np.array([210, 2015, 4.9]),
    'Book4': np.array([190, 2018, 4.7]),
    'Book5': np.array([200, 2012, 4.5]),
}
query = np.array([200, 2018, 4.8])

def euclidean(a, b):
    # √ tổng bình phương hiệu từng chiều
    return np.sqrt(np.sum((a - b) ** 2))

# Xếp hạng theo khoảng cách tăng dần = giống nhất trước
ranked = sorted(books.items(), key=lambda kv: euclidean(query, kv[1]))
for name, vec in ranked:
    print(name, round(euclidean(query, vec), 2))
```

Output thật:
```
Book5 6.01
Book4 10.0
Book3 10.44
Book1 21.54
Book2 23.85
```

Đây chính là toàn bộ ý tưởng của một vector DB thu nhỏ: **biểu diễn dữ liệu thành vector → định nghĩa distance → trả về k điểm gần nhất**. Mọi thứ phía sau chỉ là làm cho bước "tìm k gần nhất" nhanh ở quy mô tỷ bản ghi.

### 1.6. ✅ Self-check tầng Basic
1. Tại sao câu "tìm sản phẩm *giống* cái này" khó với SQL thường nhưng dễ với vector DB?
2. Trong `[180, 2010, 4.6]`, "dimension" là gì và ở đây có mấy dimension?
3. Trong ví dụ tay, vì sao Book5 lại "gần" hơn Book4 dù rating của Book4 khớp query hơn? (Gợi ý: feature nào đang lấn át?)

---

## Phần 2 — 🟡 INTERMEDIATE (Vận dụng)

### 2.1. `[MỞ RỘNG]` Vector đến từ đâu? — Embeddings

Bài gốc bỏ qua câu hỏi quan trọng nhất: **ai gán những con số đó cho dữ liệu?** Với sách thì bạn tự bịa `[pages, year, rating]`. Nhưng với một câu văn, một tấm ảnh, một đoạn audio thì sao? Câu trả lời là **embedding model**.

- **Embedding:** một mô hình ML (thường là neural network) nhận input phi cấu trúc (text/ảnh/audio) và xuất ra một vector cố định chiều, sao cho **input giống nhau về ngữ nghĩa → vector gần nhau**. Ví dụ: câu "con chó" và "chú cún" cho ra hai vector gần nhau dù không trùng một ký tự nào.
- Ví dụ model thật (2026): `text-embedding-3-small` (OpenAI, 1536 chiều), `voyage-3.5` (1024 chiều), các model open-source như `bge`, `e5`, `nomic-embed`. Ảnh dùng CLIP.

Đây là mắt xích mà 90% người mới bỏ sót: **vector DB không tự sinh ra vector**. Pipeline thật là:
```
dữ liệu thô → [embedding model] → vector → [vector DB lưu + đánh index]
câu query   → [cùng embedding model] → query vector → [vector DB tìm nearest]
```
Quy tắc vàng: **query và data phải đi qua cùng một embedding model**, nếu không hai vector nằm ở hai "hệ tọa độ" khác nhau và kết quả vô nghĩa.

### 2.2. Chọn distance metric: cosine vs dot vs L2

Có 3 cách đo "gần" phổ biến, và chọn sai là lỗi kinh điển:

| Metric | Công thức (ý tưởng) | Đo cái gì | Dùng khi |
|---|---|---|---|
| **L2 (Euclidean)** | √Σ(aᵢ−bᵢ)² | khoảng cách vị trí | features có ý nghĩa vật lý, cùng scale |
| **Cosine similarity** | (a·b)/(‖a‖‖b‖) | **góc** giữa 2 vector (bỏ qua độ dài) | text embeddings (mặc định phổ biến nhất) |
| **Dot product** | a·b | góc + độ lớn | khi độ lớn vector mang thông tin (vd recsys) |

**Vì sao text hay dùng cosine?** Vì với text, *hướng* của vector mang ngữ nghĩa, còn *độ dài* thường chỉ phản ánh độ dài văn bản — thứ ta không muốn nó ảnh hưởng. Hai đoạn cùng chủ đề nhưng một dài một ngắn nên được coi là giống nhau.

**Mẹo quan trọng:** nếu bạn **normalize** vector về độ dài 1 (unit vector), thì cosine, dot, và thứ tự của L2 cho ra **cùng một ranking**. Nhiều hệ thống normalize sẵn rồi dùng dot product vì nó tính nhanh nhất.

### 2.3. Sửa cái bẫy ở tầng Basic bằng normalization

Nhớ vụ cột `year` lấn át `rating` chứ? Giải pháp: đưa mỗi feature về cùng thang (vd z-score hoặc min-max), hoặc với embedding thật thì normalize cả vector:

```python
import numpy as np

def normalize(v):
    # đưa vector về độ dài 1 -> chỉ còn "hướng" quyết định
    return v / np.linalg.norm(v)

def cosine_sim(a, b):
    a, b = normalize(a), normalize(b)
    return np.dot(a, b)   # sau normalize, dot chính là cosine

a = np.array([1.0, 0.0])
b = np.array([1.0, 1.0])
print(round(cosine_sim(a, b), 4))   # 0.7071  (góc 45°)
```

### 2.4. Code thực tế hơn — gần với công việc thật

Đây là pattern bạn gặp trong mọi RAG demo. Code minh họa API thật của `sentence-transformers` + `faiss` (thư viện chuẩn công nghiệp). *(Code cần cài package + tải model nên tôi không chạy trong sandbox, nhưng đây đúng là API production.)*

```python
# pip install sentence-transformers faiss-cpu
from sentence_transformers import SentenceTransformer
import faiss
import numpy as np

# 1) Embedding model: biến text -> vector 384 chiều
model = SentenceTransformer('all-MiniLM-L6-v2')

docs = [
    "How do I reset my password?",
    "The cat sat on the warm windowsill.",
    "Steps to recover a forgotten account login.",
    "Photosynthesis converts sunlight into energy.",
]
emb = model.encode(docs, normalize_embeddings=True)  # normalize -> dùng dot = cosine
emb = np.asarray(emb, dtype='float32')

# 2) Xây index. IndexFlatIP = brute-force, inner product (dot).
#    Với data đã normalize, inner product == cosine similarity.
dim = emb.shape[1]
index = faiss.IndexFlatIP(dim)
index.add(emb)                       # nạp toàn bộ vector vào index

# 3) Query -> cùng model -> tìm 2 hàng xóm gần nhất
q = model.encode(["I can't log in, help"], normalize_embeddings=True).astype('float32')
scores, ids = index.search(q, k=2)   # trả về similarity + chỉ số doc

for score, i in zip(scores[0], ids[0]):
    print(f"{score:.3f}  {docs[i]}")
# Kỳ vọng: 2 câu về password/login xếp trên, dù không trùng từ với query.
```

Điểm cốt lõi: query "I can't log in" khớp với "reset my password" và "recover account login" **dù gần như không chung từ nào** — đó là sức mạnh của semantic search mà keyword search (SQL `LIKE`) không có.

### 2.5. `[MỞ RỘNG]` 3 lỗi thường gặp (staff phải biết)

1. **Dùng embedding model khác nhau cho index và query.** Vector rơi vào hai không gian khác nhau → kết quả rác. Luôn version-lock model; đổi model = phải **re-embed toàn bộ** corpus.
2. **Quên normalize nhưng lại dùng cosine/dot.** Kết quả bị độ dài vector bóp méo. Hoặc ngược lại: normalize rồi vẫn nghĩ mình đang đo L2.
3. **Kỳ vọng semantic search thay thế được exact match.** Vector search *dở* với mã đơn hàng, số phiên bản, tên riêng, error code — những thứ cần khớp chính xác. Đây là lý do tồn tại **hybrid search** (kết hợp keyword BM25 + vector), sẽ nói ở tầng Staff.

### 2.6. ✅ Self-check tầng Intermediate
1. Vẽ pipeline từ "text thô" đến "kết quả similarity search", chỉ rõ embedding model xuất hiện mấy lần và vì sao phải là cùng một model.
2. Khi nào chọn cosine thay vì L2? Sau khi normalize thì cosine, dot, L2 khác nhau thế nào về ranking?
3. Sếp bảo "thay embedding model mới xịn hơn". Việc phải làm với dữ liệu đã lưu là gì?

---

## Phần 3 — 🔴 ADVANCED (Chuyên sâu)

### 3.1. Vì sao brute-force không scale — Big-O

Cách ở tầng Basic (so query với *mọi* vector) gọi là **exact / flat search**. Chi phí mỗi query: **O(N·d)** — N vector, mỗi vector d chiều.

Làm phép tính thật: 100 triệu vector × 768 chiều × (1 phép nhân + 1 phép cộng) ≈ 1.5×10¹¹ phép tính **cho mỗi query**. Ở QPS cao là bất khả thi. Ta cần đổi *độ chính xác tuyệt đối* lấy *tốc độ* → bước sang **ANN (Approximate Nearest Neighbor)**.

> **ANN ≠ một thuật toán.** Nó là *cả một họ* thuật toán chấp nhận bỏ sót một chút hàng xóm thật để đổi lấy tốc độ gấp hàng nghìn lần. HNSW, IVF, IVF-PQ, ScaNN, DiskANN đều là các thành viên.

### 3.2. `[MỞ RỘNG]` Curse of dimensionality — chỗ khó mà người ta hay nhầm

**Nói thẳng: đây là phần phản trực giác và khó.** Ở chiều cao (hàng trăm–nghìn chiều), *mọi điểm gần như cách đều nhau*, khoảng cách giữa hàng xóm gần nhất và xa nhất co lại gần bằng nhau. Hệ quả: nhiều kỹ thuật ANN đơn giản (như LSH ngây thơ) **thất bại trên dữ liệu ngẫu nhiên**.

Tôi đã kiểm chứng điều này: chạy LSH random-hyperplane trên 10k vector Gaussian 128 chiều → **recall@5 = 0.0**. May thay, embedding thật **không** phân bố ngẫu nhiên — chúng **clustered** (gom cụm theo chủ đề). Chính cấu trúc cụm này là thứ ANN khai thác được. Bài học staff: đừng benchmark ANN trên `np.random.randn`; hãy dùng dữ liệu giống production.

### 3.3. Ba họ index cần thuộc

**a) HNSW (Hierarchical Navigable Small World)** — index phổ biến nhất 2026.
- Ý tưởng: xây một **graph nhiều tầng**. Tầng trên cùng thưa (nhảy xa), tầng dưới dày (tinh chỉnh). Query đi từ trên xuống, mỗi tầng "greedy" nhảy đến node gần query hơn — giống *skip list* trong không gian vector.
- Query time ≈ **O(log N)**. Recall cao, latency thấp.
- Nhược: **ngốn RAM** (phải giữ cả graph trong bộ nhớ) và **build/insert chậm** (xây 1–100M vector có thể mất phút đến giờ).
- Tham số: `M` (số cạnh mỗi node — cao hơn = recall tốt hơn, RAM nhiều hơn), `efConstruction` (độ rộng tìm kiếm lúc build), `efSearch` (độ rộng lúc query — **knob chỉnh recall↔latency tại runtime**).

**b) IVF (Inverted File)** — chia để trị.
- Ý tưởng: dùng **k-means** chia không gian thành `nlist` cụm (Voronoi cells). Query chỉ quét `nprobe` cụm gần nhất thay vì toàn bộ.
- Tham số: `nlist` (số cụm, cố định lúc build), `nprobe` (số cụm quét lúc query — **knob recall↔latency**).
- Ưu: build **nhanh** (giây–phút), insert rẻ, hợp workload cần freshness cao. Nhược: recall dễ tụt với query "long-tail" rơi vào ranh giới cụm.

**c) PQ (Product Quantization)** — nén để tiết kiệm RAM.
- Ý tưởng: cắt vector d chiều thành m đoạn con, mỗi đoạn thay bằng ID của centroid gần nhất (codebook). Một vector 1536 chiều × 4 byte (6KB) có thể nén xuống vài chục byte.
- Đánh đổi: mất một phần recall. Thường ghép: **IVF-PQ** (đại trà, tiết kiệm RAM cực mạnh) hoặc **HNSW-PQ**.
- Họ hàng mới: **RaBitQ** (nhị phân ngẫu nhiên, đạt recall ~HNSW ở memory ~IVFFlat), **DiskANN/Vamana** (graph để trên SSD cho tỷ-scale không vừa RAM), **ScaNN** (Google, tối ưu inner-product).

### 3.4. Bảng so sánh index

| Index | Query Big-O | Recall | RAM | Build/Insert | Chọn khi |
|---|---|---|---|---|---|
| **Flat (exact)** | O(N·d) | 100% | vừa | tức thời | corpus nhỏ (<50k), cần ground-truth |
| **HNSW** | ~O(log N) | rất cao | **cao** | chậm | mặc định cho <10M, cần latency thấp + data động |
| **IVF-Flat** | O(nprobe·N/nlist·d) | cao (tùy nprobe) | vừa | nhanh | data lớn, tĩnh, cần build nhanh / freshness |
| **IVF-PQ** | rẻ | trung bình | **rất thấp** | nhanh | tỷ-scale, RAM là bottleneck |
| **DiskANN** | log-ish | cao | thấp (SSD) | chậm | tỷ-scale không vừa RAM |

**Rule of thumb 2026:** *bắt đầu bằng HNSW* cho hầu hết RAG dưới ~10M vector; chỉ chuyển sang IVF/IVF-PQ khi scale hoặc chi phí RAM ép buộc.

### 3.5. Đo chất lượng: recall@k và "tam giác đánh đổi"

- **recall@k** = (số hàng xóm thật trong top-k mà ANN tìm được) / k. Đây là thước đo *chất lượng* của ANN. Latency vô nghĩa nếu kết quả sai.
- **Tam giác bất khả thi:** *Recall ↔ Latency ↔ Memory* — kéo góc này lên thường phải hạ góc kia. Mọi tuning (efSearch, nprobe, mức nén PQ) là di chuyển trong tam giác này.
- **Two-stage search (rất hay dùng):** giai đoạn 1 ANN thô lấy dư candidate (vd top-100) → giai đoạn 2 tính lại distance **full-precision** (hoặc dùng một reranker/cross-encoder) rồi cắt xuống top-10. Cho recall gần exact với chi phí thấp.

### 3.6. Code nâng cao — tự implement IVF từ đầu (đã chạy thật)

Đoạn này minh họa chính xác cơ chế IVF và **knob `nprobe`**. Đã chạy trên 50.000 vector 64 chiều **có cấu trúc cụm** (giống embedding thật):

```python
import numpy as np
from collections import defaultdict
np.random.seed(42)

# Dữ liệu CLUSTERED (giống embedding thật), KHÔNG phải random đều
N, d, n_clusters = 50000, 64, 50
centers = np.random.randn(n_clusters, d) * 8
data = (centers[np.random.randint(0, n_clusters, N)]
        + np.random.randn(N, d)).astype('float32')
q = (centers[7] + np.random.randn(d) * 0.5).astype('float32')

# Ground truth: exact top-10 (để đo recall)
truth = set(np.argsort(np.linalg.norm(data - q, axis=1))[:10].tolist())

# --- TRAIN: k-means thô để có nlist centroid (coarse quantizer) ---
nlist = 100
km = data[np.random.choice(N, nlist, replace=False)].copy()
assign = np.empty(N, dtype=int)
for _ in range(8):                                   # vài vòng k-means
    for i in range(0, N, 5000):                      # gán cụm theo batch
        chunk = data[i:i+5000]
        assign[i:i+5000] = np.argmin(
            np.linalg.norm(chunk[:, None, :] - km[None, :, :], axis=2), axis=1)
    for c in range(nlist):                            # cập nhật centroid
        m = assign == c
        if m.any():
            km[c] = data[m].mean(0)

# --- INVERTED FILE: cụm -> danh sách vector thuộc cụm ---
inv = {c: np.where(assign == c)[0] for c in range(nlist)}

def ivf_search(q, nprobe, k=10):
    probe = np.argsort(np.linalg.norm(km - q, axis=1))[:nprobe]  # nprobe cụm gần nhất
    cand = np.concatenate([inv[c] for c in probe]).astype(int)   # gom ứng viên
    d2 = np.linalg.norm(data[cand] - q, axis=1)                  # rerank exact
    top = cand[np.argsort(d2)[:k]]
    return set(top.tolist()), cand.size

for nprobe in [1, 5, 20, 50, 100]:
    res, scanned = ivf_search(q, nprobe)
    print(f"nprobe={nprobe:3d}  scanned={scanned:5d}/{N} "
          f"({100*scanned/N:4.1f}%)  recall@10={len(res & truth)/10:.2f}")
```

Output **thật**:
```
nprobe=  1  scanned=  392/50000 ( 0.8%)  recall@10=0.70
nprobe=  5  scanned= 2951/50000 ( 5.9%)  recall@10=1.00
nprobe= 20  scanned= 9920/50000 (19.8%)  recall@10=1.00
nprobe= 50  scanned=25275/50000 (50.5%)  recall@10=1.00
nprobe=100  scanned=50000/50000 (100.0%) recall@10=1.00
```

**Đọc kết quả này như một staff engineer:**
- `nprobe=1`: quét **0.8%** dữ liệu, recall 0.70 — nhanh nhưng bỏ sót.
- `nprobe=5`: quét **5.9%**, recall **1.00** — **sweet spot**: gần như đúng hoàn toàn với 1/17 công sức.
- `nprobe=100`: quét 100% = quay về brute-force exact.

Đây chính là toàn bộ triết lý ANN gói trong một bảng: **chỉnh một knob để trượt trên đường cong recall↔cost**. `efSearch` của HNSW đóng vai trò y hệt `nprobe`.

### 3.7. `[MỞ RỘNG]` Edge cases
- **Filtered search** ("tìm giống + `category='shoes'`"): filter *trước* hay *sau* ANN? Pre-filter dễ làm rỗng cụm → recall tụt thảm; post-filter tốn công. Các engine tốt (Qdrant) nhúng filter *trong lúc* traversal.
- **Vector trùng lặp / near-duplicate** làm loãng kết quả → cần dedup (đây là lý do các pipeline chèn bước data cleaning/dedup trước embedding).
- **Insert/delete động:** HNSW không thích xóa (để lại "tombstone"), thường phải rebuild định kỳ. IVF chịu insert tốt hơn.
- **Empty/short query, out-of-distribution query:** trả về top-k rác với score thấp — cần đặt **threshold** để nói "không có kết quả đủ tốt".

---

## Phần 4 — 🟣 STAFF LEVEL (Tư duy hệ thống & lãnh đạo kỹ thuật)

### 4.1. Thiết kế ở quy mô lớn — bắt đầu bằng phép tính bộ nhớ

Câu hỏi staff đầu tiên luôn là: *"Nó ngốn bao nhiêu RAM?"* Làm toán:

> **1 tỷ vector × 1536 chiều × 4 byte (float32) = ~6.1 TB** chỉ riêng vector thô, chưa tính graph HNSW (thêm ~1.5–2×).

Không server đơn nào giữ nổi trong RAM. Ba đòn bẩy để sống sót:
1. **Quantization:** float32 → int8 (÷4) hoặc PQ (÷10–÷40). 6TB có thể xuống vài trăm GB. Đánh đổi: recall giảm — phải đo, không đoán.
2. **Sharding (scale ngang):** chia corpus theo N shard, mỗi shard 1 máy, query fan-out song song rồi merge top-k. Đây là "distributed computing + parallel processing" mà bài gốc nhắc tới (nhưng không giải thích).
3. **Disk-based index (DiskANN):** giữ graph trên SSD, chỉ nạp phần nóng vào RAM.

**Bottleneck ở đâu?** Thường **KHÔNG** phải ở DB. Đo thực tế production: một query RAG ~300ms thì DB chỉ tốn 5–8ms, còn **~90ms là sinh embedding cho query** và phần lớn còn lại là LLM inference. Bài học staff: *đừng tối ưu tầng không phải bottleneck*. Cache embedding của query hay lặp lại, gộp batch, hoặc dùng model embedding nhỏ hơn thường lời hơn là đổi vector DB.

### 4.2. Trade-off tầm kiến trúc: khi nào NÊN / KHÔNG NÊN

**KHÔNG cần vector DB riêng khi:**
- Corpus < ~10M vector và bạn *đã* chạy Postgres → **pgvector** là default hợp lý (0.5.0+ có HNSW, ngang ngửa DB chuyên dụng ở scale 1M). Vector nằm cùng bảng dữ liệu ứng dụng, transaction ACID, không thêm hệ thống, không sync layer. Ước tính 2026: **pgvector là lựa chọn đúng cho ~70% workload** AI-agent/RAG.
- Vấn đề thật ra là *exact match* (mã, ID, tên riêng) → dùng full-text/BM25, đừng ép vector.

**NÊN dùng DB chuyên dụng khi:** scale vượt 50–100M, cần filtered search cực nhanh, cần hybrid search native, hoặc cần tách rời scale của read/write/storage.

**Landscape 2026 — bảng chọn nhanh** (đã cập nhật từ nhiều benchmark 2026):

| Công cụ | Kiểu | Điểm mạnh | Chọn khi |
|---|---|---|---|
| **pgvector** | Postgres extension | đơn giản, ACID, không thêm hệ thống | đã có Postgres, <10–50M vector |
| **Pinecone** | Managed | zero-ops, scale tự động | muốn không lo hạ tầng, chấp nhận cost cao & ít quyền tune recall |
| **Qdrant** | OSS (Rust) | filtered search nhanh nhất, latency thấp ổn định | self-host, filter nặng, cần tốc độ |
| **Weaviate** | OSS + cloud | hybrid search & multi-modal native | cần keyword+vector trong một query |
| **Milvus** | OSS (Zilliz) | tỷ-scale, sharding trưởng thành | >100M–tỷ vector, có team data-eng |

**Sự thật khó chịu (staff-level):** *"phần lớn thất bại của vector DB là tự gây ra"* — chất lượng chunking và embedding quan trọng hơn việc chọn DB nào. Chọn DB thường là *tie-breaker* sau cam kết hạ tầng sẵn có, không phải quyết định số một.

### 4.3. Chi phí, độ tin cậy, vận hành

- **Cost:** Pinecone tiện nhưng ở enterprise-scale có thể đắt gấp **5–10×** so với self-host Qdrant/Milvus. Với ngành bị quản lý (data sovereignty), managed cloud có thể *bị loại thẳng*. Đòn bẩy giảm cost lớn nhất: giảm chiều embedding, quantization, và **giảm số vector** (dedup/chunk tốt) — thường "tự trả tiền cho chính nó" qua hóa đơn nhỏ đi.
- **Latency:** đặt SLO theo **p99**, không phải trung bình. Nhớ tách riêng thời gian embedding vs DB vs LLM khi đo.
- **Reliability & failure modes:** index rebuild làm tăng RAM đột biến (build HNSW mới song song rồi drop cũ); node chết giữa fan-out → cần replica; embedding model đổi version âm thầm → **recall drift**.
- **Monitoring:** giám sát **recall@k trên một golden set cố định** theo thời gian (không chỉ latency), QPS, tỉ lệ query "no good result", memory, index build time. Recall là thứ tụt âm thầm — không có metric này thì bạn mù.

### 4.4. `[MỞ RỘNG]` Hybrid search — thứ production thực sự dùng

Pure vector search *dở* với proper noun, số hiệu, error code. Production nghiêm túc dùng **hybrid**: chạy song song **BM25 (keyword)** + **vector**, rồi hợp nhất điểm (vd Reciprocal Rank Fusion), thường thêm một **reranker** (cross-encoder) ở cuối. Câu "recall ANN vừa phải cũng chấp nhận được nếu tập bằng chứng cuối được rerank mạnh" là tư duy đúng. Weaviate/Qdrant/Vespa hỗ trợ hybrid native; pgvector/Pinecone cần ghép tay.

### 4.5. Ảnh hưởng tổ chức

- **Giải thích cho non-technical stakeholder:** *"Database thường tìm bằng từ khóa khớp chính xác. Cái này tìm bằng **ý nghĩa** — hỏi 'làm sao đăng nhập' vẫn ra bài 'khôi phục mật khẩu' dù không trùng chữ. Đó là thứ khiến chatbot/tìm kiếm của ta 'hiểu' người dùng."*
- **Ảnh hưởng roadmap:** quyết định embedding model là quyết định *khóa cứng* — đổi model = re-embed toàn bộ + downtime index + có thể tụt/tăng chất lượng. Cần versioning và A/B trên golden set trước khi rollout. Đây là loại quyết định staff phải làm chậm và cẩn thận, vì nó chạm mọi downstream team.
- **Build vs buy** là quyết định người + tiền, không chỉ kỹ thuật: team 3–5 người thì managed (Pinecone/pgvector) ít ma sát nhất; chọn Milvus mà không có người vận hành là tự bắn chân.

### 4.6. 🎤 Câu hỏi system design mẫu + hướng trả lời staff

> **"Thiết kế semantic search cho 500 triệu tài liệu, p99 < 200ms, có filter theo tenant."**

Khung trả lời của một staff engineer:
1. **Làm rõ requirement trước khi vẽ:** QPS đỉnh? Read-heavy hay write-heavy? Freshness (index ngay hay batch)? Recall mục tiêu? Budget? Filter cardinality (bao nhiêu tenant)?
2. **Ước lượng:** 500M × 1024 chiều × 4B ≈ 2TB float32 → **phải quantize + shard**. Chọn int8 hoặc PQ để vừa RAM cụm; shard theo hash để phân tải đều.
3. **Pipeline:** ingest → chunk → embed (batch, GPU) → ghi vào N shard. Query → embed (cache LRU cho query nóng) → fan-out tới shard → merge top-k → **rerank** → filter tenant *trong* traversal (payload-indexed filter kiểu Qdrant) để tránh recall sụp.
4. **Index:** HNSW cho latency (efSearch chỉnh theo p99), hoặc IVF-PQ nếu RAM là ràng buộc cứng; two-stage (ANN thô → rerank exact) để giữ recall.
5. **Chọn tool + biện minh:** ở 500M + multi-tenant + self-host → Milvus/Qdrant; nếu chấp nhận cost & managed → Pinecone. Nêu rõ trade-off, không tuyên bố "cái này tốt nhất".
6. **Vận hành:** replica cho HA, giám sát recall@k trên golden set, blue-green cho index rebuild, SLO theo p99, cost guardrail.
7. **Chốt bằng bottleneck insight:** *"Ở scale này, DB hiếm khi là nút thắt — embedding generation và rerank mới là. Tôi sẽ đo end-to-end và tối ưu tầng đắt nhất trước."* → câu này phân biệt staff với senior.

---

## Phần 5 — 🎯 CHỐT LẠI ĐỂ ĐI PHỎNG VẤN (Interview Cheatsheet)

### 5.1. Keywords bắt buộc nhớ (EN + định nghĩa 1 dòng)
- **Vector / Embedding:** mảng số biểu diễn dữ liệu; input giống nghĩa → vector gần nhau.
- **Dimension:** số phần tử của vector (embedding thật ~384–3072).
- **Similarity search / Nearest Neighbor (NN):** tìm vector gần query nhất.
- **ANN (Approximate NN):** họ thuật toán đổi chút recall lấy tốc độ lớn.
- **Distance metric:** cosine (góc), dot product (góc+độ lớn), L2 (khoảng cách).
- **Normalize:** đưa vector về độ dài 1; khi đó cosine ≡ dot ≡ ranking của L2.
- **HNSW:** graph nhiều tầng, query ~O(log N), recall cao, ngốn RAM. Knobs: M, efConstruction, efSearch.
- **IVF:** chia cụm k-means, quét nprobe cụm. Knobs: nlist, nprobe.
- **PQ (Product Quantization):** nén vector để giảm RAM, mất chút recall. IVF-PQ, HNSW-PQ.
- **recall@k:** % hàng xóm thật lọt vào top-k trả về = chất lượng ANN.
- **Curse of dimensionality:** ở chiều cao mọi điểm gần như cách đều → ANN dựa vào cấu trúc cụm của embedding thật.
- **Hybrid search:** BM25 (keyword) + vector + rerank.
- **RAG:** dùng vector search lấy context liên quan rồi đưa cho LLM sinh câu trả lời.

### 5.2. Core concepts (nếu chỉ nhớ vài điều thì nhớ cái này)
1. Vector DB trả lời "cái gì *giống* nhất", SQL trả lời "cái gì *bằng đúng*".
2. Vector không tự có — đến từ **embedding model**; query & data phải **cùng model**.
3. Text embedding → mặc định **normalize + cosine/dot**.
4. Exact search O(N·d) không scale → dùng **ANN**, và ANN đổi **recall** lấy **latency/memory**.
5. **HNSW là default 2026** cho <10M; IVF/IVF-PQ khi scale/RAM ép buộc.
6. Mọi tuning là trượt trên **tam giác Recall ↔ Latency ↔ Memory**.
7. Bottleneck thường ở **embedding + LLM**, không phải DB.
8. **pgvector là default cho ~70% use case**; DB chuyên dụng chỉ khi scale/filter/hybrid ép buộc.
9. Đo **recall@k trên golden set** theo thời gian, không chỉ latency.
10. Chất lượng **chunking + embedding** quan trọng hơn chọn DB nào.

### 5.3. Ideas / mental models (để trả lời trôi chảy, gây ấn tượng)
- **"Skip list trong không gian vector"** → HNSW.
- **"Chia để trị bằng k-means"** → IVF; `nprobe` là "quét thêm bao nhiêu cụm".
- **"Tam giác bất khả thi Recall–Latency–Memory"** → mọi trade-off nằm ở đây.
- **"DB hiếm khi là bottleneck"** → luôn hỏi lại đo end-to-end.
- **"Đổi embedding model = re-embed cả kho"** → quyết định khóa cứng, làm chậm & cẩn thận.
- **"Vector search mù với ID/proper noun"** → nên hybrid.

### 5.4. Code cần thuộc lòng
**(a) Cosine similarity + top-k bằng numpy** — interviewer hay bắt viết tại chỗ:
```python
import numpy as np
def cosine(a, b):
    return a @ b / (np.linalg.norm(a) * np.linalg.norm(b))
def topk(query, data, k=5):                 # data: (N, d)
    dn = data / np.linalg.norm(data, axis=1, keepdims=True)
    qn = query / np.linalg.norm(query)
    sims = dn @ qn                          # (N,) cosine cho mọi vector
    return np.argsort(-sims)[:k]            # k index điểm cao nhất
```
**(b) faiss tối thiểu** — chứng tỏ biết công cụ chuẩn:
```python
import faiss, numpy as np
index = faiss.IndexFlatIP(dim)   # normalize sẵn -> IP = cosine
index.add(embeddings)            # (N, dim) float32
D, I = index.search(query, k)    # I = id kết quả, D = score
```
Khi nào dùng: `IndexFlatIP` cho ground-truth/corpus nhỏ; đổi sang `IndexHNSWFlat` hoặc `IndexIVFFlat` khi N lớn.

### 5.5. Câu hỏi phỏng vấn thường gặp + gợi ý trả lời
1. **"Vector DB khác SQL DB ở đâu?"** → similarity/ANN vs exact/range; lưu high-dim vector + index chuyên biệt; đừng quên nói vector đến từ embedding.
2. **"HNSW vs IVF, chọn cái nào?"** *(trap)* → không có "tốt nhất": HNSW recall cao/latency thấp nhưng ngốn RAM & build chậm; IVF build nhanh/insert rẻ hợp freshness nhưng recall dễ tụt. Mặc định HNSW cho <10M, cân nhắc IVF-PQ khi RAM là ràng buộc.
3. **"Làm sao giảm latency mà không mất nhiều recall?"** *(trade-off)* → giảm efSearch/nprobe có kiểm soát + đo recall@k; two-stage (ANN thô → rerank exact); quantization; và **kiểm tra bottleneck có thật ở DB không**.
4. **"Cosine hay Euclidean cho text?"** → cosine (hoặc dot sau normalize), vì hướng mang nghĩa còn độ dài thường chỉ là độ dài văn bản.
5. **"1 tỷ vector 1536 chiều, thiết kế thế nào?"** → làm toán ~6TB → quantize + shard + fan-out; nêu index (IVF-PQ/DiskANN); giám sát recall; chốt bằng bottleneck.
6. **"Pure vector search có đủ không?"** *(trap)* → không, nó mù với ID/số/tên riêng → hybrid (BM25+vector+rerank).
7. **"Khi nào KHÔNG dùng vector DB?"** → khi <10M và đã có Postgres (pgvector); khi bài toán thật là exact match; khi chưa đo được vector search có cải thiện gì.

### 5.6. Câu trả lời "one-liner" đắt giá
- *"A vector DB answers 'what's similar', not 'what's equal' — and 'similar' is defined by an embedding model, not the database."*
- *"HNSW is a skip list in vector space; IVF is divide-and-conquer with k-means. Both just give you one knob to slide along the recall–latency curve."*
- *"Recall, latency, memory — pick two. Every tuning parameter is a move inside that triangle."*
- *"At scale the vector DB is rarely the bottleneck; embedding generation and the LLM are. I'd measure end-to-end before optimizing anything."*
- *"Most vector search failures are self-inflicted — chunking and embedding quality matter more than which database you picked."*
- *"Changing the embedding model means re-embedding the whole corpus. That's a locked-in decision I make slowly."*

---

*Hết. Chúc bạn phỏng vấn tốt — nếu chỉ kịp ôn 5 phút, đọc lại Mục 5.2 và 5.6.*
