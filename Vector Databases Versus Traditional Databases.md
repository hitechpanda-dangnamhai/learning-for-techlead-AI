# Giáo trình: Vector Databases vs Traditional Databases — từ Zero đến Staff Engineer

> Tái cấu trúc từ bài đọc **"Vector Databases Versus Traditional Databases"** (Richa Arora — IBM Skills Network). Giảng lại + đào sâu + mở rộng theo tư duy staff-level.
> Thuật ngữ kỹ thuật giữ nguyên tiếng Anh. Code đã chạy thật (Python stdlib `sqlite3` + numpy). Thông tin thị trường cập nhật tới 2026.
>
> Bài đọc gốc là bài **so sánh** (relational DB vs vector DB, và vector *library* vs vector *database*). Vì vậy giáo trình này xoáy vào **trục ra quyết định kiến trúc**, không lặp lại phần nội tại HNSW/IVF đã có ở giáo trình "Essential vector database concepts" trước đó (chỉ tham chiếu khi cần).
>
> `[MỞ RỘNG]` = nội dung ngoài bài gốc mà một staff engineer buộc phải biết.

---

## Phần 0 — 🗺️ Bản đồ bài học (Overview)

**Bài này dạy gì.** Không phải "vector DB là gì" (đã học), mà là câu hỏi đắt giá hơn: **"Khi nào dùng relational database, khi nào dùng vector database, và khi nào dùng cả hai?"** Bài đọc gốc đặt hai loại DB cạnh nhau và so trên 5 trục: *data representation, search & retrieval, indexing, scalability, applications*. Nó cũng phân biệt một cặp khái niệm hay bị nhầm: **vector library** (như FAISS) vs **vector database** (như Pinecone/Milvus).

**Sau khi học xong bạn sẽ làm được:**
- Giải thích relational DB lưu và truy vấn dữ liệu thế nào (tables, rows, columns, keys, SQL, ACID) và vì sao nó mạnh ở *structured data*.
- Chỉ ra chính xác **ranh giới**: câu hỏi nào SQL trả lời được, câu hỏi nào chỉ similarity search trả lời được.
- Phân biệt vector *library* vs vector *database* — và biết khi nào một library là đủ, khi nào cần cả một database.
- Đọc và phản biện bảng so sánh 5-trục (kể cả chỗ bài gốc nói hơi cũ).
- Tư duy staff: hầu hết hệ thống thật **dùng cả hai** (polyglot persistence), và biết thiết kế/bảo vệ lựa chọn đó.

**Mạch kiến thức basic → staff:**
- 🟢 **Basic:** Hai câu hỏi khác nhau ("bằng đúng cái gì" vs "giống cái gì") → hai loại DB. Chạy tay cả hai.
- 🟡 **Intermediate:** Mỗi loại *lưu* dữ liệu ra sao (B-tree vs vector index) + phân biệt vector library vs database (CRUD).
- 🔴 **Advanced:** Nội tại — B-tree O(log n) vs curse of dimensionality; ACID vs consistency của vector store; mổ xẻ bảng so sánh 5 trục.
- 🟣 **Staff:** Không phải "hoặc/hoặc" mà "cả hai" — pgvector hợp nhất, polyglot persistence, cost/ops, landscape 2026, system design.
- 🎯 **Cheatsheet:** Keywords, core concepts, mental models, code thuộc lòng, Q&A phỏng vấn.

---

## Phần 1 — 🟢 BASIC (Nền tảng)

### 1.1. Vấn đề thực tế: hai câu hỏi bản chất khác nhau

Bạn có một kho 1 triệu cuốn sách. Có hai loại câu hỏi:

- **Câu hỏi A — "bằng đúng / trong khoảng":** *"Cho tôi sách xuất bản sau 1950, rating ≥ 4.6."* Đây là câu hỏi **exact match / range**. Có đáp án đúng-sai rạch ròi.
- **Câu hỏi B — "giống cái này":** *"Cho tôi sách *giống* Dune."* Đây là câu hỏi **similarity**. Không có "đúng tuyệt đối", chỉ có "gần/xa về ý nghĩa".

**Relational database** sinh ra để trả lời loại A cực tốt. **Vector database** sinh ra để trả lời loại B. Đó là toàn bộ tinh thần của bài đọc: hai công cụ cho hai loại câu hỏi, không phải cái này thay cái kia.

### 1.2. Analogy đời thường

Hình dung một **thủ thư**:
- **Relational DB = thủ thư giỏi tra sổ.** Bạn hỏi "sách mã ISBN X", "sách mượn sau ngày Y" — trả lời tức thì, chính xác tuyệt đối, vì sổ được sắp theo đúng những trường đó.
- **Vector DB = thủ thư đọc rộng, cảm nhận nội dung.** Bạn đưa cuốn Dune và nói "kiếm cho tôi vài cuốn *na ná*" — họ không tra sổ, họ dựa vào *cảm giác về nội dung* để với tay lên kệ lấy Foundation, Neuromancer.

Người thứ nhất vô dụng với câu hỏi B; người thứ hai lãng phí (và kém chính xác) với câu hỏi A. Hệ thống thật cần **cả hai người**.

