# Chain gối đầu — Case Study: Similarity Search cho Công ty Hoa (ChromaDB end-to-end)

> Nén theo chain gối đầu: đuôi node trước = đầu node sau. Thuật ngữ giữ nguyên tiếng Anh,
> giải thích tiếng Việt ngắn. Bảng so sánh / đính chính / con số giữ dạng bảng.
> Bài gốc code nhiều bug (embeddings/collection:/n:3/generate thủ công) + gọi L2 distance là
> "similarity score" → phần đính chính giữ nguyên như mắt xích trong chain.

---

## MẠCH CHÍNH — business problem → giải pháp vector
công ty hoa nhận đánh giá xấu => vì khách đặt "hoa hồng đỏ" nhưng hết hàng, nhân viên thay tùy tiện (sai màu/loại) => hậu quả: khách bực, rework/redelivery tốn tiền, rating tụt => giải pháp: lưu toàn bộ hoa trong kho dưới dạng vector (theo đặc điểm màu/loại/hình dáng) => khi hết một loại, nhân viên similarity search để tìm hoa *giống nhất còn hàng* => rồi đề xuất cho khách TRƯỚC khi cắm => kết quả kỳ vọng: khách hài lòng hơn, rating tăng, chi phí mỗi đơn giảm => đây là *lý do kinh doanh*, không phải "vì công nghệ hay" => một staff luôn bắt đầu từ business case => 3 lợi ích: cải thiện trải nghiệm khách, thay thế nhanh khi hết hàng, tối ưu tồn kho (biết hoa nào giống nhau → gom nhóm, giảm lãng phí)

## CHAIN — 5 vector operations (rẽ, checklist vàng)
mọi dự án similarity search có 5 vector operations làm khung tư duy => operation 1: Data → Vectors (biến mô tả text/ảnh/thuộc tính thành vector số; text → word embedding, ảnh → feature extraction) => operation 2: Similarity metric (chọn cách đo gần/xa: cosine, Euclidean) => operation 3: Query → Vector (biến câu truy vấn của user thành vector bằng ĐÚNG phép biến đổi đã dùng cho data) => "cùng phép biến đổi" chính là nguyên tắc nhất quán để query và data so được với nhau => operation 4: Search (tìm các vector trong collection gần nhất với query vector) => operation 5: Retrieve & Present (lấy document + metadata trình bày cho user) => nhớ 5 bước này là nhớ xương sống của mọi ứng dụng vector

## CHAIN — Ví dụ "yellow flowers" → distance > 1 (rẽ)
kho 3 hoa embed theo màu: flower_1 đỏ [1,0,0], flower_2 vàng [0,1,0], flower_3 trắng [0,0,1] => query "yellow flowers" → vector [0,1,0] => đo khoảng cách L2 tới flower_2 (vàng) = 0 → gần nhất => tới flower_1 (đỏ) = √(1+1) ≈ 1.41, tới flower_3 (trắng) = √(1+1) ≈ 1.41 => nên flower_2 (tulip vàng) khớp nhất, đúng trực giác => chú ý: khoảng cách tới hoa KHÔNG vàng > 1 => chi tiết ">1" này giải thích tại sao output "score" của bài gốc lớn hơn 1

## CHAIN — "Score" thực ra là L2 distance, vì sao > 1 (rẽ, đính chính)
output bài gốc: flower_2 0.926, flower_1 1.040, flower_3 1.264, nói "lowest similarity score = best match" => vấn đề (a): đây là distance chứ không phải similarity score => bài lấy giá trị từ `results.distances` nhưng gọi là "score" => distance thì thấp = giống (nên "lowest = best" mới đúng) => còn "similarity score" theo nghĩa thường thì cao = giống => gọi distance là "similarity score" là mâu thuẫn nội tại => vấn đề (b): vì sao giá trị > 1? => cosine ∈ [0,1] (thực ra [−1,1]) nên không thể ra 1.264 => vậy đây KHÔNG phải cosine => ChromaDB mặc định dùng L2 (Euclidean distance squared) khi không set metric => bài gốc không set `space` nên nhận L2 squared distance, mà L2 dễ dàng > 1 => kiểm chứng: space="l2" cho flower_2 0.040 / flower_3 2.010 / flower_1 2.018; space="cosine" cho flower_2 0.007 / flower_1 0.441 / flower_3 0.443 => cùng thứ hạng nhưng thang đo khác hẳn => bài học: luôn biết mình đang dùng metric gì, đừng gọi distance là "similarity" => và vector đã normalize thì L2 và cosine cho cùng thứ hạng (chỉ con số khác)

