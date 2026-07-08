# Giáo trình: Similarity Search trong ChromaDB

> Biên soạn theo phong cách "cầm tay chỉ việc" — từ zero đến tư duy Staff Engineer.
> Giảng bằng tiếng Việt, giữ nguyên technical terms tiếng Anh để phục vụ phỏng vấn big tech.
>
> **Đây là giáo trình thứ 6 — và là capstone — của loạt về vector database.** Năm quyển trước: (1) *vector DB hoạt động thế nào* (HNSW/IVF/ANN), (2) *ứng dụng* (image/recommendation/geospatial), (3) *ChromaDB & RAG*, (4) *embeddings & distance metrics*, (5) *tạo collections & embeddings*. Quyển này khép vòng bằng **nghiệp vụ trung tâm của toàn bộ loạt: similarity search** — thao tác mà mọi thứ ở 5 quyển trước phục vụ. Đây là chỗ tất cả gặp nhau: embedding (quyển 4–5) + index HNSW (quyển 1) + distance metric (quyển 4) → **similarity search** → nuôi ứng dụng (quyển 2) và RAG (quyển 3).
>
> **Cảnh báo ngay từ đầu:** bài giảng gốc (1) có **code JS sai tham số** (`n: 3` — đúng phải là `nResults`), (2) **gộp nhầm "5 loại similarity search"**: trộn thứ *thực sự là vector/embedding similarity* (text, image, recommendation) với thứ **KHÁC hẳn** (structured data dùng *edit distance*; genomics truyền thống dùng *sequence alignment*) — mà ChromaDB **không** làm hai cái sau bằng embedding, (3) nhầm **"distances" với "similarity scores"** trong kết quả trả về, và (4) có vài câu **lủng củng do transcript** ("để ý dấu chấm sau chữ coffee", "dòng comment tên collection là lỗi"). Tôi giữ phần đúng, đính chính, và chạy thật code. Đánh dấu 🛠️ **[Đính chính]** và ➕ **[Mở rộng ngoài bài gốc]**.

---

## Phần 0 — 🗺️ Bản đồ bài học (Overview)

Cả 5 quyển trước đều xoay quanh một động tác duy nhất mà giờ ta gọi đích danh: **similarity search** — *"cho một item tham chiếu (query), tìm những item *giống nó nhất* trong tập dữ liệu."* Bài giảng gốc định nghĩa đúng điều đó (similarity search là thao tác nền tảng của information retrieval), liệt kê **5 loại** (text, image/video, structured data, recommendation, genomics), rồi đưa code ChromaDB chạy một query "coffee". Ta sẽ đi qua tất cả, nhưng **tách bạch** loại nào *thực sự* là vector/embedding similarity (việc của ChromaDB) và loại nào là *cơ chế khác* (edit distance, alignment) chỉ tình cờ cùng tên "similarity".

**Sau khi học xong, bạn sẽ có thể:**
- Giải thích similarity search là gì, vì sao nó là "trái tim" của vector database.
- Phân biệt **semantic similarity (embedding)** với **syntactic/string similarity (edit distance)** và **sequence alignment** — ba cơ chế khác nhau, đừng lẫn.
- Đọc và viết đúng một **ChromaDB query** (queryTexts/queryEmbeddings, `nResults`, `where` filter, cấu trúc kết quả) — với API 2026.
- Hiểu **distance vs similarity score** và vì sao cần **threshold**.
- Biết mỗi trong "5 loại" thực chất chạy bằng cơ chế nào, và khi nào ChromaDB *không* phải công cụ đúng.
- Thiết kế similarity search ở quy mô lớn (2 tầng, hybrid, cost).

**Mạch kiến thức từ basic → staff:**
- 🟢 **Basic:** similarity search là gì (analogy) → giải phẫu một query → hello-world "coffee".
- 🟡 **Intermediate:** 5 loại similarity search (kèm tách bạch embedding vs không-embedding) → API query đúng 2026 → distance vs similarity → lỗi thường gặp.
- 🔴 **Advanced:** cơ chế thật (ANN/HNSW), tự implement edit distance để thấy nó *khác* vector similarity, threshold, khi embedding thắng cổ điển (genomics ESM), Big-O.
- 🟣 **Staff:** similarity search ở scale (2 tầng retrieve→rerank, hybrid), cost, khi nào KHÔNG dùng vector similarity, monitoring, system design.
- 🎯 **Cheatsheet + tổng kết cả 6 quyển.**

---

## Phần 1 — 🟢 BASIC (Nền tảng)

### 1.1. "Tại sao" — bài toán tìm-thứ-giống-nhau

