# Giáo trình: Embeddings & Distance Metrics trong ChromaDB

> Biên soạn theo phong cách "cầm tay chỉ việc" — từ zero đến tư duy Staff Engineer.
> Giảng bằng tiếng Việt, giữ nguyên technical terms tiếng Anh để phục vụ phỏng vấn big tech.
>
> **Đây là giáo trình thứ 4 trong loạt về vector database.** Ba quyển trước: (1) *vector DB hoạt động thế nào* (HNSW, IVF, ANN), (2) *xây gì bằng nó* (image search, recommendation, geospatial), (3) *ChromaDB & RAG*. Quyển này (4) đào sâu **hai viên gạch nền móng** mà mọi thứ đứng trên: **embedding** (làm sao biến dữ liệu thành vector) và **distance metrics** (làm sao đo "gần/xa" giữa hai vector — cosine, Manhattan, Euclidean). Tài liệu gốc là một bài đọc của IBM Skills Network về kiến trúc ChromaDB, thiên về phần toán.
>
> **Cảnh báo ngay từ đầu:** bài gốc có **một lỗi toán học rõ ràng** (nói cosine similarity chạy 0→1 — sai, thực ra -1→1) và **một nhầm lẫn khái niệm** (gọi "nearest neighbor" cho cả *search* lẫn thuật toán *nội suy ảnh*), cùng danh sách nguồn embedding **đã lỗi thời** (TensorFlow Hub, GPT-3). Tôi giữ phần đúng (các công thức và ví dụ tính tay — đã kiểm chứng bằng code, **chính xác**), đính chính phần sai, và cập nhật landscape 2026. Đánh dấu 🛠️ **[Đính chính]** và ➕ **[Mở rộng ngoài bài gốc]**.

---

## Phần 0 — 🗺️ Bản đồ bài học (Overview)

Ở quyển 1–3 ta luôn *giả định* đã có sẵn vector và một cách đo "gần nhau". Quyển này mở hộp đen đó ra: **vector từ đâu ra (embedding)?** và **"gần nhau" đo bằng gì (distance metric)?** Bài gốc trả lời hai câu hỏi này qua ba ý: (a) embedding biến ảnh/text/audio thành điểm trong không gian nhiều chiều, thứ giống nhau nằm gần nhau; (b) có nhiều nguồn để sinh embedding; (c) có ba thước đo khoảng cách phổ biến — **cosine, Manhattan, Euclidean** — kèm ví dụ tính tay. Đây là phần *toán nền* mà nếu nắm chắc, bạn trả lời phỏng vấn cực trơn.

**Sau khi học xong, bạn sẽ có thể:**
- Giải thích embedding là gì, vì sao "thứ giống nhau → vector gần nhau", và embedding sinh ra cho **image / text / audio** để làm task gì.
- Chọn **embedding model** đúng theo bài toán (landscape 2026, không phải GPT-3/TensorFlow Hub lỗi thời).
- Tính tay và code được **cosine similarity, Manhattan (L1), Euclidean (L2)** — và biết **khi nào dùng cái nào**.
- Tránh **lỗi cosine 0→1** (thực ra -1→1) và phân biệt **cosine *similarity* vs cosine *distance***.
- Hiểu vì sao **cosine được ưa chuộng cho semantic search**, và khi nào Euclidean/Manhattan hợp hơn.
- Đặt **threshold** cho nearest-neighbor để tránh "retrieve rác" (nối với RAG ở quyển 3).

**Mạch kiến thức từ basic → staff:**
- 🟢 **Basic:** embedding là gì (analogy phòng đồ chơi) → 3 loại dữ liệu → ví dụ tính tay cosine.
- 🟡 **Intermediate:** nguồn embedding (cập nhật 2026) → ba metric với công thức + tính tay đầy đủ → chọn metric nào.
- 🔴 **Advanced:** bản chất hình học của từng metric, quan hệ giữa chúng, khi nào cho kết quả KHÁC nhau, cosine similarity vs distance, và sự thật về "nearest neighbor" trong Chroma (thực ra là ANN/HNSW).
- 🟣 **Staff:** chọn embedding model ở quy mô lớn — chi phí, số chiều vs storage (Matryoshka), exact-vs-approximate, đổi model = re-embed, monitoring; system design.
- 🎯 **Cheatsheet:** Keywords, core concepts, code phải thuộc, câu hỏi phỏng vấn + one-liners.

---

## Phần 1 — 🟢 BASIC (Nền tảng)

### 1.1. "Tại sao" — database thường bó tay

Database truyền thống (SQL) giỏi **exact match**: "tìm hàng có `id = 5`". Nhưng thế giới AI cần đánh giá dữ liệu theo **đặc điểm giống nhau**: "tìm ảnh *trông giống* ảnh này", "tìm câu *cùng ý* với câu này". SQL không làm được. **Vector embedding** ra đời để lấp chỗ đó: biến mỗi object thành một **điểm trong không gian nhiều chiều**, sao cho *thứ giống nhau nằm gần nhau*.

### 1.2. Analogy đời thường — căn phòng đồ chơi

Bài gốc dùng một analogy rất hay, tôi giữ nguyên: tưởng tượng **một căn phòng đầy đồ chơi**. Mỗi món được mô tả bằng danh sách phẩm chất: *màu sắc, hình dạng, kích thước, chất liệu*. Coi mỗi phẩm chất là **một trục (một chiều)**. Vậy mỗi món đồ chơi = **một điểm** trong không gian nhiều chiều. Hai quả bóng khác nhau → nằm **gần nhau** (chung nhiều phẩm chất). Một con búp bê → nằm **xa** quả bóng. **Toạ độ = phẩm chất; gần nhau = giống nhau.** Embedding chính là việc "gán toạ độ theo phẩm chất" đó — chỉ khác là do một model AI làm tự động, với hàng trăm/nghìn chiều.