### 1.3. Định nghĩa đơn giản (giải thích mọi thuật ngữ)

**Bên relational (bài gốc mô tả kỹ phần này):**
- **Relational database:** DB tổ chức dữ liệu thành **tables** (bảng), gồm **rows** (hàng = một record/bản ghi) và **columns** (cột = một property/attribute).
- **Key:** cột dùng để định danh và nối bảng. **Primary key** (khóa chính) định danh duy nhất một row; **foreign key** (khóa ngoại) trỏ sang primary key của bảng khác để tạo **relationship** (quan hệ).
- **SQL (Structured Query Language):** ngôn ngữ truy vấn/thao tác, với 4 lệnh lõi **SELECT / INSERT / UPDATE / DELETE**.
- Relational DB mạnh khi **structured data** và quan hệ giữa các entity được định nghĩa rõ.

**Bên vector:**
- **Vector:** mảng số nhiều chiều biểu diễn một data item; mỗi **dimension** ứng với một attribute/feature.
- **Similarity search / nearest neighbor query:** cho một vector, tìm các vector *gần nhất*.
- **Embedding / transformer:** như bài gốc minh họa — ảnh/text/audio đi qua *transformer* tương ứng (Image Transformer, NLP Transformer, Audio Transformer) để biến thành vector rồi mới lưu vào vector DB.

### 1.4. Ví dụ chạy tay (thấy sự khác biệt bằng mắt)

Cùng một cuốn sách, hai cách biểu diễn:

| title | pages | year | rating | vector `[sci-fi, romance, epic-scale]` |
|---|---|---|---|---|
| Dune | 412 | 1965 | 4.8 | `[0.90, 0.10, 0.80]` |
| Foundation | 255 | 1951 | 4.6 | `[0.88, 0.10, 0.90]` |
| Neuromancer | 271 | 1984 | 4.5 | `[0.85, 0.05, 0.60]` |
| Pride & Prejudice | 432 | 1813 | 4.7 | `[0.00, 0.95, 0.20]` |
| Emma | 474 | 1815 | 4.4 | `[0.05, 0.90, 0.25]` |

**Câu hỏi A (SQL):** "year > 1950 AND rating ≥ 4.6" → quét từng row, kiểm tra điều kiện: Dune (✓1965, ✓4.8), Foundation (✓1951, ✓4.6), Neuromancer (✓nhưng 4.5 ✗), hai cuốn Austen (✗year). Kết quả: **Dune, Foundation**. Đúng-sai tuyệt đối.

**Câu hỏi B (vector):** "giống Dune" → tính cosine similarity giữa vector Dune `[0.9,0.1,0.8]` và từng cuốn. Foundation `[0.88,0.1,0.9]` gần như trùng hướng → similarity ~1.0. Emma `[0.05,0.9,0.25]` lệch hẳn hướng → similarity thấp. Kết quả xếp hạng: **Foundation > Neuromancer > … > Emma**. Không có "đúng/sai", chỉ có "gần/xa".

Điểm mấu chốt để "thấy": vector search **gom nhóm 3 cuốn sci-fi lại với nhau bất kể năm hay rating** — thứ mà `WHERE genre='sci-fi'` chỉ làm được *nếu* bạn đã có sẵn cột genre gán tay. Vector nắm được "chất sci-fi" mà không cần ai gán nhãn thủ công.

### 1.5. Code "hello world" — cùng data, hai loại query (đã chạy thật)

Dùng `sqlite3` (có sẵn trong Python) cho relational, numpy cho vector:

```python
import sqlite3, numpy as np

rows = [
    ('Dune',        412, 1965, 4.8, [0.90,0.10,0.80]),
    ('Neuromancer', 271, 1984, 4.5, [0.85,0.05,0.60]),
    ('Pride&Prej',  432, 1813, 4.7, [0.00,0.95,0.20]),
    ('Foundation',  255, 1951, 4.6, [0.88,0.10,0.90]),
    ('Emma',        474, 1815, 4.4, [0.05,0.90,0.25]),
]

# ---------- RELATIONAL: exact/range bằng SQL ----------
con = sqlite3.connect(':memory:')
con.execute('CREATE TABLE books(title TEXT, pages INT, year INT, rating REAL)')
con.executemany('INSERT INTO books VALUES (?,?,?,?)',
                [(t,p,y,r) for t,p,y,r,_ in rows])
print('SQL  -> year>1950 AND rating>=4.6:')
for row in con.execute(
    'SELECT title,year,rating FROM books '
    'WHERE year>1950 AND rating>=4.6 ORDER BY rating DESC'):
    print('   ', row)

# ---------- VECTOR: similarity search ----------
titles = [t for t,*_ in rows]
vecs = np.array([v for *_,v in rows], float)
vecs /= np.linalg.norm(vecs, axis=1, keepdims=True)   # normalize -> dùng cosine
q = vecs[0]                                            # query = Dune
sims = vecs @ q
print('VEC  -> giong "Dune":')
for i in np.argsort(-sims):
    print(f'    {titles[i]:12s} sim={sims[i]:.3f}')
```

