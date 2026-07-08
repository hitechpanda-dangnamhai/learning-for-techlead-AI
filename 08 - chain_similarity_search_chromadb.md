# Chain gối đầu — Similarity Search trong ChromaDB (capstone)

> Nén theo chain gối đầu: đuôi node trước = đầu node sau. Thuật ngữ giữ nguyên tiếng Anh,
> giải thích tiếng Việt ngắn. Bảng so sánh / đính chính / tổng kết giữ dạng bảng.
> Bài gốc lỗi: code JS `n:3` (phải `nResults`) + gộp nhầm "5 loại similarity" + nhầm "distances"
> với "similarity scores" → phần đính chính giữ nguyên như mắt xích trong chain.

---

## MẠCH CHÍNH — similarity search là gì → cơ chế bên trong
similarity search là tìm item giống nhất với một query trong tập dữ liệu => đây là nghiệp vụ trung tâm của vector database (và của information retrieval) => nó khác exact match của SQL: tìm *gần giống*, không phải *trùng khớp* => item tham chiếu mà ta muốn tìm thứ giống nó gọi là query (một câu text, một ảnh, một vector) => nhưng độ "giống" phụ thuộc ngữ cảnh => giống về ý nghĩa gọi là semantic similarity => giống về chuỗi ký tự gọi là syntactic / string similarity => ChromaDB chuyên về semantic similarity qua embedding => bên trong, similarity search = distance metric + ANN index => distance metric (cosine/L2/ip) *đo* gần/xa giữa hai vector => ANN index (HNSW) *tìm nhanh* các vector gần nhất mà không quét hết N => nhờ HNSW độ phức tạp là O(log N) thay vì O(N) => nên kết quả là approximate (gần đúng), đánh đổi recall lấy tốc độ

## CHAIN — Giải phẫu 5 bước một query (rẽ)
mọi similarity search trong ChromaDB gồm 5 bước => bước 1 nhận query (text hoặc vector) => bước 2 embed query: nếu là text thì biến thành vector bằng đúng embedding function của collection (quyển 5) => bước 3 tìm k gần nhất: HNSW so query vector với các vector trong collection, dùng distance metric đã chọn => bước 4 lọc (tùy chọn): áp `where` filter theo metadata => bước 5 trả kết quả: ids, documents, metadatas, distances

## CHAIN — Ví dụ "coffee" → threshold (rẽ)
query "coffee" đo cosine distance tới từng doc (thấp = giống) => d3 "coffee beans..." 0.041, d1 "espresso coffee" 0.093, d4 "sports car" 0.954 => nên top-3 = [d3, d1, d4] => nhận xét: d4 (xe hơi) lọt top-3 dù hoàn toàn không liên quan => vì top-k LUÔN trả về đủ k cái, kể cả khi cái cuối rất xa => nên phải đặt threshold trên distance => chỉ nhận kết quả nếu đủ gần (vd d < 0.5 loại xe hơi 0.954) => threshold là cơ chế "nếu không đủ giống → đừng trả" => chống đưa rác vào prompt RAG (quyển 3) hay recommendation => ngưỡng phụ thuộc metric/model/domain nên phải đo trên ground-truth

## CHAIN — API query đúng 2026 (rẽ, đính chính)
bài gốc viết `collection.query({queryTexts:["coffee"], n:3})` => nhưng tên tham số `n` là SAI => đúng phải là `nResults` (JS) / `n_results` (Python) => query dạng text đưa qua `queryTexts` / `query_texts` (Chroma tự embed) => hoặc query dạng vector sẵn đưa qua `queryEmbeddings` / `query_embeddings` => lọc metadata bằng `where` => và nên bọc query trong try/catch để error handling (bài gốc đúng ý này) => lưu ý: `.query()` được gọi TRÊN một collection object => không truyền "tên collection" vào trong `query` => phải lấy object bằng `get_collection(name)` trước rồi mới gọi `.query()`