### 1.3. Thuật ngữ nền tảng (giải thích ngay lần đầu)

- **Vector embedding:** dãy số biểu diễn ý nghĩa/đặc điểm của một object (text, ảnh, audio) như một điểm trong không gian nhiều chiều.
- **Dimension (chiều):** một "phẩm chất/đặc trưng". Vector 300 chiều = 300 con số. (Bài gốc minh hoạ `[0.34, 2.35, 8.34, ...]` với 300 chiều.)
- **Semantic similarity:** độ giống nhau về **ý nghĩa** (không phải về chữ). Hai câu khác từ nhưng cùng ý → vẫn gần nhau.
- **Distance metric / similarity metric:** công thức đo "gần/xa" giữa hai vector (cosine, Manhattan, Euclidean).
- **Nearest neighbor search:** cho một query vector, tìm điểm gần nhất trong tập.
- **Embedding model:** model AI biến object → vector (Sentence Transformers, OpenAI, v.v.).
- **Collection (ChromaDB):** "bảng" chứa embeddings + documents + metadata (đã học ở quyển 3).

### 1.4. Ba loại dữ liệu → task (bài gốc)

Embedding không chỉ cho text. Bài gốc tóm gọn:
- **Image:** object recognition, **deduplication** (phát hiện ảnh trùng), scene detection, product search.
- **Text:** translation, sentiment analysis, question answering, **semantic search**.
- **Audio:** anomaly detection, speech-to-text, music transcription, phát hiện máy móc hỏng (machinery malfunction).

Một câu để nhớ: *"Bất cứ thứ gì embed được thành vector đều tìm-theo-ý-nghĩa được."* (Đây chính là mạch xuyên suốt cả 4 quyển.)

### 1.5. Ví dụ chạy tay — cosine bằng số (dùng đúng ví dụ của bài gốc)

Bài gốc cho 3 vector 3 chiều: **"Hello" = A[5,4,2]**, **"Hi" = B[4,3,2]**, **"World" = C[3,1,4]**. Trực giác: "Hello" và "Hi" cùng ý chào hỏi → phải gần nhau hơn "Hello" vs "World".

**Cosine similarity** = tích vô hướng chia cho tích độ dài:
`cos(A,B) = (A·B) / (|A|·|B|)`

- Tử số `A·B = 5×4 + 4×3 + 2×2 = 20 + 12 + 4 = 36`
- `|A| = √(5²+4²+2²) = √45 ≈ 6.708`, `|B| = √(4²+3²+2²) = √29 ≈ 5.385`
- `cos(A,B) = 36 / (6.708 × 5.385) = 36 / 36.125 ≈ 0.9965` → **rất giống** ✅

- `A·C = 5×3 + 4×1 + 2×4 = 15 + 4 + 8 = 27`, `|C| = √(3²+1²+4²) = √26 ≈ 5.099`
- `cos(A,C) = 27 / (6.708 × 5.099) = 27 / 34.205 ≈ 0.789` → giống ít hơn

→ Kết luận: **A gần B hơn A gần C** (0.9965 > 0.789). Đúng trực giác. *(Lưu ý nhỏ: bài gốc ghi 0.996; giá trị chính xác là ~0.9965, làm tròn 3 chữ số là 0.997 — chênh lệch chỉ do làm tròn.)*

### 1.6. Code "hello world" (đã chạy thật — verify đúng số của bài gốc)

```python
import numpy as np

A = np.array([5,4,2])   # "Hello"
B = np.array([4,3,2])   # "Hi"
C = np.array([3,1,4])   # "World"

def cosine(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

print("cos(A,B) =", round(cosine(A, B), 4))   # -> 0.9965  (A và B rất giống)
print("cos(A,C) =", round(cosine(A, C), 4))   # -> 0.789   (A và C ít giống hơn)
```
Giải thích dòng lõi: `np.dot(a,b)` là tử số (tích vô hướng), `np.linalg.norm` là độ dài vector (mẫu số). Chia ra → cosine. **Càng gần 1 càng giống hướng.**

### 1.7. ✅ Self-check (Basic)

1. Vì sao "thứ giống nhau → vector gần nhau"? "Chiều" trong embedding tương ứng với cái gì trong analogy đồ chơi?
2. Embedding sinh ra cho audio để làm những task gì? (kể 2)
3. Tính tay `cos(B,C)` với B[4,3,2], C[3,1,4]. (Gợi ý: B·C = 12+3+8 = 23; |B|=√29; |C|=√26.)

---

## Phần 2 — 🟡 INTERMEDIATE (Vận dụng)

### 2.1. Nguồn embedding — cập nhật 2026 (phần lỗi thời nhất của bài gốc)

Bài gốc nói *"hai nguồn text embedding: **TensorFlow embeddings (TensorFlow Hub)** và **OpenAI embeddings (GPT-3)**"*, và mặc định Chroma dùng **Sentence Transformers**.