Output **thật**:
```
SQL  -> year>1950 AND rating>=4.6:
    ('Dune', 1965, 4.8)
    ('Foundation', 1951, 4.6)
VEC  -> giong "Dune":
    Dune         sim=1.000
    Foundation   sim=0.998
    Neuromancer  sim=0.993
    Emma         sim=0.296
    Pride&Prej   sim=0.217
```

Cùng một dữ liệu, hai câu hỏi, hai cơ chế hoàn toàn khác — đây là câu chuyện trung tâm của cả bài đọc.

### 1.6. ✅ Self-check tầng Basic
1. Cho một câu hỏi bất kỳ, làm sao biết nó thuộc loại A (SQL) hay loại B (similarity)?
2. Trong relational DB: row, column, primary key, foreign key mỗi thứ là gì?
3. Vì sao vector search gom được 3 cuốn sci-fi lại mà không cần cột `genre` gán tay?

---

## Phần 2 — 🟡 INTERMEDIATE (Vận dụng)

### 2.1. Mỗi loại *lưu* dữ liệu thế nào (đây là gốc rễ mọi khác biệt)

**Relational — lưu theo hàng, đánh index bằng B-tree.**
Dữ liệu nằm trong bảng; để tra nhanh cột `year`, DB dựng một **B-tree index** trên cột đó. B-tree là cây cân bằng, giữ dữ liệu *có thứ tự*, cho phép tìm exact và range trong **O(log n)**. Đây là lý do `WHERE year>1950` nhanh: DB nhảy tới nhánh cây thay vì quét cả triệu row.

**Vector — lưu vector, đánh index bằng cấu trúc cho không gian nhiều chiều.**
Bài gốc nói vector DB dùng "metric trees and hashing". *(Xem cảnh báo cập nhật ở Phần 3 — 2026 chủ yếu dùng HNSW/IVF, không phải metric trees.)* Ý chung: B-tree vô dụng cho câu hỏi "gần nhất" trong 768 chiều, nên cần loại index khác hẳn (graph, cluster, hash) để làm **ANN (Approximate Nearest Neighbor)**.

> Một dòng nhớ: **relational sắp dữ liệu theo *giá trị có thứ tự*; vector sắp theo *độ gần trong không gian*.** Khác nhau từ tầng lưu trữ, nên khác nhau ở mọi thứ phía trên.

### 2.2. `[MỞ RỘNG]` Vector library vs Vector database — cặp khái niệm bài gốc chạm tới nhưng chưa mổ kỹ

Bài đọc nói: vector library "chủ yếu in-memory", còn vector database "có full CRUD và thường nằm trong production deployment". Đây là một phân biệt **cực hay bị hỏi trong phỏng vấn**. Mổ rõ:

- **Vector library** (FAISS, Annoy, HNSWlib, ScaNN): chỉ là **thuật toán ANN trần**. <cite index="31-1">FAISS cho bạn thuật toán; một vector database cho bạn thuật toán *cộng thêm* persistence, filtering, replication, access control, và một API. Nếu bạn cần thao tác CRUD, bạn cần một database chứ không phải một library.</cite> Library thường <cite index="33-1">là công cụ in-memory, tĩnh, tập trung vào ANN trên một không gian vector cố định — hiệu quả cho similarity search nhưng không quản lý được dữ liệu theo thời gian.</cite>
- **Điểm đau kỹ thuật:** <cite index="32-1">thêm document mới vào index có sẵn của các library như FAISS hay Annoy rất khó vì dữ liệu đã index là bất biến (immutable); HNSWlib là ngoại lệ vì có CRUD và hỗ trợ đọc-ghi đồng thời, nhưng vẫn thiếu deployment ecosystem, replication và fault tolerance.</cite>
- **Vector database** (Pinecone, Qdrant, Weaviate, Milvus): <cite index="33-1">cung cấp full CRUD, metadata filtering, và distributed persistence cho dataset lớn.</cite> Đây là thứ dùng được trong production.

**4 "trại" bạn nên nhớ** (khung phân loại chuẩn 2026): <cite index="31-1">managed (Pinecone) — bạn gửi vector, query vector, người khác lo phần còn lại; open-source purpose-built (Qdrant, Weaviate, Milvus, Chroma) — self-host hoặc dùng managed cloud của hãng; database extensions (pgvector, Elasticsearch) — thêm vector search vào DB bạn đã chạy; và libraries (FAISS, Annoy) — vector search trần không có tầng database.</cite>

> Analogy: **library = động cơ; database = cả chiếc xe** (động cơ + khung + tay lái + phanh + bảo hành). Prototyping thì mượn động cơ chạy thử được; chở khách thương mại thì phải có cả xe.

### 2.3. Code thực tế hơn — dùng đúng "công cụ đúng việc"

Trong hệ thống thật, bạn hay viết **cùng lúc** cả SQL lẫn vector query. Ví dụ pattern "lọc bằng SQL rồi similarity trong tập đã lọc" — nền tảng của filtered vector search:

```python
# Ý tưởng: metadata filter (relational-style) + similarity (vector-style)
# pgvector cho phép làm cả hai TRONG MỘT câu SQL:
#
#   SELECT title
#   FROM books
#   WHERE year > 1950                      -- filter kiểu relational (B-tree)
#   ORDER BY embedding <=> '[0.9,0.1,0.8]' -- '<=>' = cosine distance (vector index)
#   LIMIT 3;
#
# <=> là toán tử distance của pgvector. Một câu, hai thế giới.

# Minh họa bằng numpy cho dễ chạy: lọc trước, rồi rank vector trong tập còn lại
import numpy as np
titles = ['Dune','Neuromancer','Pride&Prej','Foundation','Emma']
years  = np.array([1965,1984,1813,1951,1815])
vecs   = np.array([[0.9,0.1,0.8],[0.85,0.05,0.6],[0.0,0.95,0.2],
                   [0.88,0.1,0.9],[0.05,0.9,0.25]], float)
vecs  /= np.linalg.norm(vecs, axis=1, keepdims=True)

mask = years > 1950                    # bước 1: filter (relational)
q = vecs[0]
sims = np.where(mask, vecs @ q, -1)    # bước 2: chỉ rank cái đã lọt filter
for i in np.argsort(-sims):
    if mask[i]:
        print(f'{titles[i]:12s} {sims[i]:.3f}')
# -> chỉ còn sách sau 1950, xếp theo độ giống Dune
```

Bài học: **relational và vector không loại trừ nhau — chúng ghép được.** Đây là bước đệm tới tư duy staff ở Phần 4.

### 2.4. `[MỞ RỘNG]` 3 lỗi thường gặp
1. **Ép vector search làm việc của SQL** (tìm theo ID, số đơn hàng, khoảng ngày). Vector search "mờ" theo bản chất — dùng cho exact match là sai công cụ, chậm hơn và kém chính xác hơn `WHERE`.
2. **Chọn library khi thật ra cần database.** Bắt đầu với FAISS trong notebook, đến lúc cần thêm/xóa/cập nhật vector liên tục, có metadata filter, có HA thì vỡ trận — phải viết lại. <cite index="30-1">Quy tắc: FAISS cho prototype/nghiên cứu khi latency quan trọng hơn persistence; database (Pinecone/Milvus/…) cho production.</cite>
3. **Quên rằng relational DB *cũng* làm được vector giờ đây.** Elasticsearch/Postgres đều thêm được vector search — không phải cứ có nhu cầu similarity là phải dựng hệ thống mới. (Lưu ý: <cite index="32-1">full-text DB như Elasticsearch dựa trên inverted index, làm vector similarity không mạnh bằng DB chuyên dụng.</cite>)

### 2.5. ✅ Self-check tầng Intermediate
1. Vì sao B-tree index tra `year>1950` cực nhanh nhưng vô dụng cho "gần Dune nhất"?
2. Nêu 3 thứ một vector *database* có mà một vector *library* (FAISS) không có.
3. Câu SQL pgvector `ORDER BY embedding <=> ... LIMIT 3` đang làm gì, và `WHERE` trong cùng câu đó đóng vai trò gì?

---

## Phần 3 — 🔴 ADVANCED (Chuyên sâu)

### 3.1. Nội tại indexing: B-tree vs vector index, và Big-O

**B-tree (relational):**
- Cấu trúc: cây cân bằng, node chứa key *đã sắp thứ tự* + con trỏ. Chiều cao ~log_b(n).
- Exact lookup & range: **O(log n)**. Insert/delete: O(log n), tự cân bằng.
- **Tại sao hợp SQL:** dữ liệu 1 chiều, có quan hệ thứ tự toàn phần (`<`, `=`, `>`). B-tree khai thác đúng thứ tự đó.

**Vector index (HNSW/IVF):**
- Không có "thứ tự toàn phần" trong 768 chiều → B-tree bó tay. Phải dùng ANN: HNSW (graph nhiều tầng, query ~**O(log n)** nhưng hằng số lớn + ngốn RAM) hoặc IVF (chia cụm k-means). *(Chi tiết cơ chế đã có ở giáo trình trước; ở đây chỉ nhấn tương phản.)*
- Đánh đổi lõi: vector index **approximate** (đổi chút recall lấy tốc độ), còn B-tree **exact**. Đây là khác biệt triết học: relational hứa *đúng*, vector hứa *đủ gần, đủ nhanh*.

> 🔍 **Cập nhật/đính chính bài gốc (staff phải để ý):** bảng so sánh của bài đọc ghi vector DB dùng *"metric trees and hashing"*. Điều này **hơi lỗi thời**. Metric trees (KD-tree, ball tree) **sụp đổ ở chiều cao** vì curse of dimensionality; LSH (hashing) cũng đã nhường sân. Thực tế 2026: index mặc định là **HNSW** (graph), rồi tới **IVF / IVF-PQ**, DiskANN cho tỷ-scale. Một staff engineer đọc tài liệu đào tạo phải nhận ra và cập nhật những chỗ như vậy thay vì học vẹt.