## CHAIN — Distance vs Similarity score (rẽ, đính chính)
bài gốc nói kết quả chứa "IDs, distances, similarity scores, metadata" => nhưng không chính xác => ChromaDB trả về ids, documents, metadatas, distances => tức chỉ có `distances`, KHÔNG có trường "similarity score" riêng => vì distance và similarity là nghịch đảo của nhau => với space="cosine", Chroma trả cosine *distance* (thấp = giống) => muốn "similarity" thì tự tính similarity = 1 − distance => đừng tưởng đó là hai con số riêng biệt

## CHAIN — Semantic vs syntactic: edit distance (rẽ, đính chính loại ③)
bài gốc gộp structured-data similarity vào cùng nhóm nhưng cơ chế khác hẳn => structured data trên chuỗi dùng edit distance (Levenshtein) => edit distance là số phép chèn/xóa/thay tối thiểu để biến chuỗi a → b => vd edit("kitten","sitting") = 3 => edit("coffee","tea") = 5 (rất xa về *chữ*) => nhưng về embedding, "coffee" và "tea" lại gần nhau (đều là đồ uống) => đây là điểm "aha": edit distance đo giống về chuỗi ký tự (syntactic), còn embedding đo giống về ý nghĩa (semantic) => hai loại "similarity" hoàn toàn khác nhau => ChromaDB làm semantic, KHÔNG làm edit distance => Big-O của edit distance là O(m·n) (quy hoạch động), khác hẳn O(log N) của HNSW

## CHAIN — Genomics: embedding thay alignment (rẽ, đính chính loại ⑤)
genomics tìm chuỗi DNA/RNA/protein giống nhau => truyền thống dùng sequence alignment => alignment cổ điển là BLAST và Smith-Waterman => chúng dựa trên weighted edit distance + ma trận thay thế (BLOSUM/PAM) => nên alignment cổ điển KHÔNG phải vector DB => hiện đại (2024–2026) dùng protein language model embeddings (ESM-2, ProtT5, Ankh) => rồi tính cosine similarity giữa các embedding => cách này thường thắng BLOSUM/BLAST cho "remote homology" (họ hàng xa) => và đây chính là vector similarity → vector DB (như ChromaDB) trở nên hữu ích => nên "genomics similarity" đang chuyển dịch từ alignment cổ điển sang embedding

## CHAIN — 3 lỗi thường gặp (rẽ độc lập)
lỗi 1: dùng `n` thay `nResults`/`n_results` → sai tên tham số → lỗi/không chạy => lỗi 2: tưởng ChromaDB làm được mọi "similarity" → nó chỉ làm embedding/vector similarity, KHÔNG làm edit distance hay sequence alignment cổ điển → string similarity phải dùng thư viện khác (Levenshtein) => lỗi 3: không đặt threshold → nhét rác vào kết quả (top-k luôn trả đủ k)

## CHAIN — Edge cases (rẽ độc lập)
top-k trả rác khi kho nhỏ / không có match thật → dùng threshold => query rỗng / ngoài domain → embedding vô nghĩa, recall thấp → cần fallback => metric không khớp embedding model → xếp hạng lệch => kết quả trùng lặp (near-duplicate) → giảm diversity → dedup hoặc rerank => nhầm distance với similarity → sắp xếp ngược (nhớ cosine distance gần 0 = giống)

## CHAIN — Pipeline nhiều tầng ở scale (rẽ, staff)
ở production, một similarity search tốt thường là pipeline nhiều tầng, hiếm khi một cú query đơn => tầng 1 là retrieval (thô): ChromaDB/HNSW trả top-k lớn (100–500), nhanh, recall cao => tầng 2 là re-ranking (tinh): cross-encoder chấm lại → top-k nhỏ chính xác => tầng 3 là hybrid search: kết hợp vector (semantic) + BM25/full-text (keyword) + metadata filter => ChromaDB bản mới hỗ trợ dense + sparse(BM25) + full-text + regex trong một interface => với tên riêng/mã/ID mà embedding hay bỏ sót, hybrid là bắt buộc