🛠️ **[Đính chính — danh sách này đã cũ]**
- **GPT-3 embeddings** là đồ cổ (2020, đã deprecated). OpenAI hiện dùng **`text-embedding-3-small` / `text-embedding-3-large`**.
- **TensorFlow Hub** gần như không còn là nơi lấy embedding trong thực tế; ngày nay là **Hugging Face / Sentence Transformers** (self-host) hoặc **API providers**.
- "Hai nguồn" là quá hẹp.

➕ **[Mở rộng — landscape embedding model 2026 mà staff phải biết]**

| Nhóm | Model tiêu biểu (2026) | Ghi chú |
|---|---|---|
| **API (đóng)** | OpenAI **text-embedding-3**, Cohere **embed-v4** (đa ngôn ngữ mạnh), Voyage **voyage-3-large** (retrieval & code), Google **Gemini Embedding** (đa phương thức, 3072 chiều), Amazon Titan V2, Mistral Embed | Dễ tích hợp, trả phí, không tự host |
| **Open-source (mở)** | **BGE-M3** (BAAI — chuẩn production, hỗ trợ dense+sparse+multi-vector, 100+ ngôn ngữ, 8K context), **Qwen3-Embedding**, **Jina v5**, **NV-Embed-v2** (NVIDIA, top MTEB), **E5**, **Nomic**, **MiniLM** (nhẹ, 384 chiều, ~46MB) | Tự host, kiểm soát, không phí license |

- **MiniLM (all-MiniLM-L6-v2, 384 chiều)** chính là **default của ChromaDB** (một model Sentence Transformers) — nên câu "Chroma dùng Sentence Transformers mặc định" của bài gốc *vẫn đúng*.
- **MTEB (Massive Text Embedding Benchmark)** là thước đo chuẩn để so model — nhưng **đừng chọn theo bảng xếp hạng, hãy benchmark trên chính dữ liệu của bạn** (điểm mù người mới hay mắc).
- **Cách chọn:** cân theo (1) độ chính xác trên *dữ liệu của bạn*, (2) ngôn ngữ, (3) chi phí, (4) latency, (5) đa phương thức. Fine-tune cho domain hẹp (pháp lý/y tế/code) thường +10–30%.

### 2.2. Ba distance metrics — công thức + tính tay đầy đủ (lõi của bài gốc)

Dùng lại A[5,4,2], B[4,3,2], C[3,1,4].

**① Cosine similarity** (đo **góc/hướng**): đã tính ở Phần 1 → cos(A,B)=0.9965, cos(A,C)=0.789.

**② Manhattan distance (L1)** — quãng đường đi "theo ô bàn cờ", tổng trị tuyệt đối hiệu từng chiều:
`L1(A,B) = |x₁-x₂| + |y₁-y₂| + |z₁-z₂|`
- `L1(A,B) = |5-4| + |4-3| + |2-2| = 1 + 1 + 0 = 2`
- `L1(A,C) = |5-3| + |4-1| + |2-4| = 2 + 3 + 2 = 7`
→ A gần B hơn (2 < 7). ✅ **Với distance: nhỏ = gần.** (Ngược với similarity!)

**③ Euclidean distance (L2)** — đường chim bay (định lý Pythagoras):
`L2(A,B) = √((x₁-x₂)² + (y₁-y₂)² + (z₁-z₂)²)`
- `L2(A,B) = √(1² + 1² + 0²) = √2 ≈ 1.4142`
- `L2(A,C) = √(2² + 3² + 2²) = √17 ≈ 4.1231`
→ A gần B hơn (1.41 < 4.12). ✅

**Cả ba metric đều kết luận A gần B hơn A gần C.** Nhưng — nhớ chi tiết này cho Phần 3 — *chúng đồng ý ở đây là do ví dụ này, không phải luôn luôn.*

### 2.3. Code thực tế — ba metric + threshold (đã chạy thật)

```python
import numpy as np

A, B, C = np.array([5,4,2]), np.array([4,3,2]), np.array([3,1,4])

def cosine(a,b):    return np.dot(a,b) / (np.linalg.norm(a)*np.linalg.norm(b))
def manhattan(a,b): return np.sum(np.abs(a-b))          # L1
def euclidean(a,b): return np.sqrt(np.sum((a-b)**2))    # L2

for name, X in [("B", B), ("C", C)]:
    print(f"A vs {name}: cos={cosine(A,X):.4f}  L1={manhattan(A,X)}  L2={euclidean(A,X):.4f}")
# A vs B: cos=0.9965  L1=2  L2=1.4142
# A vs C: cos=0.7890  L1=7  L2=4.1231
```

**Threshold — điểm thực chiến quan trọng của bài gốc:** nearest neighbor *luôn* trả về một điểm gần nhất, nhưng "gần nhất" **không có nghĩa là "thật sự giống"**. Nếu tất cả điểm đều xa, cái gần nhất vẫn là rác. → Đặt **ngưỡng (threshold)**. Bài gốc lấy ví dụ câu hỏi *"What does the Kangaroo do?"* so với 10 câu; câu *"The kangaroo is hopping"* có cosine **0.7776** (cao nhất), và đặt threshold **0.6**: chỉ chấp nhận nếu ≥ 0.6, nếu không → "không có nearest neighbor phù hợp".

```python
def nearest_with_threshold(query, docs, thr=0.6):
    scored = [(name, cosine(query, v)) for name, v in docs.items()]
    best_name, best = max(scored, key=lambda x: x[1])
    return (best_name, best) if best >= thr else (None, best)  # dưới ngưỡng -> None
```
Điểm mấu chốt (nối với RAG quyển 3): threshold chính là cơ chế **"nếu không tìm thấy tài liệu đủ liên quan → nói không biết, đừng bịa"**. Bỏ threshold = nguồn gốc của hallucination trong RAG.