### 3.2. `[MỞ RỘNG]` Consistency & transactions: ACID vs "gần đúng, cuối cùng"

Đây là trục bài gốc *không* nhắc nhưng cực quan trọng:

- **Relational = ACID** (Atomicity, Consistency, Isolation, Durability). Chuyển tiền ngân hàng: hoặc trừ-và-cộng cả hai, hoặc không gì cả; không bao giờ mất tiền giữa chừng. Đây là lãnh địa relational thống trị, và vector DB **không** cố cạnh tranh.
- **Vector DB** thường ưu tiên throughput/scale hơn strong consistency; nhiều hệ **eventually consistent**, index có độ trễ cập nhật. Bạn không đặt sổ cái tài chính lên vector DB.
- **Hệ quả thiết kế:** dữ liệu *source of truth* (đơn hàng, người dùng, số dư) sống ở relational; *biểu diễn để tìm-giống* (embedding) sống ở vector store, được sinh ra *từ* source of truth. Vector store thường là **derived data** — mất thì re-embed lại được, nên yêu cầu durability nhẹ hơn.

### 3.3. Mổ xẻ bảng so sánh 5 trục của bài gốc (bản staff)

| Trục | Traditional / Relational | Vector | Ghi chú staff `[MỞ RỘNG]` |
|---|---|---|---|
| **Data representation** | tables, rows, columns — structured data | vector nhiều chiều — encode được ảnh/text/audio/sensor (unstructured) | Vector *là derived* từ dữ liệu thô qua embedding; bản thân dữ liệu gốc vẫn nên nằm ở đâu đó có cấu trúc. |
| **Search & retrieval** | SQL, exact & range | similarity / nearest-neighbor | Không loại trừ nhau → **hybrid search** (BM25 + vector) là chuẩn production. |
| **Indexing** | B-tree | *bài gốc:* "metric trees & hashing" | **Thực tế 2026: HNSW/IVF/PQ.** Metric trees đã lỗi thời ở high-dim. |
| **Scalability** | scale khó, thường sharding | thiết kế cho horizontal scaling | Toán RAM tàn nhẫn: 1B×1536×4B ≈ 6TB → buộc quantization + shard. Relational scale write cũng đau không kém (sharding, replica). |
| **Applications** | app nghiệp vụ, hệ giao dịch | RAG, recsys, semantic search, anomaly detection, multimedia | Case thật thường **dùng cả hai** trong một sản phẩm. |

### 3.4. Code nâng cao — vì sao index "cây theo tọa độ" chết ở chiều cao (đã chạy thật ý tưởng)

Curse of dimensionality làm B-tree/KD-tree kiểu relational vô dụng cho similarity. Minh họa: ở chiều càng cao, khoảng cách nearest và farthest **co lại gần bằng nhau**, nên "cắt theo từng trục" (kiểu KD-tree) không loại được ứng viên nào:

```python
import numpy as np
np.random.seed(0)
for d in [2, 10, 100, 1000]:
    X = np.random.randn(2000, d)
    q = np.random.randn(d)
    dist = np.linalg.norm(X - q, axis=1)
    ratio = (dist.max() - dist.min()) / dist.min()
    # ratio -> 0 nghĩa là 'gần' và 'xa' gần như bằng nhau => cắt trục vô nghĩa
    print(f'd={d:4d}  (max-min)/min = {ratio:.3f}')
```

Output **thật**:
```
d=   2  (max-min)/min = 128.363
d=  10  (max-min)/min = 4.116
d= 100  (max-min)/min = 0.543
d=1000  (max-min)/min = 0.152
```

Đọc: ở `d=2`, điểm xa nhất cách query gấp ~129 lần điểm gần nhất → "gần" và "xa" phân biệt cực rõ, cắt trục hiệu quả (KD-tree ổn). Càng lên chiều cao tỉ lệ càng co: ở `d=1000` chỉ còn 0.152 (điểm xa nhất chỉ cách ~1.15 lần điểm gần nhất) → mọi điểm gần như *cách đều*, "cắt theo trục" không loại được ai. **Đây chính là lý do vector DB phải bỏ metric tree, chuyển sang graph (HNSW) khai thác cấu trúc cụm của embedding thật.** (Và là lý do câu "metric trees" trong bài gốc cần cập nhật.)

### 3.5. `[MỞ RỘNG]` Edge cases khi so hai thế giới
- **Update-heavy trên vector:** re-embed + rebuild index tốn kém; relational update một cột thì rẻ. Nếu dữ liệu đổi liên tục, cân nhắc IVF (insert rẻ) hoặc chấp nhận rebuild theo lịch.
- **Join:** relational join nhiều bảng là bản năng; vector store không có "join" đúng nghĩa — muốn kết hợp thường phải kéo id về rồi join ở relational.
- **Exact-match trong vector:** ID/tên riêng/số phiên bản → vector "mờ" làm hỏng; phải kèm keyword/filter (hybrid).
- **Cold start / OOD query:** vector search vẫn trả top-k dù query lạc quẻ → cần threshold để nói "không có kết quả đủ giống".