Bạn có 1 triệu bài viết. Người dùng gõ *"quán cà phê yên tĩnh để làm việc"*. Họ **không** biết tiêu đề bài nào. Bạn cần tìm những bài *giống về ý nghĩa* với câu đó. Đây là **similarity search** — thao tác nền tảng của **information retrieval** (truy hồi thông tin). Nó khác hẳn "tìm chính xác" (exact match) của SQL: ta tìm *gần giống*, không phải *trùng khớp*.

Điểm cốt lõi (nối cả loạt): độ "giống" **phụ thuộc ngữ cảnh** — giống về *ý nghĩa* (semantic), về *chữ* (syntactic), về *đặc trưng ảnh*, hay về *chuỗi DNA*? Mỗi kiểu "giống" cần một cách đo khác nhau. ChromaDB chuyên về **semantic similarity qua embedding**.

### 1.2. Analogy đời thường

Tưởng tượng một **quản thủ thư viện có trí nhớ ý nghĩa**. Bạn đưa một cuốn sách và nói "tìm cho tôi 3 cuốn *giống cuốn này nhất*". Anh ta không so từng chữ trên bìa — anh ta hiểu *nội dung* và với tay ra khu vực sách *cùng chủ đề*. "3 cuốn giống nhất" = **top-3 similarity search**. "Hiểu nội dung để đặt sách gần nhau" = **embedding** (quyển 4–5). "Với tay ra khu vực lân cận thật nhanh" = **HNSW index** (quyển 1). Similarity search = *toàn bộ động tác* đó.

### 1.3. Thuật ngữ nền tảng (giải thích ngay lần đầu)

- **Similarity search:** tìm item giống nhất với một query item trong tập dữ liệu. Nghiệp vụ trung tâm của vector DB.
- **Query (truy vấn):** item tham chiếu bạn muốn tìm thứ giống nó (một câu text, một ảnh, một vector).
- **top-k / nResults:** số kết quả gần nhất muốn lấy (vd top-3).
- **Distance:** khoảng cách giữa hai vector; **thấp = giống** (quyển 4).
- **Similarity score:** độ giống; **cao = giống**. Là *nghịch đảo* của distance, không phải hai thứ riêng biệt.
- **Semantic similarity:** giống về *ý nghĩa* (embedding). — thứ ChromaDB làm.
- **Syntactic/string similarity:** giống về *chuỗi ký tự* (vd edit distance). — cơ chế khác, không phải embedding.
- **queryTexts / queryEmbeddings:** đưa query dạng *text* (Chroma tự embed) hoặc dạng *vector sẵn*.
- **where:** bộ lọc metadata trong query (vd chỉ `cat="drink"`).

### 1.4. Giải phẫu một similarity search (từng bước)

Mọi similarity search trong ChromaDB gồm các bước:
1. **Nhận query** (text hoặc vector).
2. **Embed query** — nếu là text, biến thành vector *bằng đúng embedding function của collection* (quyển 5).
3. **Tìm k gần nhất** — HNSW so query vector với các vector trong collection, dùng distance metric đã chọn (quyển 1 & 4).
4. **Lọc (tùy chọn)** — áp `where` filter theo metadata.
5. **Trả kết quả** — ids, documents, metadatas, **distances**.

### 1.5. Ví dụ chạy tay — top-3 cho query "coffee"

Collection có 5 document, đã embed. Query = "coffee". Ta đo distance (cosine distance, thấp = giống) tới từng doc:

| Doc | Nội dung | Cosine distance tới "coffee" | Hạng |
|---|---|---|---|
| d3 | "coffee beans roasted dark, best coffee ever" | 0.041 | 🥇 |
| d1 | "I love a strong espresso coffee" | 0.093 | 🥈 |
| d4 | "a fast red sports car" | 0.954 | 🥉 (nhưng rất xa!) |

→ top-3 = [d3, d1, d4]. Nhận xét quan trọng: **d4 (xe hơi) lọt top-3 dù hoàn toàn không liên quan** — vì "top-3" *luôn* trả về 3 cái, kể cả khi cái thứ 3 rất xa. Đây chính là lý do cần **threshold** (Phần 3) — nếu không, bạn sẽ đưa rác vào kết quả (và vào prompt RAG — quyển 3).

### 1.6. Code "hello world" — query "coffee" (đã chạy thật)

```python
import chromadb
# (giả sử collection 'beverages' đã tạo với embedding function + add 5 docs)
res = col.query(query_texts=["coffee"], n_results=3)   # tự embed "coffee" rồi tìm top-3

for i in range(len(res["ids"][0])):
    print(res["ids"][0][i], round(res["distances"][0][i], 3), res["documents"][0][i])
# d3 0.041 'coffee beans roasted dark, best coffee ever'
# d1 0.093 'I love a strong espresso coffee'
# d4 0.954 'a fast red sports car'      <-- xa, nhưng vẫn lọt top-3
```
Giải thích: `query_texts=["coffee"]` → Chroma **tự embed** "coffee" bằng EF của collection (quyển 5), rồi HNSW tìm 3 vector gần nhất. Kết quả trả về `ids, documents, metadatas, distances`.