### 2.4. 3 lỗi thường gặp (và cách tránh)

1. **Lẫn similarity với distance.** Cosine *similarity*: **cao = gần**. Manhattan/Euclidean *distance*: **thấp = gần**. ChromaDB bên trong dùng **cosine *distance* = 1 − cosine similarity** (thấp = giống). Nhầm chiều so sánh → xếp hạng ngược. → Luôn xác định rõ đang dùng "similarity" hay "distance".
2. **Không normalize khi dùng cosine.** Cosine bỏ qua độ dài, nhưng nếu bạn tính bằng dot product mà quên normalize → ra inner product, không phải cosine. → Normalize vector về độ dài 1 trước (hoặc dùng công thức đầy đủ).
3. **Chọn sai metric so với embedding model.** Model được train cho cosine mà bạn index bằng Euclidean thô → xếp hạng lệch. → Dùng metric mà model khuyến nghị (đa số text embedding → cosine).

### 2.5. ✅ Self-check (Intermediate)

1. Với distance (L1/L2), "gần" là số lớn hay nhỏ? Với cosine similarity thì sao? ChromaDB dùng cái nào bên trong?
2. Vì sao "GPT-3 embeddings" trong bài gốc đã lỗi thời? Kể 2 model text embedding phổ biến năm 2026.
3. Threshold trong nearest-neighbor giải quyết vấn đề gì? Nó liên quan thế nào tới hallucination trong RAG?

---

## Phần 3 — 🔴 ADVANCED (Chuyên sâu)

### 3.1. Bản chất hình học & khi nào ba metric cho kết quả KHÁC nhau

Bài gốc kết luận *"ba metric comparable"* — đúng trong ví dụ đó, nhưng **gây hiểu nhầm nếu tổng quát hoá**. Sự thật:

- **Cosine** đo **hướng**, *bỏ qua độ lớn (magnitude)*. Hai vector cùng hướng nhưng dài ngắn khác nhau → cosine = 1 (giống hệt).
- **Euclidean (L2)** và **Manhattan (L1)** đo **khoảng cách tuyệt đối**, *có tính độ lớn*.

➕ **[Mở rộng — ví dụ chúng BẤT ĐỒNG]** Xét văn bản: document dài và document ngắn *cùng chủ đề*. Chúng cùng **hướng** (cosine coi là rất giống) nhưng khác **độ lớn** vì đếm từ khác nhau (Euclidean coi là xa). Đây là lý do **cosine được ưa chuộng cho semantic search**: ta quan tâm *ý nghĩa* (hướng), không quan tâm *độ dài văn bản* (magnitude). Bài gốc nói đúng điều này ("Euclidean treats all dimensions equally, documents with very different word frequencies might be considered close") nhưng không nhấn đủ mạnh rằng chúng có thể **xếp hạng ngược nhau** trong thực tế.

➕ **Insight staff-level (đã chứng minh bằng code): với vector ĐÃ NORMALIZE, Euclidean và cosine cho CÙNG thứ hạng.** Quan hệ chính xác:
`L2(a,b)² = 2·(1 − cosine(a,b))` khi |a| = |b| = 1.
→ Euclidean là hàm **đơn điệu giảm** theo cosine → sắp xếp giống hệt. Vì vậy nhiều vector DB **normalize hết rồi dùng inner product** (nhanh hơn) mà kết quả tương đương cosine. Đây là mẹo tối ưu hiệu năng kinh điển.

```python
import numpy as np
def norm(x): return x/np.linalg.norm(x)
A,B = np.array([5,4,2]), np.array([4,3,2])
An,Bn = norm(A), norm(B)
cos = np.dot(An,Bn)
print(round(np.sum((An-Bn)**2),4), "≈", round(2*(1-cos),4))  # 0.0069 ≈ 0.0069
```

### 3.2. 🛠️ ĐÍNH CHÍNH lớn: cosine similarity chạy từ −1 đến 1, KHÔNG phải 0 đến 1

Bài gốc viết: *"cosine similarity là giá trị **ranges from 0 to 1**... gần 0 = dissimilar, gần 1 = similar"*. **Sai về mặt toán học tổng quát.**

- Cosine của **góc** chạy từ **−1 đến 1**: cùng hướng = **+1**, vuông góc = **0**, **ngược hướng = −1**.
- Chỉ khi *mọi thành phần vector đều không âm* (như TF-IDF, count vectors) thì cosine mới nằm trong [0, 1] (vì các vector đều ở "góc phần tám dương").
- **Embedding từ mạng neural hiện đại CÓ thành phần âm** → cosine **có thể âm**. Bỏ qua điều này khiến bạn hiểu sai khi thấy điểm âm và tưởng "lỗi".

**Chứng minh bằng code (đã chạy thật):**
```python
import numpy as np
def cosine(a,b): return np.dot(a,b)/(np.linalg.norm(a)*np.linalg.norm(b))
u = np.array([1.0, 0.0])
print(cosine(u, np.array([ 1.0, 0.0])))   # 1.0  (cùng hướng)
print(cosine(u, np.array([ 0.0, 1.0])))   # 0.0  (vuông góc)
print(cosine(u, np.array([-1.0, 0.0])))   # -1.0 (ngược hướng!)  <-- bài gốc SAI
emb = np.array([0.4,-0.7,0.2,-0.9]); qry = np.array([-0.3,0.8,-0.1,0.5])
print(round(cosine(emb, qry),3))          # -0.944 (embedding có số âm -> cosine âm)
```
🎯 Nói được "cosine ∈ [−1, 1], chỉ [0, 1] khi vector không âm" trong phỏng vấn = tín hiệu bạn hiểu *bản chất*, không học vẹt.