## CHAIN — 5 bước triển khai & bug bài gốc (rẽ, đính chính)
bài gốc chia 5 bước: setup → prepare data → add → fetch all → search => khung đúng nhưng code có 5 bug/pattern lỗi thời => tóm tắt: `embeddings` phải là `embeddingFunction`; `collection:` trong query() phải bỏ; `n:3` phải là `nResults`; `generate` thủ công phải chuyển auto-embed; `DefaultEmbeddingFunction` phải lấy từ package `@chroma-core/default-embed` (chi tiết ở bảng)

## CHAIN — Metadata filter thắng embedding (rẽ)
bài gốc để embedding "đoán" màu vàng từ text "yellow flowers" => nhưng với florist thật, màu là thuộc tính có cấu trúc rõ ràng => nên xử lý màu bằng metadata filter, không phó mặc embedding => vì embedding có thể nhầm "yellow" nếu mô tả không chứa chữ đó, hoặc lẫn golden/amber => còn filter `where={"color":"yellow"}` thì chính xác tuyệt đối => nguyên tắc staff: thuộc tính rời rạc rõ ràng (màu, loại, giá, mùa) → metadata filter => còn điểm mờ ngữ nghĩa (phong cách, cảm xúc bó hoa) → embedding => kết hợp cả hai = hybrid

## CHAIN — Auto-embed vs manual generate (rẽ)
bài gốc generate embedding thủ công rồi truyền `embeddings:` vào add => vấn đề: nếu lúc query quên embed câu query bằng đúng model đó → kết quả rác => auto-embed (gắn EF vào collection) loại bỏ rủi ro này => vì Chroma đảm bảo add và query dùng cùng EF => nên pattern manual bị coi là dated => ngoại lệ hợp lệ: khi embed offline bằng pipeline riêng (GPU batch) rồi nạp vector sẵn → truyền `embeddings` là cố ý, nhưng vẫn phải tự lo query dùng cùng model

## CHAIN — Big-O của add / query / get (rẽ)
`add` = embed (O(số token) mỗi doc, phần đắt nhất là API EF) + chèn HNSW ~O(log N) => `query` = embed query + HNSW search ~O(log N), kết quả approximate (ANN) không phải exact => `get()` (fetch all) = O(N) => bài gốc dùng `collection.get()` để kiểm tra data, ổn với 3 hoa => nhưng với triệu SKU, `get()` không filter là O(N) tốn kém => nên chỉ dùng để debug, đừng dùng trong hot path

## CHAIN — Edge cases (rẽ độc lập)
top-3 luôn trả 3 kể cả rác: hoa đỏ/trắng vẫn lọt top-3 cho query "vàng" → cần threshold distance để chỉ đề xuất hoa đủ giống => hết sạch loại tương tự: không hoa nào dưới ngưỡng → nói "không có thay thế phù hợp", đừng ép đề xuất hoa lệch => mô tả nghèo nàn ("A pure white lily" ít đặc trưng) → embedding yếu → làm giàu mô tả (hình dáng, mùa, dịp) hoặc thêm metadata => đơn vị/định dạng không nhất quán giữa các mô tả → embedding lệch

## CHAIN — Scale florist thật & bottleneck (rẽ, staff)
case study có 3 hoa nhưng florist thật có hàng chục nghìn–triệu SKU (loại × màu × kích cỡ × nhà cung cấp), nhiều cửa hàng, tồn kho đổi từng giờ => tồn kho realtime là bài toán khó nhất => vì "hoa thay thế" chỉ hữu ích nếu còn hàng ở cửa hàng đó ngay bây giờ => nên phải kết hợp similarity search với filter `where={"in_stock":true, "store_id":...}` + freshness pipeline => vector "giống" mà hết hàng thì vô dụng => substitution (đề xuất hoa thay thế) chính là một recommendation problem (quyển 2) => có thể nâng lên two-tower (khách này + hoa này → gợi ý) + tín hiệu "khách thường chấp nhận thay thế nào" => cold-start: hoa/nhà cung cấp mới chưa có mô tả tốt → embedding yếu → fallback theo metadata (màu/loại) cho tới khi đủ dữ liệu => bottleneck không phải vector search (vài triệu SKU Chroma/Qdrant thừa sức) => mà là chất lượng mô tả hoa (data quality) + đồng bộ tồn kho → rác vào, gợi ý rác ra