## CHAIN — Bottleneck & cost (rẽ, staff)
latency của similarity search cộng dồn embed query (API EF → mạng) + HNSW search + rerank => cache query nóng (semantic cache) giúp giảm nhiều => về cost: API embedding tốn tiền embed mỗi query, rerank cross-encoder tốn compute => nhưng ở RAG, LLM generation mới thống trị cost (quyển 3) => về scale kho: vài triệu vector thì Chroma OK, tỷ vector thì chuyển Milvus/Qdrant (Chroma không cho billion-scale)

## CHAIN — Khi nào KHÔNG dùng vector similarity (rẽ, staff)
KHÔNG dùng vector similarity khi cần match *chính xác chuỗi* (tên file, mã đơn, biển số) → dùng exact match / edit distance / full-text => vì "coffee" và "tea" gần về nghĩa nhưng đó không phải điều bạn muốn khi tìm mã sản phẩm => KHÔNG dùng khi bài toán là căn chỉnh chuỗi sinh học truyền thống → dùng BLAST/Smith-Waterman => KHÔNG dùng khi dữ liệu nhỏ, luật rõ → dùng SQL/filter => KHÔNG dùng khi cần giải thích chính xác vì sao match → embedding là hộp đen, string/rule minh bạch hơn

## CHAIN — Reliability & monitoring (rẽ, staff)
giám sát phân bố distance của top-1: nếu trung bình *tăng dần* → drift hoặc model không hợp domain => giám sát tỉ lệ query dưới threshold ("không tìm thấy"): tăng đột biến → dữ liệu thiếu hoặc embedding hỏng => giám sát recall@k trên ground-truth: search có thể *chạy* nhưng trả *sai* mà không báo lỗi (hỏng âm thầm) => failure modes: EF tải model fail (quyển 5), API embedding rate-limit, metric sai, near-duplicate làm lệch

## CHAIN — System design 50M tài liệu (rẽ, khung staff)
đề: semantic search 50 triệu tài liệu, lọc theo phòng ban & ngày, p99 < 150ms, không được trả kết quả không liên quan => bước 1 làm rõ: "không liên quan" → cần threshold, lọc phòng ban/ngày → metadata filter (`where`), recall mục tiêu? => bước 2 index & metric: HNSW, space="cosine", embedding model multilingual, normalize => bước 3 pipeline: query → embed (cache) → hybrid (vector + BM25 để bắt tên riêng) top-100 kèm `where` filter → rerank cross-encoder → threshold loại rác → top-k => bước 4 scale: 50M → shard/replicate (quyển 1), tách compute/storage => bước 5 đáp p99: cắt ef_search động, cache, giới hạn rerank => bước 6 chống trả rác: threshold + "không đủ giống → trả rỗng, đừng bịa" => bước 7 monitoring: recall@k + phân bố distance + tỉ lệ rỗng => bước 8 kết bằng trade-off (vector vs hybrid; threshold chặt ít rác nhưng có thể sót vs lỏng)

---

## BẢNG — 5 loại similarity (tách bạch embedding vs không-embedding)
| Loại | Cơ chế THẬT | ChromaDB (embedding) làm? |
|---|---|---|
| **① Text document** | word/document **embedding** → vector similarity | ✅ đúng chuyên môn |
| **② Image / Video** | deep NN **embedding** (CLIP/SigLIP), color histogram | ✅ đúng chuyên môn |
| **④ Recommendation** | user/item **embedding** (two-tower) → vector similarity | ✅ đúng (tầng retrieval) |
| **③ Structured data** | Euclidean trên số, **edit distance** trên chuỗi | ⚠️ edit distance KHÔNG phải embedding — Chroma không làm |
| **⑤ Genomics** | cổ điển: **sequence alignment (BLAST)**; hiện đại: protein **embedding (ESM-2)** | ⚠️ cổ điển KHÔNG phải vector DB; hiện đại thì có |