---

## Phần 4 — 🟣 STAFF LEVEL (Tư duy hệ thống & lãnh đạo kỹ thuật)

### 4.1. Sự thật lớn nhất: đây KHÔNG phải "hoặc/hoặc"

Bài đọc trình bày như hai lựa chọn đối lập. Ở tầm staff, câu trả lời đúng gần như luôn là: **dùng cả hai, đúng việc của mỗi loại** — gọi là **polyglot persistence**.

Kiến trúc điển hình của một sản phẩm AI:
```
[Postgres/MySQL]  <- source of truth: users, orders, documents, permissions (ACID)
        |  (embedding pipeline: đọc dữ liệu gốc -> transformer -> vector)
        v
[Vector store]    <- derived: embeddings để semantic search / RAG / recsys
```
- **Relational** giữ sự thật, quan hệ, giao dịch, phân quyền.
- **Vector** giữ *biểu diễn để tìm-giống*, có thể tái tạo bất cứ lúc nào từ relational.
- Truy vấn người dùng thường **fan-out cả hai** rồi hợp nhất (filter kiểu SQL + rank kiểu vector).

### 4.2. Quyết định kiến trúc: khi nào chọn gì

**Chỉ relational là đủ khi:** dữ liệu structured, câu hỏi là exact/range/aggregation/join, cần ACID. Đừng thêm vector DB chỉ vì "nghe hot".

**Cần vector khi:** câu hỏi bản chất là *similarity* (semantic search, RAG, recommendation, anomaly/dedup, image/audio retrieval) mà `WHERE`/`LIKE` không diễn đạt nổi.

**Cần cả hai (đa số hệ thật):** sản phẩm có cả nghiệp vụ giao dịch *và* tính năng "tìm theo ý nghĩa".

**Library hay Database?** <cite index="30-1">FAISS cho prototype/nghiên cứu khi latency quan trọng hơn persistence; Pinecone cho production semantic search/RAG ít bảo trì; Milvus cho enterprise scale phân tán; Weaviate khi cần structured filtering + hybrid search.</cite> Nếu cần CRUD/persistence/HA → database, không phải library.

**Đừng dựng hệ mới nếu chưa cần:** nếu đã chạy Postgres và <~10–50M vector → **pgvector** hợp nhất cả hai thế giới trong một hệ, ACID cho dữ liệu gốc + vector search cùng chỗ, không thêm hệ thống, không sync layer. Đây là default hợp lý cho phần lớn use case vừa và nhỏ.

**Landscape 2026 — bảng chọn nhanh:**

| Công cụ | Loại | Điểm mạnh | Chọn khi |
|---|---|---|---|
| **pgvector** | extension trên Postgres | hợp nhất relational + vector, ACID, không thêm hệ thống | đã có Postgres, <10–50M vector |
| **FAISS / Annoy / HNSWlib** | library | ANN thô, nhanh, nhẹ | prototype, static data, tự lo hạ tầng |
| **Pinecone** | managed DB | zero-ops, hybrid search, scale | không muốn lo hạ tầng, chấp nhận cost cao |
| **Qdrant** | OSS (Rust) | filtered search nhanh, latency thấp | self-host, filter nặng |
| **Weaviate** | OSS + cloud | hybrid search & multi-modal native | cần keyword+vector một query |
| **Milvus** | OSS (Zilliz) | tỷ-scale, phân tán trưởng thành | >100M–tỷ vector, có team vận hành |

### 4.3. Scale, cost, reliability, monitoring

- **Toán bộ nhớ:** 1B × 1536 chiều × 4B ≈ **6TB** vector thô → không server đơn nào giữ nổi → **quantization (int8/PQ) + sharding + fan-out**. Relational tỷ-row cũng đau: partitioning, read replica, đôi khi sharding.
- **Cost:** <cite index="34-1">mẫu hình phổ biến — khởi đầu bằng Pinecone (dễ nhất), rồi migrate sang self-host (Qdrant/Weaviate) khi scale để tối ưu chi phí, thường ở mốc ~50–100M vector hoặc hóa đơn cloud >$500/tháng.</cite> Managed đổi tiền lấy sự yên tâm; OSS đổi ops-overhead lấy tiền.
- **Bottleneck thật:** thường **KHÔNG** ở DB. Trong RAG, sinh embedding cho query + LLM inference mới ngốn phần lớn latency; DB chỉ vài ms. Đừng tối ưu nhầm tầng.
- **Failure modes & monitoring:** relational — deadlock, replication lag, hot partition. Vector — index rebuild làm phình RAM, recall drift khi đổi embedding model, "no good result" cho query lạc. Giám sát: relational nhìn query latency/lock/replication; vector nhìn **recall@k trên golden set** (recall tụt âm thầm), QPS, memory.

### 4.4. `[MỞ RỘNG]` Hybrid search & derived-data discipline
- **Hybrid search:** production nghiêm túc chạy song song **BM25 (keyword)** + **vector**, hợp nhất điểm rồi rerank — vì vector mù với ID/số/tên riêng còn keyword mù với ngữ nghĩa.
- **Kỷ luật derived data:** coi vector store là *cache thông minh* của relational. Có pipeline re-embed idempotent, versioned; khi đổi embedding model thì rebuild được từ source. Nguyên tắc: **đừng để vector store thành source of truth cho thứ không tái tạo được.**