### 1.7. ✅ Self-check (Basic)

1. Similarity search khác exact match (SQL) ở điểm gì?
2. Trong ví dụ "coffee", vì sao document về xe hơi vẫn lọt top-3? Điều đó gợi ý ta cần gì?
3. "Distance" và "similarity score" có phải hai thứ khác nhau không?

---

## Phần 2 — 🟡 INTERMEDIATE (Vận dụng)

### 2.1. Năm loại similarity search — TÁCH BẠCH embedding vs không-embedding

Bài gốc liệt kê 5 loại. Nhưng gộp chung chúng gây hiểu nhầm rằng ChromaDB làm được tất cả bằng embedding. Sự thật:

| Loại | Cơ chế THẬT | ChromaDB (embedding) làm? |
|---|---|---|
| **① Text document** | Word/document **embedding** → vector similarity | ✅ Đúng chuyên môn |
| **② Image / Video** | Deep NN **embedding** (CLIP/SigLIP — quyển 2), color histogram | ✅ Đúng chuyên môn |
| **④ Recommendation** | User/item **embedding** (two-tower — quyển 2) → vector similarity | ✅ Đúng (tầng retrieval) |
| **③ Structured data** | Euclidean trên số, **edit distance** trên chuỗi | ⚠️ Edit distance **KHÔNG** phải embedding — ChromaDB không làm |
| **⑤ Genomics** | Truyền thống: **sequence alignment (BLAST, edit distance)**. Hiện đại: protein **embedding** (ESM-2) | ⚠️ Cổ điển KHÔNG phải vector DB; hiện đại thì có |

🛠️ **[Đính chính cốt lõi]** Ba loại ①②④ là **semantic similarity qua embedding** — đúng sân của ChromaDB. Nhưng ③ (edit distance) và ⑤ (alignment cổ điển) là **cơ chế hoàn toàn khác**, ChromaDB *không* làm bằng embedding. Gộp cả 5 vào "similarity search kiểu vector DB" là sai. Ta sẽ làm rõ ③ và ⑤ ở Phần 3.

### 2.2. API query đúng 2026 (sửa code lỗi của bài gốc)

🛠️ **[Đính chính code]** Bài gốc viết `collection.query({ queryTexts: ["coffee"], n: 3 })` và giải thích "n parameter = 3". **Sai tên tham số.** Trong ChromaDB, tham số là **`nResults`** (JS) / **`n_results`** (Python), *không phải* `n`.

Ngoài ra bài gốc còn hai câu lủng củng do transcript:
- *"để ý dấu chấm sau chữ coffee"* → chỉ là nhiễu/typo trong code gốc, không mang ý nghĩa gì.
- *"dòng comment tên collection là một lỗi... collection argument phải chỉ định tên collection"* → **lủng củng.** Thực tế: `collection.query()` được gọi **trên một collection object** (bạn đã có object rồi), *không* truyền "tên collection" vào trong `query`. Bạn lấy object bằng `getCollection(name)`/`get_collection(name)` *trước*, rồi gọi `.query()` trên nó.

**Query đúng chuẩn — JavaScript 2026:**
```javascript
// collection đã được tạo/lấy trước đó (quyển 5)
async function performSimilaritySearch(collection) {
  try {                                                  // error handling (bài gốc đúng ý này)
    const results = await collection.query({
      queryTexts: ["coffee"],                            // Chroma tự embed
      nResults: 3,                                       // ĐÚNG: nResults, KHÔNG phải n
      where: { cat: "drink" },                           // (tùy chọn) lọc metadata
    });
    console.log("Similar documents:", results.documents);
  } catch (err) {
    console.error("Search failed:", err);                // xử lý lỗi gọn gàng
  }
}
performSimilaritySearch(collection);                     // gọi hàm, truyền collection OBJECT
```

**Query đúng chuẩn — Python (đã chạy thật):**
```python
res = col.query(
    query_texts=["coffee"],        # hoặc query_embeddings=[[...]] nếu có vector sẵn
    n_results=3,                   # top-k
    where={"cat": "drink"},        # (tùy chọn) chỉ lấy metadata cat=drink
    # include=["documents","distances","metadatas"]  # chọn field trả về
)
print(res["documents"][0])
# -> ['coffee beans roasted dark, best coffee ever', 'I love a strong espresso coffee', 'Green tea is calming']
```

### 2.3. Distance vs Similarity score — cấu trúc kết quả thật