## BẢNG — Mỗi loại similarity → cơ chế → công cụ → Big-O
| Loại | Đo bằng | Công cụ điển hình | Big-O |
|---|---|---|---|
| Semantic (text/image/rec) | embedding + cosine/L2 | **ChromaDB + HNSW** | ~O(log N) search |
| String/syntactic | edit distance (Levenshtein) | thư viện string, DB text | O(m·n) mỗi cặp |
| Sequence (genomics cổ điển) | alignment (BLAST/SW) | BLAST, Smith-Waterman | ~O(m·n) mỗi cặp |
| Sequence (genomics hiện đại) | protein embedding + cosine | ESM-2 + vector DB | ~O(log N) search |
| Numeric tabular | Euclidean k-NN | vector DB hoặc k-NN cổ điển | tùy index |

## BẢNG — Đính chính bài gốc
| Bài gốc | Đúng ra là |
|---|---|
| code `n: 3` | 🚨 phải `nResults: 3` (JS) / `n_results=3` (Python) |
| "5 loại" gộp chung | 🚨 embedding (text/image/rec) ≠ edit distance (structured-string) ≠ alignment cổ điển (genomics) |
| kết quả có "distances **và** similarity scores" | 🚨 chỉ có `distances`; similarity là nghịch đảo, tự tính (1 − distance) |
| query như hộp đen exact | thực ra approximate qua **HNSW** (O(log N)) |
| (không nhắc) threshold | top-k luôn trả đủ k kể cả rác → thiếu cơ chế lọc |
| "comment tên collection là lỗi" | lủng củng — `.query()` gọi trên object, không truyền tên vào |
| genomics đơn giản hoá | không phân biệt alignment cổ điển (BLAST) vs embedding hiện đại (ESM-2) |

## BẢNG — Tổng kết cả loạt 6 quyển (capstone)
| Quyển | Chủ đề | Một câu cốt lõi |
|---|---|---|
| 1 | Các loại vector database | brute-force O(N) chết ở scale → **ANN/HNSW** O(log N), đổi recall lấy tốc độ |
| 2 | Ứng dụng vector DB | embed mọi thứ → similarity search; recsys là **retrieval→ranking 2 tầng**; "vector" GIS ≠ embedding |
| 3 | ChromaDB & RAG | RAG = **thi mở sách ở inference-time** (không train); vá knowledge-cutoff & private-data, không vá bias |
| 4 | Embeddings & distance metrics | cosine ∈ **[−1,1]** (không phải 0–1); đo **hướng** không đo độ lớn; normalize → cosine≡L2 |
| 5 | Collections & embeddings | collection ≈ ngăn tủ (không phải SQL table); **EF là contract** — đổi model = re-embed cả kho |
| 6 | Similarity search | **semantic (embedding) ≠ syntactic (edit distance)**; search = metric + HNSW; top-k cần **threshold** |

> Sợi chỉ đỏ: embed dữ liệu thành vector (4,5) → lưu trong collection (5) → index bằng HNSW (1) → **similarity search** (6) → nuôi ứng dụng (2) và RAG (3).

---

## TỪ KHÓA MỒI
- Mạch chính: **search = metric + HNSW, approximate**
- Giải phẫu 5 bước: **nhận → embed → tìm k → lọc → trả**
- Ví dụ coffee: **xe hơi 0.954 lọt top-3 → threshold**
- API query 2026: **nResults không phải n**
- Distance vs similarity: **chỉ có distances, tự tính 1−d**
- Semantic vs syntactic: **coffee/tea edit=5 nhưng gần nghĩa**
- Genomics: **BLAST → ESM-2 + cosine**
- 3 lỗi: **n / mọi similarity / thiếu threshold**
- Edge cases: **near-duplicate, metric lệch**
- Pipeline scale: **retrieve → rerank → hybrid**
- Bottleneck & cost: **Chroma không billion-scale**
- Khi nào KHÔNG: **mã đơn cần exact match**
- Monitoring: **distance tăng dần = drift**
- System design: **50M, p99<150ms, không trả rác**