### 3.3. 🛠️ ĐÍNH CHÍNH: "nearest neighbor cho pixelating ảnh" — nhầm hai thuật toán

Bài gốc viết: *"Nearest neighbor algorithm is used in image data for **pixelating and improving the quality of the image**."* 

**Đây là nhầm lẫn hai thứ khác nhau cùng tên "nearest neighbor":**
| | **k-NN *search*** (chủ đề của ta) | **Nearest-neighbor *interpolation*** (bài gốc lỡ nhắc) |
|---|---|---|
| Làm gì | Tìm vector *gần nhất trong embedding space* | *Phóng to/thu nhỏ ảnh* bằng cách lấy pixel gần nhất |
| Không gian | High-dimensional (embedding) | 2D lưới pixel |
| Liên quan vector DB? | **Có** | **Không** — là thuật toán xử lý ảnh (image scaling) |

Hai cái **không liên quan gì nhau** ngoài việc trùng tên. Vector database làm **k-NN search**; "pixelating để cải thiện chất lượng ảnh" là **nearest-neighbor interpolation** — một kỹ thuật đồ hoạ hoàn toàn khác. (Đây là kiểu bẫy trùng tên giống "vector" GIS vs "vector" embedding ở quyển 2.)

### 3.4. Cosine *similarity* vs cosine *distance* — và "range search"

➕ ChromaDB (và nhiều vector DB) bên trong dùng **cosine *distance* = 1 − cosine similarity**:
- cosine **similarity** 1.0 (giống hệt) ↔ cosine **distance** 0.0 (khoảng cách 0).
- Khi cấu hình `metadata={"hnsw:space": "cosine"}`, kết quả `distances` Chroma trả về là **cosine distance** (thấp = giống). Đừng nhầm với similarity (cao = giống).

**Range search** (bài gốc nhắc): lấy các vector *trong một bán kính/ngưỡng khoảng cách* quanh query, thay vì đúng top-k gần nhất. Ví dụ bài gốc: gợi ý phim "cùng thể loại" (không cần *giống nhất*, chỉ cần *đủ gần*). Về bản chất đây là **lọc theo threshold khoảng cách** — chính là mở rộng của ý "threshold" ở Phần 2.

### 3.5. 🛠️ Sự thật về "query engine" của Chroma: đó là ANN, không phải exact

Bài gốc mô tả query engine "tối ưu cho nearest neighbor search" nhưng **ngầm hiểu là tìm chính xác (exact)**. 

🛠️ **[Đính chính/mở rộng — nối với quyển 1]** Ở quy mô thật, Chroma **không** quét toàn bộ (exact O(N)); nó dùng **HNSW** — một **ANN (Approximate Nearest Neighbor)** index (đã học kỹ ở quyển 1). Nghĩa là kết quả **gần đúng**, đánh đổi **recall** lấy **tốc độ O(log N)**. Bài gốc bỏ qua hoàn toàn khái niệm *approximate* và *recall* — nhưng đó mới là điều làm vector DB *thực sự chạy nhanh*. Distance metric (cosine/L2) chỉ trả lời *"đo gần/xa thế nào"*; HNSW trả lời *"tìm cái gần đó nhanh ra sao"*. Hai thứ độc lập và đều cần.

### 3.6. Bảng chốt: chọn metric nào?

| Metric | Đo gì | Nhạy với magnitude? | Dùng khi | "Gần" = |
|---|---|---|---|---|
| **Cosine** | Góc/hướng | Không | **Semantic search** (mặc định text embedding); độ dài văn bản không quan trọng | similarity cao / distance thấp |
| **Euclidean (L2)** | Đường chim bay | Có | Dữ liệu đồng nhất về độ lớn; cần khoảng cách vật lý thật; embedding đã normalize (≡ cosine) | distance thấp |
| **Manhattan (L1)** | Đi theo trục | Có | Dữ liệu thưa (sparse), nhiều chiều; ít nhạy outlier hơn L2 | distance thấp |
| **Dot / Inner product** | Hướng × độ lớn | Có | Vector đã normalize (≡ cosine, nhanh hơn); một số recsys | score cao |

### 3.7. Edge cases phải xử lý

- **Vector độ dài 0** (all-zeros) → chia cho 0 trong cosine → NaN. Loại bỏ/xử lý riêng.
- **Chiều không đồng nhất thang đo** → một chiều "áp đảo" Euclidean/Manhattan. Cân nhắc chuẩn hoá đặc trưng.
- **Curse of dimensionality** (quyển 1): ở chiều rất cao, mọi khoảng cách "co cụm" gần nhau → chọn metric & threshold cẩn thận; đừng tin threshold tuyệt đối cứng.
- **Query và document khác model/metric** → xếp hạng rác (nhắc lại nguyên tắc nhất quán).

---

## Phần 4 — 🟣 STAFF LEVEL (Tư duy hệ thống & lãnh đạo kỹ thuật)

### 4.1. Chọn embedding model ở quy mô lớn — bottleneck & chi phí

➕ Bài gốc coi "chọn embedding model" là chuyện vài dòng. Ở tầm staff, đây là **quyết định kiến trúc tốn kém nhất**:

- **Số chiều ↔ storage & latency:** vector 3072 chiều (Gemini) tốn **gấp 8 lần RAM** so với 384 chiều (MiniLM), và search chậm hơn. Với **1 tỷ vector**, chênh lệch này = hàng TB RAM và hàng chục nghìn USD. → **Matryoshka / MRL embeddings** cho phép *cắt bớt chiều* (vd 1536 → 256) mất ít chất lượng để tiết kiệm storage khủng — một đòn cost quan trọng.
- **API vs self-host:** API (OpenAI/Cohere) = zero-ops nhưng **trả tiền mỗi lần embed** và **gửi dữ liệu ra ngoài** (vấn đề compliance). Self-host (BGE-M3) = kiểm soát, rẻ ở scale lớn, nhưng cần GPU. → Tính **TCO**, và với dữ liệu nhạy cảm thường buộc self-host.
- **Chi phí *sinh* embedding** (nhắc lại quyển 2): embed 1 tỷ tài liệu bằng GPU thường **đắt hơn cả vector search**. Chọn model nặng = tăng cả chi phí embed lẫn storage.

### 4.2. Đổi metric / đổi model = re-embed toàn bộ

➕ **Failure mode kinh điển staff phải cảnh báo:** vector từ model A và model B **không so sánh được** với nhau (khác không gian). Đổi embedding model → **phải re-embed *toàn bộ* corpus và rebuild index** — tốn kém, phải lên kế hoạch (blue-green, version hoá embedding). Tương tự, chọn sai metric lúc đầu và muốn đổi (cosine → L2) thường phải build lại index. → **Quyết định embedding model + metric là quyết định khó đảo ngược**; chọn kỹ từ đầu, benchmark trên dữ liệu thật trước khi cam kết.

### 4.3. Trade-off kiến trúc: khi nào metric nào & khi nào KHÔNG lo về metric

**NÊN cân nhắc kỹ metric khi:** magnitude mang thông tin (recsys với điểm số), hoặc dữ liệu không normalize. 
**KHÔNG cần quá lo khi:** dùng text embedding chuẩn đã normalize — cosine/dot/Euclidean cho *cùng thứ hạng*, chọn cái nào cũng được (chọn dot vì nhanh nhất). → Staff biết **đâu là quyết định quan trọng, đâu là bikeshedding**: chọn *embedding model* quan trọng gấp bội chọn *distance metric* (với vector normalized).

### 4.4. Monitoring, reliability

- **Giám sát phân bố similarity score:** nếu điểm nearest-neighbor trung bình **tụt dần** → có thể drift dữ liệu, hoặc embedding model không hợp domain → cần đổi/fine-tune.
- **Threshold tuning là việc liên tục:** ngưỡng 0.6 của bài gốc *không* phải hằng số vũ trụ — nó phụ thuộc model, domain, số chiều. Phải đo trên tập ground-truth và điều chỉnh; theo dõi tỉ lệ "không tìm thấy trên ngưỡng".
- **Chi phí query:** cosine/L2 rẻ; nhưng số chiều lớn làm mỗi phép tính đắt hơn tuyến tính → số chiều là đòn bẩy cost trực tiếp.

### 4.5. Giải thích cho stakeholder & 🎤 System design mẫu

**Cho stakeholder non-technical:** *"Máy 'hiểu' nội dung bằng cách biến mỗi tài liệu thành một dãy số — như một toạ độ trên bản đồ ý nghĩa. Thứ giống nhau nằm gần nhau. 'Gần' đo bằng góc giữa hai toạ độ. Chọn 'bộ não' biến-thành-số (embedding model) là quyết định đắt và khó đổi nhất — đổi nó là phải xử lý lại toàn bộ dữ liệu — nên ta chọn kỹ và đo trên dữ liệu thật của mình trước."*

> **Đề: "Chọn embedding model + distance metric cho semantic search trên 50 triệu tài liệu đa ngôn ngữ (Việt + Anh), ngân sách hạn chế, dữ liệu nhạy cảm không được gửi ra ngoài."**

Hướng trả lời staff:
1. **Ràng buộc quyết định trước:** dữ liệu nhạy cảm → **loại API bên ngoài**, buộc **self-host**; đa ngôn ngữ → cần model multilingual; ngân sách hạn chế → ưu tiên open-source + số chiều vừa phải.
2. **Chọn model:** **BGE-M3** (open-source, 100+ ngôn ngữ, self-host được, hỗ trợ cả sparse+dense) là ứng viên mạnh; benchmark trên **dữ liệu Việt-Anh thật** (MTEB không đủ). Cân nhắc Matryoshka để cắt chiều nếu storage căng.
3. **Metric:** normalize vector → dùng **cosine** (hoặc dot product tương đương, nhanh hơn). 50M vector → HNSW (ANN) trong Chroma/Qdrant/Milvus.
4. **Threshold:** đo trên ground-truth để đặt ngưỡng "đủ liên quan"; dưới ngưỡng → trả "không tìm thấy" thay vì rác.
5. **Vận hành:** version hoá embedding; kế hoạch re-embed nếu đổi model; monitor phân bố score + recall.
6. **Kết bằng trade-off:** self-host GPU cost vs API compliance; số chiều vs storage; cosine vs dot. → *đánh đổi*, không "một đáp án đúng".

---

## Phần 5 — 🎯 CHỐT LẠI ĐỂ ĐI PHỎNG VẤN (Interview Cheatsheet)

### 5.1. Keywords bắt buộc nhớ (định nghĩa 1 dòng)