🛠️ Bài gốc nói kết quả chứa *"IDs, distances, similarity scores, và metadata"*. **Không chính xác:** ChromaDB trả về `ids, documents, metadatas, distances` — **chỉ có `distances`, KHÔNG có trường "similarity score" riêng.** Distance và similarity là *nghịch đảo* của nhau (quyển 4): với `space="cosine"`, Chroma trả **cosine *distance*** (thấp = giống). Muốn "similarity" thì tự tính `similarity = 1 − distance`. Đừng tưởng đó là hai con số riêng biệt.

```python
res = col.query(query_texts=["coffee"], n_results=3)
print("keys:", [k for k,v in res.items() if v is not None])
# -> ['ids', 'documents', 'included', 'metadatas', 'distances']   (KHÔNG có 'similarity_scores')
```

### 2.4. 3 lỗi thường gặp (và cách tránh)

1. **Dùng `n` thay `nResults`/`n_results`.** → Sai tên tham số → lỗi/không chạy. Nhớ `nResults` (JS), `n_results` (Python).
2. **Tưởng ChromaDB làm được mọi "similarity".** → ChromaDB làm **embedding/vector similarity**; *không* làm edit distance hay sequence alignment cổ điển. Cần string similarity → dùng thư viện khác (Levenshtein).
3. **Không đặt threshold → nhét rác vào kết quả.** → top-k luôn trả đủ k, kể cả cái xa. Lọc theo ngưỡng distance (Phần 3).

### 2.5. ✅ Self-check (Intermediate)

1. Trong "5 loại", loại nào ChromaDB *thực sự* làm bằng embedding, loại nào không? Vì sao?
2. Tham số top-k trong ChromaDB tên là gì (JS và Python)? Bài gốc viết sai thành gì?
3. Kết quả query trả về "distances" hay "similarity scores"? Chuyển đổi thế nào?

---

## Phần 3 — 🔴 ADVANCED (Chuyên sâu)

### 3.1. Cơ chế thật: similarity search = distance metric + ANN index

Bài gốc mô tả "collection.query thực hiện similarity search" như một hộp đen. Bên trong nó là **hai thứ độc lập** (đã học rời ở quyển 1 & 4, giờ ghép lại):
- **Distance metric** (cosine/L2/ip) — *đo* gần/xa giữa hai vector (quyển 4).
- **ANN index (HNSW)** — *tìm nhanh* các vector gần nhất mà không quét hết N (quyển 1), cho **O(log N)** thay vì O(N).

→ Similarity search = "dùng HNSW để nhanh chóng tìm k vector có distance nhỏ nhất tới query". Kết quả là **approximate** (gần đúng), đánh đổi recall lấy tốc độ. Bài gốc hoàn toàn bỏ qua tầng ANN này — nhưng nó mới là thứ khiến search *nhanh*.

### 3.2. Edit distance — tự implement để thấy nó KHÁC vector similarity

Bài gốc nói structured/genomics similarity dùng **edit distance**. Đây là thuật toán **chuỗi** cổ điển (Levenshtein), *không* liên quan embedding. Tự implement để thấy rõ:

```python
def edit_distance(a, b):
    """Số phép chèn/xóa/thay tối thiểu để biến chuỗi a -> b (Levenshtein)."""
    m, n = len(a), len(b)
    dp = [[0]*(n+1) for _ in range(m+1)]
    for i in range(m+1): dp[i][0] = i           # xóa hết a
    for j in range(n+1): dp[0][j] = j           # chèn hết b
    for i in range(1, m+1):
        for j in range(1, n+1):
            cost = 0 if a[i-1] == b[j-1] else 1
            dp[i][j] = min(dp[i-1][j] + 1,       # xóa
                           dp[i][j-1] + 1,       # chèn
                           dp[i-1][j-1] + cost)  # thay
    return dp[m][n]

print(edit_distance("kitten", "sitting"))  # 3
print(edit_distance("coffee", "tea"))      # 5   <-- rất XA về CHỮ
```
**Đối chiếu (đã chạy thật):** `edit("coffee","tea") = 5` (rất xa về *ký tự*). Nhưng về **embedding**, "coffee" và "tea" **gần nhau** (đều là đồ uống). 👉 Đây là điểm "aha": **edit distance đo giống về *chuỗi ký tự* (syntactic); embedding đo giống về *ý nghĩa* (semantic) — hai loại "similarity" hoàn toàn khác nhau.** ChromaDB làm cái sau; edit distance là cái trước. Big-O của edit distance = **O(m·n)** (quy hoạch động), khác hẳn O(log N) của HNSW.

### 3.3. Genomics: khi embedding *thay thế* alignment cổ điển (kiến thức 2026)