## ĐÃ PHỦ
- **Vấn đề & định nghĩa:** similarity search = tìm thứ giống nhất, nghiệp vụ trung tâm IR, khác exact match (gần giống ≠ trùng khớp), độ giống phụ thuộc ngữ cảnh ✓
- **Thuật ngữ:** similarity search, query, top-k/nResults, distance (thấp=giống), similarity score (nghịch đảo distance), semantic vs syntactic, queryTexts/queryEmbeddings, where ✓
- **Cơ chế:** distance metric (đo) + ANN/HNSW (tìm nhanh), O(log N) vs O(N), approximate đổi recall lấy tốc độ ✓
- **Giải phẫu 5 bước:** nhận query → embed (cùng EF) → tìm k (HNSW+metric) → lọc where → trả (ids/documents/metadatas/distances) ✓
- **Ví dụ coffee:** d3 0.041, d1 0.093, d4 0.954 (xe hơi lọt top-3), top-k luôn đủ k → threshold ✓
- **5 loại similarity** (bảng): ①text ②image ④rec = embedding ✅; ③structured (edit distance) ⑤genomics cổ điển (alignment) = cơ chế khác ✓
- **API query 2026:** `n` sai → nResults/n_results, queryTexts/queryEmbeddings, where, try/catch, .query() trên object không truyền tên, get_collection trước ✓
- **Distance vs similarity score:** chỉ có distances (không có similarity_scores), nghịch đảo, cosine distance thấp=giống, similarity=1−distance, keys thật ✓
- **Edit distance:** Levenshtein chèn/xóa/thay, edit(kitten,sitting)=3, edit(coffee,tea)=5, embedding gần nhưng chữ xa, semantic≠syntactic, O(m·n) vs O(log N) ✓
- **Genomics:** cổ điển BLAST/Smith-Waterman (weighted edit distance + BLOSUM/PAM, không phải vector DB); hiện đại ESM-2/ProtT5/Ankh + cosine (thắng BLAST cho remote homology, là vector DB); đang chuyển dịch ✓
- **3 lỗi:** dùng n, tưởng làm mọi similarity, thiếu threshold ✓
- **Threshold:** xe hơi 0.954, đặt ngưỡng d<0.5, "không đủ giống → đừng trả", chống rác RAG, đo ground-truth ✓
- **Bảng mỗi loại → cơ chế → công cụ → Big-O** (semantic/string/genomics cổ điển/genomics hiện đại/numeric) ✓
- **Edge cases:** top-k rác kho nhỏ, query rỗng/OOD, metric không khớp model, near-duplicate, nhầm distance/similarity ✓
- **Pipeline scale:** retrieval thô (100-500) → rerank cross-encoder → hybrid (dense+sparse BM25+full-text+regex), tên riêng/mã cần hybrid ✓
- **Bottleneck & cost:** latency embed+HNSW+rerank, semantic cache, API embed/rerank compute, RAG LLM thống trị, Chroma không billion-scale ✓
- **Khi nào KHÔNG dùng:** match chuỗi chính xác (exact/edit/full-text), alignment sinh học (BLAST), dữ liệu nhỏ luật rõ (SQL), cần explainability (rule) ✓
- **Reliability & monitoring:** distance top-1 tăng=drift, tỉ lệ dưới threshold, recall@k ground-truth (hỏng âm thầm), failure modes (EF fail/rate-limit/metric sai/near-dup) ✓
- **System design 50M:** làm rõ threshold+filter → HNSW cosine multilingual → pipeline hybrid+rerank+threshold → scale shard → p99 → chống rác → monitoring → trade-off ✓
- **Bảng tổng kết 6 quyển + sợi chỉ đỏ** (capstone) ✓
- **Bảng đính chính bài gốc** (7 điểm) ✓
- *Bỏ có chủ đích (không phải kiến thức):* analogy "quản thủ thư viện", self-check, one-liner tiếng Anh (chỉ diễn đạt lại ý đã phủ).