### 4.5. Ảnh hưởng tổ chức
- **Giải thích cho non-technical stakeholder:** *"Database cũ trả lời câu hỏi có đáp án chính xác — 'đơn hàng nào sau ngày X'. Cái mới trả lời câu hỏi mờ — 'tài liệu nào *liên quan* tới câu hỏi này'. Ta cần cả hai: một cái giữ sổ sách chính xác, một cái giúp sản phẩm 'hiểu' người dùng."*
- **Ảnh hưởng roadmap:** thêm vector store = thêm một hệ phải vận hành, backup, monitor, và một *embedding pipeline* mới. Đây là cam kết vận hành dài hạn, không phải "cài thêm thư viện". Quyết định embedding model là khóa cứng (đổi = re-embed cả kho).
- **Build vs buy:** team nhỏ → managed (Pinecone) hoặc pgvector (tái dùng Postgres sẵn có) ít ma sát nhất; chọn Milvus mà không có người vận hành phân tán là tự bắn chân.

### 4.6. 🎤 Câu hỏi system design mẫu + hướng trả lời staff

> **"Thiết kế backend cho một e-commerce có (a) trang quản lý đơn hàng và (b) tính năng 'sản phẩm tương tự'. Chọn database thế nào?"**

Khung trả lời staff:
1. **Tách hai loại workload:** (a) là **relational thuần** — đơn hàng, tồn kho, thanh toán cần ACID, join, exact/range → Postgres/MySQL. Đừng đụng vector ở đây.
2. **(b) là similarity** — "sản phẩm tương tự" theo hình ảnh/mô tả → embedding + vector search.
3. **Kiến trúc polyglot:** Postgres là source of truth; một pipeline embed sản phẩm → vector store (derived). "Similar" query = fan-out: filter kiểu SQL (còn hàng, đúng danh mục, hợp giá) **+** rank vector (giống về hình/nội dung), rồi merge.
4. **Chọn tool có biện minh:** nếu catalog <10–50M và đã chạy Postgres → **pgvector** (một hệ, đỡ ops, filter+vector cùng câu SQL). Nếu >100M hoặc cần scale vector độc lập → tách Qdrant/Milvus, chấp nhận sync layer.
5. **Vận hành:** filter *trong* traversal để không sụp recall; giám sát recall@k trên golden set; embedding pipeline versioned; rebuild được từ source.
6. **Chốt bằng nhận thức trade-off:** *"Tôi không thay Postgres bằng vector DB — tôi thêm vector *bên cạnh* nó, coi embedding là derived data. Và tôi sẽ đo end-to-end trước, vì ở scale này bottleneck nhiều khả năng là embedding/inference chứ không phải bản thân DB."* → câu này phân biệt staff với senior.

---

## Phần 5 — 🎯 CHỐT LẠI ĐỂ ĐI PHỎNG VẤN (Interview Cheatsheet)

### 5.1. Keywords bắt buộc nhớ (EN + định nghĩa 1 dòng)
- **Relational database:** DB dạng tables/rows/columns, truy vấn bằng SQL, mạnh ở structured data.
- **Primary key / Foreign key:** khóa định danh duy nhất một row / khóa trỏ sang bảng khác tạo relationship.
- **SQL (SELECT/INSERT/UPDATE/DELETE):** ngôn ngữ truy vấn & thao tác của relational DB.
- **ACID:** Atomicity, Consistency, Isolation, Durability — bảo đảm giao dịch, lãnh địa của relational.
- **B-tree index:** cây cân bằng có thứ tự, cho exact & range lookup O(log n).
- **Vector database:** DB lưu vector nhiều chiều, truy vấn bằng similarity search, có full CRUD + persistence.
- **Vector library (FAISS/Annoy):** thuật toán ANN trần, in-memory, thường tĩnh, KHÔNG persistence/CRUD/API.
- **CRUD:** Create/Read/Update/Delete — thứ database có mà library thường thiếu.
- **Similarity / nearest-neighbor search:** tìm vector gần nhất với query.
- **ANN:** Approximate Nearest Neighbor — đổi chút recall lấy tốc độ (HNSW/IVF/PQ).
- **Curse of dimensionality:** ở chiều cao mọi điểm gần như cách đều → metric tree/KD-tree sụp, phải dùng graph index.
- **Polyglot persistence:** dùng nhiều loại DB, mỗi loại đúng việc, trong một hệ.
- **Derived data:** vector store thường là dữ liệu suy ra từ source of truth relational, tái tạo được.
- **Hybrid search:** BM25 (keyword) + vector + rerank.