➕ **[Mở rộng — làm rõ loại ⑤]** Bài gốc nói genomics dùng similarity search để tìm chuỗi DNA/RNA/protein giống nhau. Chính xác hơn:
- **Truyền thống:** dùng **sequence alignment** — **BLAST** (một trong các công trình được trích dẫn nhiều nhất lịch sử khoa học), Smith-Waterman — dựa trên **weighted edit distance + ma trận thay thế (BLOSUM/PAM)**. **Đây KHÔNG phải vector DB.**
- **Hiện đại (2024–2026):** dùng **protein language model embeddings** (ESM-2, ProtT5, Ankh) rồi tính **cosine similarity** giữa các embedding — và cách này **thường thắng BLOSUM/BLAST** cho "remote homology" (họ hàng xa). Đây **chính là vector similarity** → vector DB (như ChromaDB) trở nên hữu ích. Trích một nghiên cứu: cách tiếp cận vector similarity "có thể tăng tốc đáng kể bằng phần cứng hiện đại và hashing" — tức đúng địa hạt ANN/HNSW.

→ Bài học staff: "genomics similarity" đang **chuyển dịch** từ alignment cổ điển sang embedding. Nói được cả hai (và biết cái nào là vector DB) là tín hiệu hiểu sâu.

### 3.4. Threshold — cái phanh chống rác (nối quyển 4)

Từ ví dụ "coffee": xe hơi lọt top-3 với distance 0.954. Giải pháp: **đặt ngưỡng distance**. Chỉ nhận kết quả nếu đủ gần:
```python
res = col.query(query_texts=["coffee"], n_results=3)
GOOD = [(doc, d) for doc, d in zip(res["documents"][0], res["distances"][0]) if d < 0.5]
# -> loại bỏ xe hơi (0.954), chỉ giữ các document đủ liên quan
```
Đây là cơ chế **"nếu không đủ giống → đừng trả"** — chống đưa rác vào RAG (quyển 3) hay recommendation. Ngưỡng phụ thuộc metric/model/domain, phải đo trên ground-truth.

### 3.5. Bảng: mỗi loại similarity → cơ chế → công cụ

| Loại similarity | Đo bằng | Công cụ điển hình | Big-O |
|---|---|---|---|
| Semantic (text/image/rec) | Embedding + cosine/L2 | **ChromaDB + HNSW** | ~O(log N) search |
| String/syntactic | Edit distance (Levenshtein) | thư viện string, DB text | O(m·n) mỗi cặp |
| Sequence (genomics cổ điển) | Alignment (BLAST/SW) | BLAST, Smith-Waterman | ~O(m·n) mỗi cặp |
| Sequence (genomics hiện đại) | Protein embedding + cosine | ESM-2 + vector DB | ~O(log N) search |
| Numeric tabular | Euclidean k-NN | vector DB hoặc k-NN cổ điển | tùy index |

### 3.6. Edge cases phải xử lý

- **top-k trả rác khi kho nhỏ/không có match thật** → threshold.
- **Query rỗng / ngoài domain** → embedding vô nghĩa; recall thấp; nên có fallback.
- **Metric không khớp embedding model** (quyển 4) → xếp hạng lệch.
- **Kết quả trùng lặp (near-duplicate)** → giảm diversity; dedup hoặc rerank.
- **Nhầm distance với similarity** → sắp xếp ngược; nhớ cosine *distance* gần 0 = giống.

---

## Phần 4 — 🟣 STAFF LEVEL (Tư duy hệ thống & lãnh đạo kỹ thuật)

### 4.1. Similarity search ở quy mô lớn — hiếm khi là một cú query đơn

➕ Ở tầm production, một "similarity search" tốt thường là **pipeline nhiều tầng** (tổng hợp cả loạt):
1. **Retrieval (tầng thô):** ChromaDB/HNSW trả top-k lớn (vd 100–500) — nhanh, recall cao (quyển 1 & 2).
2. **Re-ranking (tầng tinh):** cross-encoder chấm lại → top-k nhỏ chính xác (quyển 2 & 3).
3. **Hybrid search:** kết hợp **vector (semantic) + BM25/full-text (keyword) + metadata filter** — ChromaDB bản mới hỗ trợ dense + sparse(BM25) + full-text + regex trong một interface (quyển 3). Với tên riêng/mã/ID mà embedding hay bỏ sót, hybrid là bắt buộc.

→ Bài gốc dừng ở "một query trả top-3". Staff phải thấy cả pipeline.

### 4.2. Bottleneck & cost

- **Latency:** cộng dồn embed query (nếu dùng API EF → mạng) + HNSW search + rerank. Cache query nóng (semantic cache) giúp nhiều.
- **Cost:** với API embedding, *mỗi query* tốn tiền embed; rerank bằng cross-encoder tốn compute; ở RAG, LLM generation mới thống trị cost (quyển 3).
- **Scale kho:** vài triệu vector → Chroma OK; tỷ vector → chuyển Milvus/Qdrant (quyển 1); Chroma không cho billion-scale.

### 4.3. Khi nào KHÔNG dùng vector similarity