## CHAIN — Khi nào KHÔNG cần vector DB (rẽ, staff)
KHÔNG cần vector DB nếu thay thế chỉ theo luật rõ ("hồng đỏ hết → luôn thay bằng thược dược đỏ") → một bảng mapping/quy tắc rẻ và minh bạch hơn => KHÔNG cần nếu chỉ lọc theo màu/loại/giá (thuộc tính rời rạc) → SQL/metadata filter là đủ => embedding chỉ thêm giá trị khi cần "giống về phong cách/cảm giác" (bó hoa lãng mạn vs rực rỡ) — điểm mờ khó mô tả bằng luật => KHÔNG cần nếu dữ liệu nhỏ (vài trăm SKU tĩnh) → brute-force cosine trong bộ nhớ là xong => vector DB đáng giá khi cần similarity ngữ nghĩa mờ + quy mô lớn + kết hợp filter → đừng "AI hoá" một bài toán mapping

## CHAIN — Cost & monitoring nối business (rẽ, staff)
với API EF, embed mỗi SKU/mỗi query tốn tiền → ở triệu SKU nên self-host EF hoặc batch offline => số chiều là đòn bẩy storage/latency => monitoring phải nối với business: không chỉ recall@k/latency => mà cả tỉ lệ khách chấp nhận đề xuất thay thế, chi phí rework/redelivery, rating/satisfaction => vì đó mới là lý do dự án tồn tại => một staff đóng vòng từ chỉ số hệ thống tới chỉ số kinh doanh => failure modes: stock lỗi thời (đề xuất hoa đã hết), mô tả nghèo → gợi ý lệch, EF tải model fail, threshold sai → đề xuất rác

## CHAIN — System design 500 cửa hàng (rẽ, khung staff)
đề: hệ gợi ý hoa thay thế cho chuỗi 500 cửa hàng, 200k SKU, tồn kho cập nhật mỗi 5 phút, tôn trọng tồn kho từng cửa hàng => bước 1 làm rõ: "giống" theo màu/loại (rời rạc) hay phong cách (mờ) → quyết định embedding vs filter; tồn kho theo từng cửa hàng → filter bắt buộc => bước 2 data model: mỗi SKU = document (mô tả giàu) + metadata (color, type, price, season, store_id, in_stock, updated_at) => bước 3 pipeline: mô tả → embed (batch, EF nhất quán) → collection; hybrid query: embedding (giống phong cách) + where (màu nếu khách chỉ định + in_stock + store_id) => bước 4 freshness: stock cập nhật 5 phút → upsert metadata; đề xuất phải lọc in_stock=true tại store_id => bước 5 threshold + fallback: dưới ngưỡng giống → "không có thay thế phù hợp" thay vì ép => bước 6 scale/cost: 200k SKU nhỏ với vector DB, bottleneck là data quality + sync tồn kho, self-host EF nếu cost cao => bước 7 monitoring: tỉ lệ chấp nhận thay thế, rework cost, rating (nối business) => bước 8 kết bằng trade-off (embedding vs rule-based mapping, hybrid, threshold chặt/lỏng)

---

## BẢNG — 8 chỗ code bài gốc sai / lỗi thời + sửa
| Bài gốc (sai/cũ) | Sửa đúng 2026 | Vì sao |
|---|---|---|
| `getOrCreateCollection({ embeddings: default_emd })` | `embeddingFunction: embedder` | property đúng là **`embeddingFunction`** |
| `collection.query({ collection: collectionName, ... })` | bỏ `collection:` | `query()` gọi trên collection object, không truyền tên vào |
| `n: 3` | `nResults: 3` / `n_results=3` | tham số top-k tên là **nResults**, không phải `n` |
| `default_emd.generate(...)` + `add({ embeddings })` | chỉ `add({ documents })` | gắn EF vào collection → **auto-embed**, khỏi generate thủ công |
| `require('chromadb')` lấy `DefaultEmbeddingFunction` | `@chroma-core/default-embed` | EF tách package riêng từ JS V3 |
| gọi `results.distances` là "similarity score" | là **distance** (thấp=giống), >1 vì mặc định L2 | distance ≠ similarity score; L2 không phải cosine |
| `allItems.documents[allItems.ids.indexOf(id)]` | thừa | `query()` đã trả `documents` trực tiếp |
| không set distance metric & không threshold | set `space` + đặt threshold | tránh hiểu nhầm score & đề xuất rác |

## BẢNG — Cùng dữ liệu, L2 (mặc định) vs cosine (con số thật)
| Doc | space="l2" (mặc định, distance có thể >1) | space="cosine" (∈[0,2]) |
|---|---|---|
| flower_2 (tulip vàng — khớp) | **0.040** 🥇 | **0.007** 🥇 |
| flower_1 (hồng đỏ) | 2.018 | 0.441 |
| flower_3 (ly trắng) | 2.010 | 0.443 |

> Cùng thứ hạng (tulip vàng nhất), khác thang đo. Vector normalize → L2 và cosine cùng thứ hạng.