### 5.2. Core concepts (nếu chỉ nhớ vài điều thì nhớ cái này)
1. Relational trả lời **"bằng đúng / trong khoảng"**; vector trả lời **"giống cái gì"**. Hai câu hỏi khác bản chất.
2. Khác biệt bắt nguồn từ **tầng lưu trữ**: relational sắp theo *giá trị có thứ tự* (B-tree); vector sắp theo *độ gần trong không gian* (HNSW/IVF).
3. Relational = **exact + ACID**; vector = **approximate + scale/throughput**.
4. **Vector library ≠ vector database**: library cho *thuật toán*; database cho thuật toán + persistence + CRUD + filter + API. Cần CRUD → cần database.
5. Ở high-dim, **metric tree/KD-tree chết** (curse of dimensionality) → dùng graph/cluster index. (Bài gốc nói "metric trees" là hơi cũ.)
6. Câu trả lời staff gần như luôn là **"dùng cả hai"** (polyglot); vector store là **derived data** của relational.
7. **pgvector** hợp nhất hai thế giới cho use case vừa/nhỏ — thường không cần dựng hệ vector riêng.
8. Bottleneck thật của RAG thường là **embedding + LLM**, không phải DB.

### 5.3. Ideas / mental models
- **"Thủ thư tra sổ vs thủ thư đọc rộng"** → relational vs vector.
- **"Library = động cơ; database = cả chiếc xe."** → FAISS vs Pinecone.
- **"Source of truth vs derived data"** → relational giữ sự thật; vector là biểu diễn tái tạo được.
- **"Không phải thay thế, mà là thêm bên cạnh."** → polyglot persistence.
- **"Cần CRUD thì cần database, không phải library."** → câu chốt phân biệt.

### 5.4. Code cần thuộc lòng
**(a) SQL vs vector trên cùng data** (thể hiện hiểu ranh giới):
```python
# SQL: exact/range
con.execute("SELECT title FROM books WHERE year>1950 AND rating>=4.6")
# Vector: similarity (numpy)
sims = (V @ q)                       # V đã normalize -> cosine
top  = np.argsort(-sims)[:k]
```
**(b) pgvector — filter + similarity một câu** (thể hiện biết hợp nhất):
```sql
SELECT title
FROM products
WHERE in_stock = true                     -- relational filter (B-tree)
ORDER BY embedding <=> '[...]'::vector     -- vector distance (HNSW)
LIMIT 5;
```
Khi nào dùng: catalog vừa phải, đã có Postgres, muốn một hệ thay vì hai.

### 5.5. Câu hỏi phỏng vấn thường gặp + gợi ý trả lời
1. **"Vector DB khác relational DB ở đâu?"** → data representation (vector vs table), retrieval (similarity vs SQL exact), indexing (HNSW/IVF vs B-tree), consistency (approximate vs ACID). Nhấn: hai *loại câu hỏi* khác nhau.
2. **"Vector library vs vector database?"** *(hay hỏi)* → library = ANN thô, in-memory, tĩnh, không CRUD/persistence (FAISS); database = library + persistence + CRUD + filter + replication + API (Pinecone/Milvus). "Cần CRUD → cần database."
3. **"Khi nào KHÔNG nên dùng vector DB?"** *(trap)* → khi câu hỏi là exact/range/aggregation/transaction; khi đã có Postgres và <10M vector (dùng pgvector); khi chưa đo được vector cải thiện gì.
4. **"Relational hay vector cho hệ ngân hàng?"** *(trap)* → sổ cái phải relational + ACID; vector chỉ cho tính năng phụ như phát hiện gian lận (anomaly) — và vẫn là derived, không phải source of truth.
5. **"Vì sao B-tree không dùng được cho similarity search?"** → B-tree khai thác thứ tự toàn phần 1 chiều; ở high-dim không có thứ tự đó + curse of dimensionality khiến cắt-trục vô nghĩa → phải graph/cluster index.
6. **"Thiết kế 'sản phẩm tương tự' cho e-commerce."** → polyglot: Postgres source of truth + vector derived; query = SQL filter + vector rank; pgvector nếu vừa scale, tách Qdrant/Milvus nếu lớn.
7. **"Đánh đổi lớn nhất khi thêm vector store là gì?"** → thêm một hệ phải vận hành + một embedding pipeline (khóa cứng model); vector là eventually consistent & approximate — chấp nhận được vì nó là derived data.

### 5.6. Câu trả lời "one-liner" đắt giá
- *"Relational answers 'what exactly matches'; vector answers 'what's similar'. Different questions, different storage, different engines."*
- *"FAISS gives you the algorithm; a vector database gives you the algorithm plus persistence, CRUD, filtering, and an API. Need CRUD? You need a database."*
- *"It's rarely relational vs vector — it's relational for the source of truth, vector for the derived representation, queried together."*
- *"B-trees die in high dimensions; that's the whole reason vector indexes exist."*
- *"If I already run Postgres and I'm under ~10M vectors, pgvector unifies both worlds — no second system, no sync layer."*
- *"The vector store is a smart cache of the relational truth — if I can't re-embed it from source, it shouldn't live only there."*

---

*Hết. Nếu chỉ kịp ôn 5 phút: đọc lại Mục 5.2 (core concepts) và 5.6 (one-liners) — đủ để trả lời trôi trục so sánh.*