Staff biết nói "không":
- **Cần match *chính xác chuỗi*** (tên file, mã đơn, biển số) → **exact match / edit distance / full-text**, không phải embedding (nhớ: "coffee" vs "tea" gần về nghĩa nhưng đó không phải điều bạn muốn khi tìm mã sản phẩm).
- **Bài toán là *căn chỉnh chuỗi* sinh học truyền thống** → BLAST/Smith-Waterman (trừ khi bạn chủ động chuyển sang embedding).
- **Dữ liệu nhỏ, luật rõ** → SQL/filter thường; đừng dựng vector DB.
- **Cần giải thích chính xác vì sao match** → embedding là hộp đen; string/rule minh bạch hơn.

### 4.4. Reliability & monitoring

- **Giám sát phân bố distance** của top-1: nếu trung bình *tăng dần* → drift / model không hợp domain.
- **Tỉ lệ query dưới threshold** ("không tìm thấy"): tăng đột biến → dữ liệu thiếu hoặc embedding hỏng.
- **Recall@k trên ground-truth** (quyển 1): search có thể *chạy* nhưng trả *sai* mà không báo lỗi ("hỏng âm thầm").
- **Failure modes:** EF tải model fail (quyển 5), API embedding rate-limit, metric sai, near-duplicate làm lệch.

### 4.5. Giải thích cho stakeholder & 🎤 System design mẫu

**Cho stakeholder non-technical:** *"Tìm-theo-ý-nghĩa: người dùng gõ 'cà phê yên tĩnh' vẫn ra đúng quán dù mô tả không chứa mấy chữ đó. Đánh đổi: nó tìm *gần giống* chứ không *trùng khớp* — nên với thứ cần khớp chính xác (mã đơn, tên file) ta vẫn dùng tìm kiếm truyền thống. Và 'top 3' luôn trả 3 kết quả kể cả khi không cái nào thật sự hợp — nên ta đặt ngưỡng để tránh trả rác."*

> **Đề: "Thiết kế semantic search cho 50 triệu tài liệu, cho phép lọc theo phòng ban & ngày, p99 < 150ms, và không được trả kết quả không liên quan."**

Hướng trả lời staff:
1. **Làm rõ trước:** "không liên quan" → cần **threshold**; lọc phòng ban/ngày → **metadata filter (`where`)**; recall mục tiêu?
2. **Index & metric:** HNSW, `space="cosine"`, embedding model multilingual (quyển 4–5), normalize.
3. **Pipeline:** query → embed (cache) → **hybrid** (vector + BM25 để bắt tên riêng) top-100 **kèm `where` filter** phòng ban/ngày → **rerank** cross-encoder → **threshold** loại rác → top-k.
4. **Scale:** 50M → shard/replicate (quyển 1); tách compute/storage.
5. **Đáp p99:** cắt `ef_search` động, cache, giới hạn rerank.
6. **Chống trả rác:** threshold + "không đủ giống → trả rỗng, đừng bịa".
7. **Monitoring:** recall@k + phân bố distance + tỉ lệ rỗng.
8. **Kết bằng trade-off:** vector vs hybrid; threshold chặt (ít rác, có thể sót) vs lỏng. → *đánh đổi*.

---

## Phần 5 — 🎯 CHỐT LẠI ĐỂ ĐI PHỎNG VẤN (Interview Cheatsheet)

### 5.1. Keywords bắt buộc nhớ (định nghĩa 1 dòng)

- **Similarity search:** tìm item giống nhất với query; nghiệp vụ trung tâm của vector DB.
- **Semantic similarity:** giống về *ý nghĩa* (embedding) — việc của ChromaDB.
- **Syntactic/string similarity:** giống về *chuỗi ký tự* (edit distance) — cơ chế khác.
- **Edit distance (Levenshtein):** số phép chèn/xóa/thay tối thiểu giữa 2 chuỗi; O(m·n); KHÔNG phải embedding.
- **Sequence alignment (BLAST/Smith-Waterman):** căn chỉnh chuỗi sinh học cổ điển; dựa edit distance + BLOSUM.
- **Protein embedding (ESM-2/ProtT5):** biến chuỗi protein → vector; cosine similarity; xu hướng thay alignment.
- **top-k / nResults / n_results:** số kết quả gần nhất; **tên đúng là nResults (JS), n_results (Python)** — KHÔNG phải `n`.
- **queryTexts / queryEmbeddings:** query dạng text (auto-embed) hoặc vector sẵn.
- **where:** filter metadata trong query.
- **distance vs similarity score:** nghịch đảo nhau; Chroma trả **distances** (cosine distance gần 0 = giống).
- **threshold:** ngưỡng distance để loại kết quả không đủ giống (chống rác).
- **ANN / HNSW:** cơ chế tìm nhanh gần đúng O(log N) (quyển 1).
- **hybrid search:** vector + BM25/full-text + filter.
- **re-ranking:** tầng cross-encoder chấm tinh sau retrieval.