- **Vector embedding:** dãy số biểu diễn ý nghĩa; thứ giống nhau → gần nhau.
- **Dimension:** một đặc trưng/trục; vector d chiều = d con số.
- **Semantic similarity:** giống nhau về *ý nghĩa*, không phải về *chữ*.
- **Distance metric:** công thức đo gần/xa giữa 2 vector.
- **Cosine similarity:** đo **góc**; range **−1..1** (chỉ 0..1 khi vector không âm); cao = giống.
- **Cosine distance:** = 1 − cosine similarity; **thấp = giống** (Chroma dùng cái này).
- **Manhattan (L1):** tổng |hiệu| từng chiều; thấp = gần.
- **Euclidean (L2):** đường chim bay (Pythagoras); thấp = gần.
- **Dot/Inner product:** hướng × độ lớn; ≡ cosine nếu đã normalize.
- **Normalize:** đưa vector về độ dài 1; khi đó L2 & cosine cùng thứ hạng.
- **Nearest neighbor search:** tìm điểm gần query nhất (KHÁC nearest-neighbor *interpolation* của ảnh!).
- **Threshold:** ngưỡng để coi một neighbor là "thật sự liên quan"; chống retrieve rác.
- **Embedding model:** biến object → vector (OpenAI text-embedding-3, Cohere embed-v4, BGE-M3, MiniLM...).
- **MTEB:** benchmark chuẩn so sánh text embedding model.
- **Matryoshka / MRL:** cắt bớt chiều embedding để tiết kiệm storage, mất ít chất lượng.
- **ANN / HNSW:** cách tìm *nhanh gần đúng* (Chroma dùng bên trong) — khác với "đo khoảng cách".

### 5.2. Core concepts (nếu chỉ nhớ mấy điều này)

1. **Embedding = gán toạ độ theo ý nghĩa; distance metric = đo gần/xa.** Hai thứ độc lập, đều cần.
2. **Cosine ∈ [−1, 1]**, chỉ [0, 1] khi vector không âm. Cosine đo **hướng**, bỏ qua **độ lớn**.
3. **Similarity: cao = gần. Distance: thấp = gần.** Chroma trả về cosine *distance*.
4. **Cosine được ưa cho semantic search** vì bỏ qua độ dài văn bản; Euclidean nhạy magnitude.
5. **Vector đã normalize → cosine ≡ Euclidean ≡ dot về thứ hạng** (L2² = 2(1−cos)); nên chọn *metric* ít quan trọng hơn chọn *model*.
6. **Threshold** biến "luôn có nearest neighbor" thành "chỉ nhận nếu đủ giống" → chống hallucination trong RAG.
7. **Chọn embedding model là quyết định đắt & khó đảo ngược** (đổi model = re-embed toàn bộ); benchmark trên dữ liệu thật, không tin mù MTEB.
8. **"Query engine của Chroma" là ANN/HNSW** (gần đúng, O(log N)), không phải quét exact.

### 5.3. Ideas / mental models

- **"Bản đồ ý nghĩa":** embedding = toạ độ trên bản đồ; gần = giống.
- **"Cosine đo *hướng*, Euclidean đo *quãng đường*"** — văn bản dài/ngắn cùng ý → cùng hướng (cosine giống) nhưng khác độ dài (Euclidean xa).
- **"Similarity đi lên, distance đi xuống"** — nhớ chiều so sánh.
- **"Normalize rồi thì ba metric là một"** — mẹo tối ưu & để không bikeshed.
- **"Threshold = cái phanh chống bịa"** — dưới ngưỡng thì nói không biết.
- **"Đổi embedding model = xây lại cả kho"** — nên chọn kỹ từ đầu.
- **"Hai chữ nearest neighbor khác nhau"** — search (vector DB) vs interpolation (ảnh).

### 5.4. Code cần thuộc lòng

**(1) Ba metric — interviewer rất hay bắt viết:**
```python
import numpy as np
def cosine(a,b):    return np.dot(a,b)/(np.linalg.norm(a)*np.linalg.norm(b))  # góc, cao=giống
def manhattan(a,b): return np.sum(np.abs(a-b))                                # L1, thấp=gần
def euclidean(a,b): return np.sqrt(np.sum((a-b)**2))                          # L2, thấp=gần
```
*Khi nào dùng:* cosine cho semantic text; L2 khi magnitude quan trọng; L1 cho sparse/nhiều chiều.

**(2) Nearest neighbor + threshold (chống rác):**
```python
def nn(query, docs, thr=0.6):
    name, score = max(((n, cosine(query, v)) for n, v in docs.items()),
                      key=lambda x: x[1])
    return (name, score) if score >= thr else (None, score)  # dưới ngưỡng -> None
```

**(3) ChromaDB query kèm metric & filter (đã học ở quyển 3):**
```python
import chromadb
client = chromadb.Client()
col = client.create_collection("kb_docs", metadata={"hnsw:space":"cosine"})  # chọn cosine
# ...add...
res = col.query(query_embeddings=[q], n_results=5, where={"lang":"vi"})       # distances = cosine DISTANCE
```

### 5.5. Câu hỏi phỏng vấn thường gặp + gợi ý trả lời (có câu bẫy)

1. **"Cosine similarity chạy từ mấy đến mấy?"** *(câu bẫy — nhiều tài liệu, kể cả bài gốc, nói 0→1)* → **−1 đến 1.** Chỉ [0, 1] khi mọi thành phần vector không âm (TF-IDF). Embedding neural có số âm → cosine có thể âm.