## BẢNG — Metadata filter vs Embedding (khi nào dùng)
| Thuộc tính | Cách xử lý đúng | Vì sao |
|---|---|---|
| Rời rạc, rõ ràng (màu, loại, giá, mùa, in_stock) | **metadata filter** (`where`) | chính xác tuyệt đối; embedding có thể nhầm (yellow ↔ golden/amber) |
| Mờ, ngữ nghĩa (phong cách, cảm xúc bó hoa) | **embedding** | không mô tả được bằng luật |
| Cả hai | **hybrid** (embedding + `where`) | vừa giống phong cách vừa đúng ràng buộc rời rạc |

---

## TỪ KHÓA MỒI
- Mạch chính: **business case giảm rework**
- 5 vector operations: **data→vector...retrieve&present**
- Ví dụ yellow flowers: **distance tới hoa không vàng >1**
- Score thực ra L2: **1.264 không phải cosine**
- 5 bước & bug: **embeddingFunction/nResults/auto-embed**
- Metadata filter thắng embedding: **màu là filter**
- Auto-embed vs manual: **cùng EF add & query**
- Big-O: **get() là O(N), chỉ debug**
- Edge cases: **mô tả nghèo → embedding yếu**
- Scale florist thật: **tồn kho realtime khó nhất**
- Khi nào KHÔNG: **luật rõ → mapping table**
- Cost & monitoring: **nối rework cost + rating**
- System design: **500 cửa hàng, in_stock per store**

## ĐÃ PHỦ
- **Business problem:** đánh giá xấu do thay hoa tùy tiện, rework/redelivery tốn tiền, rating tụt; giải pháp vector+similarity search đề xuất trước khi cắm; 3 lợi ích (trải nghiệm/thay thế nhanh/tối ưu tồn kho); bắt đầu từ business case ✓
- **5 vector operations:** data→vector, similarity metric, query→vector (cùng phép biến đổi = nhất quán), search, retrieve&present ✓
- **Ví dụ yellow flowers:** [1,0,0]/[0,1,0]/[0,0,1], query [0,1,0], L2 tới vàng=0, tới đỏ/trắng √2≈1.41, >1 ✓
- **"Score" là L2 distance:** output 0.926/1.040/1.264, (a) distance ≠ similarity score (mâu thuẫn nội tại), (b) >1 vì mặc định L2 squared không phải cosine, kiểm chứng l2 vs cosine, cùng thứ hạng khác thang, normalize → cùng thứ hạng ✓
- **5 bước & 8 bug code** (bảng): embeddings→embeddingFunction, collection: thừa, n→nResults, generate→auto-embed, package @chroma-core/default-embed, distance-as-score, allItems thừa, thiếu metric+threshold ✓
- **Metadata filter vs embedding** (chain + bảng): màu rời rạc → filter (chính xác, embedding nhầm yellow/golden), phong cách mờ → embedding, kết hợp hybrid ✓
- **Auto-embed vs manual:** generate thủ công → query rác nếu khác model, auto-embed đảm bảo cùng EF, manual dated, ngoại lệ embed offline GPU batch ✓
- **Big-O:** add = embed O(token) + HNSW O(log N), query = embed + HNSW O(log N) approximate, get() = O(N) chỉ debug không hot path ✓
- **Edge cases:** top-3 trả rác → threshold, hết loại tương tự → "không có thay thế", mô tả nghèo → làm giàu/metadata, format không nhất quán → lệch ✓
- **Scale florist thật:** triệu SKU, tồn kho realtime khó nhất (where in_stock+store_id+freshness), vector giống mà hết hàng vô dụng, substitution=recommendation (two-tower), cold-start → fallback metadata, bottleneck = data quality + sync tồn kho ✓
- **Khi nào KHÔNG dùng:** luật rõ → mapping table, chỉ lọc rời rạc → SQL/filter, data nhỏ tĩnh → brute-force, vector DB đáng giá khi similarity mờ + scale + filter ✓
- **Cost & monitoring:** API EF phí mỗi SKU/query → self-host/batch, số chiều đòn bẩy, monitoring nối business (tỉ lệ chấp nhận/rework cost/rating), đóng vòng chỉ số, failure modes ✓
- **System design 500 cửa hàng:** làm rõ giống-màu-vs-phong-cách + filter tồn kho → data model metadata → pipeline hybrid → freshness upsert → threshold+fallback → scale/cost → monitoring business → trade-off ✓
- **Bảng L2 vs cosine con số** (0.040/2.018/2.010 vs 0.007/0.441/0.443) ✓
- *Bỏ có chủ đích (không phải kiến thức):* analogy "nhân viên bán hoa lão luyện", self-check, one-liner tiếng Anh (chỉ diễn đạt lại ý đã phủ).