### 5.2. Core concepts (nếu chỉ nhớ mấy điều này)

1. **Similarity search = distance metric (đo) + ANN/HNSW (tìm nhanh).** Kết quả gần đúng, đánh đổi recall lấy tốc độ.
2. **ChromaDB làm *semantic* similarity qua embedding** — KHÔNG làm edit distance/alignment cổ điển.
3. **"5 loại" tách làm hai nhóm:** embedding (text/image/rec — sân nhà) vs không-embedding (structured-string/genomics cổ điển — cơ chế khác).
4. **Semantic ≠ syntactic:** "coffee" vs "tea" gần về *nghĩa* (embedding) nhưng xa về *chữ* (edit distance = 5).
5. **top-k luôn trả đủ k** kể cả rác → cần **threshold**.
6. **Kết quả có `distances`, không có "similarity score" riêng;** distance thấp = giống (cosine distance).
7. **API đúng: `nResults`/`n_results`, không phải `n`;** gọi `.query()` trên collection object.
8. **Ở scale: retrieve (thô) → rerank (tinh) → hybrid → threshold;** hiếm khi là một cú query đơn.

### 5.3. Ideas / mental models

- **"Quản thủ thư viện trí nhớ ý nghĩa"** với tay ra khu vực sách cùng chủ đề = similarity search.
- **"Semantic (nghĩa) vs syntactic (chữ)"** — coffee/tea gần nghĩa, xa chữ.
- **"top-k là cái lưới luôn kéo lên đủ k con cá, kể cả rác"** → cần threshold.
- **"distance đi xuống, similarity đi lên"** — cùng thông tin, nghịch chiều.
- **"Embedding đang xâm chiếm cả genomics"** — ESM-2 thay BLAST.
- **"Search = đo (metric) + tìm (HNSW)"** — hai chuyện tách rời.

### 5.4. Code cần thuộc lòng

**(1) ChromaDB similarity search đúng chuẩn:**
```python
res = col.query(query_texts=["coffee"], n_results=3, where={"cat":"drink"})  # n_results, KHÔNG n
best = [(d, dist) for d, dist in zip(res["documents"][0], res["distances"][0]) if dist < 0.5]  # threshold
```
*Khi nào dùng:* mọi truy hồi semantic; nhớ threshold để chống rác.

**(2) Edit distance (string similarity — để đối chiếu, interviewer hay hỏi):**
```python
def edit_distance(a, b):
    dp = list(range(len(b)+1))
    for i, ca in enumerate(a, 1):
        prev, dp[0] = dp[0], i
        for j, cb in enumerate(b, 1):
            prev, dp[j] = dp[j], min(dp[j]+1, dp[j-1]+1, prev+(ca!=cb))
    return dp[-1]
```
*Khi nào dùng:* fuzzy string match, chính tả, mã/tên — KHÔNG phải semantic.

**(3) JS query 2026 (sửa lỗi bài gốc):**
```javascript
const results = await collection.query({ queryTexts: ["coffee"], nResults: 3 }); // nResults!
```

### 5.5. Câu hỏi phỏng vấn thường gặp + gợi ý trả lời (có câu bẫy)

1. **"Similarity search bên trong hoạt động thế nào?"** → **distance metric** (đo gần/xa) + **ANN/HNSW** (tìm nhanh O(log N), gần đúng). Không phải quét exact.

2. **"ChromaDB có làm được fuzzy string match (kiểu 'cofee' → 'coffee') không?"** *(câu bẫy semantic vs syntactic)* → Không phải bằng embedding; đó là **edit distance** (string). Embedding bắt *ý nghĩa*, không bắt *lỗi chính tả ký tự*. Cần tool khác (Levenshtein/full-text).

3. **"top-3 luôn trả 3 kết quả — nếu không cái nào liên quan thì sao?"** *(câu bẫy)* → Vẫn trả 3 (kể cả rác). → Đặt **threshold** trên distance; dưới ngưỡng → loại/trả rỗng.

4. **"Kết quả query trả về similarity score à?"** *(câu bẫy)* → Trả **distances** (Chroma). Similarity = nghịch đảo; với cosine, distance gần 0 = giống. Không có trường score riêng.

5. **"Genomics dùng vector DB để tìm chuỗi giống, đúng không?"** *(câu bẫy cổ điển vs hiện đại)* → Truyền thống dùng **alignment (BLAST)**, không phải vector DB. Hiện đại dùng **protein embedding (ESM-2) + cosine** → mới là vector DB, và thường thắng BLAST cho remote homology.

6. **"Tham số top-k trong ChromaDB tên gì?"** *(câu bẫy — bài giảng sai)* → **`nResults`** (JS) / **`n_results`** (Python). Không phải `n`.