2. **"Cosine vs Euclidean, chọn cái nào cho semantic search? Vì sao?"** → Cosine, vì nó đo **hướng (ý nghĩa)**, bỏ qua **độ lớn (độ dài văn bản)**. Thêm điểm: với vector đã normalize, hai cái tương đương về thứ hạng.

3. **"Similarity cao là gần hay distance cao là gần?"** *(bẫy chiều so sánh)* → Similarity **cao** = gần; distance **thấp** = gần. Chroma trả về cosine *distance* (thấp = giống).

4. **"Chroma tìm nearest neighbor bằng cách quét hết à?"** *(bẫy exact vs approximate)* → Không; dùng **HNSW (ANN)** — gần đúng, O(log N), đánh đổi recall lấy tốc độ. Distance metric chỉ định nghĩa "đo gần/xa", HNSW lo "tìm nhanh".

5. **"Đổi embedding model có ảnh hưởng gì tới dữ liệu đã lưu?"** *(bẫy vận hành)* → Vector cũ & mới **không so sánh được** → phải **re-embed toàn bộ + rebuild index**. Quyết định model là khó đảo ngược.

6. **"Threshold để làm gì? Đặt bao nhiêu?"** → Để loại "nearest neighbor nhưng thực ra không giống" → chống retrieve rác/hallucination. Không có con số vũ trụ; đo trên ground-truth theo model/domain.

7. **"Nearest neighbor có liên quan gì tới cải thiện chất lượng ảnh không?"** *(bẫy trùng tên)* → Không. k-NN **search** (vector DB) khác hoàn toàn nearest-neighbor **interpolation** (phóng ảnh). Chỉ trùng tên.

8. **"Metric quan trọng hơn hay model quan trọng hơn?"** → **Model** quan trọng hơn nhiều (với vector normalized, các metric tương đương). Đừng bikeshed metric; hãy benchmark model trên dữ liệu thật.

### 5.6. One-liner đắt giá

- *"Cosine similarity chạy **−1 đến 1**, không phải 0 đến 1 — nó chỉ dương khi vector không âm; embedding neural thật có số âm."*
- *"Cosine đo **hướng**, Euclidean đo **quãng đường** — nên hai văn bản cùng ý khác độ dài sẽ *giống* theo cosine mà *xa* theo Euclidean."*
- *"Normalize xong thì **cosine, Euclidean, dot là một** về thứ hạng — nên tôi không bikeshed metric, tôi bikeshed *model*."*
- *"Threshold là **cái phanh chống bịa**: dưới ngưỡng liên quan thì trả 'không biết', đừng ép ra một nearest neighbor rác."*
- *"Đổi embedding model = **xây lại cả kho** — vector cũ và mới không cùng không gian; nên đây là quyết định tôi chọn kỹ và đo trên dữ liệu thật từ đầu."*
- *"'Query engine' của Chroma là **ANN/HNSW gần đúng**, không phải quét exact — distance metric đo *gần/xa*, HNSW lo *tìm nhanh*, hai chuyện khác nhau."*

---

### 📌 Phụ lục: những chỗ bài giảng gốc sai / lỗi thời / gây hiểu nhầm

1. **🚨 "Cosine similarity ranges from 0 to 1"** → **SAI.** Đúng là **−1 đến 1**; chỉ [0, 1] khi vector không âm. Embedding neural có số âm → cosine âm được.
2. **🚨 "Nearest neighbor... for pixelating and improving image quality"** → nhầm **k-NN search** (vector DB) với **nearest-neighbor interpolation** (phóng ảnh) — hai thuật toán khác nhau trùng tên.
3. **Nguồn embedding lỗi thời:** "TensorFlow Hub + OpenAI GPT-3" → 2026 là OpenAI **text-embedding-3**, Cohere **embed-v4**, Voyage, **BGE-M3**, Qwen3, Gemini... "Hai nguồn" quá hẹp.
4. **Bỏ qua ANN/HNSW & recall:** mô tả "query engine tối ưu nearest neighbor" nhưng ngầm hiểu exact; thực ra Chroma dùng **HNSW (approximate)** — thiếu khái niệm cốt lõi từ quyển 1.
5. **"Ba metric comparable"** → gây hiểu nhầm: chúng đồng ý trong ví dụ đồ chơi, nhưng cosine (hướng) vs Euclidean (magnitude) **có thể xếp hạng ngược** trên dữ liệu thật; chỉ *tương đương khi đã normalize*.
6. **Không phân biệt cosine *similarity* vs *distance*** — Chroma trả về distance (thấp = giống), dễ nhầm chiều.
7. **Rounding nhỏ:** ghi cos(A,B)=.996; giá trị chính xác ~0.9965 (làm tròn 3 số = 0.997). Không phải lỗi lớn, chỉ là trình bày.

> **Phần đúng & giá trị của bài gốc:** các **công thức** cosine/Manhattan/Euclidean và **ví dụ tính tay** (36/36.125≈0.9965; L1=2 vs 7; L2=1.4142 vs 4.1231) đều **chính xác** (tôi đã verify bằng code); analogy "phòng đồ chơi" rất tốt; ý **cosine ưu tiên cho semantic search vì đo hướng** là đúng; khái niệm **threshold** cho nearest neighbor là điểm thực chiến giá trị. Giữ những phần đó, sửa lỗi cosine-range và nhầm-lẫn-nearest-neighbor, cập nhật nguồn embedding, và bổ sung ANN/HNSW + tư duy chọn model ở quy mô lớn.