7. **"Khi nào KHÔNG nên dùng semantic search?"** → Khi cần **match chính xác chuỗi** (mã đơn, biển số, tên file) → exact/edit distance/full-text; dữ liệu nhỏ luật rõ → SQL; cần explainability → rule.

### 5.6. One-liner đắt giá

- *"Similarity search = **đo bằng distance metric, tìm nhanh bằng HNSW** — hai chuyện tách rời, kết quả gần đúng."*
- *"ChromaDB làm **semantic** similarity (ý nghĩa), không phải **syntactic** (chữ) — 'coffee' và 'tea' gần về nghĩa nhưng xa 5 phép edit về chữ."*
- *"**top-k là cái lưới luôn kéo đủ k con cá, kể cả rác** — nên tôi luôn kèm một threshold trên distance."*
- *"Chroma trả **distances chứ không phải similarity score** — cosine distance gần 0 mới là giống; nhầm chiều là xếp hạng ngược."*
- *"Genomics đang chuyển từ **BLAST (alignment)** sang **ESM embeddings + cosine** — đó là lúc vector database bước vào sinh học."*
- *"Tham số top-k là **nResults**, không phải `n` — và `.query()` gọi trên collection *object*, không truyền tên collection vào."*

---

### 🏁 TỔNG KẾT CẢ LOẠT 6 QUYỂN (capstone)

| Quyển | Chủ đề | Một câu cốt lõi |
|---|---|---|
| 1 | Các loại vector database | Brute-force O(N) chết ở scale → **ANN/HNSW** O(log N), đánh đổi recall lấy tốc độ. |
| 2 | Ứng dụng vector DB | Embed mọi thứ → mọi bài toán quy về similarity search; recsys là **retrieval→ranking 2 tầng**; "vector" GIS ≠ "vector" embedding. |
| 3 | ChromaDB & RAG | RAG = **thi mở sách ở inference-time** (không train); vá knowledge-cutoff & private-data, không vá bias. |
| 4 | Embeddings & distance metrics | Cosine ∈ **[−1,1]** (không phải 0–1); đo **hướng** không đo độ lớn; normalize → cosine≡L2. |
| 5 | Collections & embeddings | Collection ≈ ngăn tủ (không phải SQL table); **EF là contract** — đổi model = re-embed cả kho. |
| 6 | Similarity search | **Semantic (embedding) ≠ syntactic (edit distance);** search = metric + HNSW; top-k cần **threshold**. |

**Sợi chỉ đỏ xuyên suốt:** *Embed dữ liệu thành vector (4,5) → lưu trong collection (5) → index bằng HNSW (1) → **similarity search** (6) → nuôi ứng dụng (2) và RAG (3).* Nắm sợi chỉ này là nắm toàn bộ vector database.

---

### 📌 Phụ lục: những chỗ bài giảng gốc sai / thiếu

1. **🚨 Code JS sai tham số:** `n: 3` → phải **`nResults: 3`** (JS) / `n_results=3` (Python).
2. **🚨 Gộp nhầm "5 loại":** trộn embedding similarity (text/image/rec) với **edit distance** (structured-string) và **alignment cổ điển** (genomics) — ChromaDB *không* làm hai cái sau bằng embedding.
3. **🚨 Nhầm kết quả:** nói có "distances **và** similarity scores"; thực tế chỉ có **`distances`** (similarity là nghịch đảo, tự tính).
4. **Bỏ qua ANN/HNSW:** mô tả query như hộp đen exact; thực ra là approximate qua HNSW.
5. **Không nhắc threshold:** top-k luôn trả đủ k kể cả rác — thiếu cơ chế lọc.
6. **Câu lủng củng do transcript:** "để ý dấu chấm sau coffee" (nhiễu), "comment tên collection là lỗi" (thực ra `.query()` gọi trên object, không truyền tên).
7. **Genomics đơn giản hoá:** không phân biệt alignment cổ điển (BLAST) vs embedding hiện đại (ESM-2).

> **Phần đúng & giá trị của bài gốc:** định nghĩa **similarity search là thao tác nền tảng của information retrieval, phụ thuộc ngữ cảnh** — rất chuẩn; khung **5 lĩnh vực ứng dụng** (text/image/structured/recommendation/genomics) là cách điểm danh use case hợp lý; nhấn mạnh **error handling (try/catch)** quanh query là thói quen tốt; ý **structured data dùng metric riêng theo kiểu dữ liệu** (Euclidean cho số, edit distance cho chuỗi) là đúng — chỉ cần làm rõ đó không phải embedding. Giữ những phần đó, sửa code + phân loại + cấu trúc kết quả, và bổ sung cơ chế ANN + threshold + tư duy pipeline ở scale.
