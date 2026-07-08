# Chain gối đầu — Embeddings & Distance Metrics trong ChromaDB

> Nén theo chain gối đầu: đuôi node trước = đầu node sau. Thuật ngữ giữ nguyên tiếng Anh,
> giải thích tiếng Việt ngắn. Bảng so sánh / đính chính / công thức tính tay giữ dạng bảng.
> Bài gốc có lỗi toán (cosine 0→1) + nhầm "nearest neighbor" + nguồn embedding lỗi thời
> → phần đính chính giữ nguyên như mắt xích trong chain.

---

## MẠCH CHÍNH — vì sao cần embedding + distance metric
SQL giỏi exact match ("tìm hàng có id=5") => nhưng AI cần đánh giá dữ liệu theo đặc điểm giống nhau ("ảnh trông giống ảnh này", "câu cùng ý với câu này") => SQL không làm được điều đó => nên sinh ra vector embedding => vector embedding biến mỗi object thành một điểm trong không gian nhiều chiều => sao cho thứ giống nhau nằm gần nhau => mỗi con số trong vector là một dimension (một phẩm chất/đặc trưng) => vd vector 300 chiều = 300 con số => độ giống nhau về ý nghĩa (không phải về chữ) gọi là semantic similarity => nên hai câu khác từ nhưng cùng ý vẫn gần nhau => "gần/xa" giữa hai vector đo bằng distance metric => ba distance metric phổ biến là cosine, Manhattan, Euclidean => cho một query vector, tìm điểm gần nhất gọi là nearest neighbor search => còn bản thân vector do một embedding model (AI) sinh ra tự động

## CHAIN — 3 loại dữ liệu → task (rẽ từ "embedding")
embedding không chỉ cho text => với image làm object recognition, deduplication (phát hiện ảnh trùng), scene detection, product search => với text làm translation, sentiment analysis, question answering, semantic search => với audio làm anomaly detection, speech-to-text, music transcription, phát hiện máy móc hỏng (machinery malfunction) => câu để nhớ: bất cứ thứ gì embed được thành vector đều tìm-theo-ý-nghĩa được

## CHAIN — Cosine similarity tính tay (rẽ từ "cosine")
cosine similarity đo góc/hướng giữa hai vector => công thức: cos(A,B) = (A·B) / (|A|·|B|) => tử số A·B là tích vô hướng (dot product) => mẫu số |A|·|B| là tích hai độ dài vector => vd A[5,4,2], B[4,3,2]: A·B = 5×4+4×3+2×2 = 36 => |A|=√45≈6.708, |B|=√29≈5.385 => cos(A,B) = 36/36.125 ≈ 0.9965 (rất giống) => với C[3,1,4]: A·C=27, |C|=√26≈5.099, cos(A,C)=27/34.205≈0.789 (giống ít hơn) => nên A gần B hơn A gần C => càng gần 1 càng giống hướng

## CHAIN — Manhattan & Euclidean tính tay (rẽ từ "distance metric")
Manhattan distance (L1) là tổng trị tuyệt đối hiệu từng chiều => L1(A,B) = |5-4|+|4-3|+|2-2| = 2, còn L1(A,C) = |5-3|+|4-1|+|2-4| = 7 => nên A gần B hơn (2 < 7) => với distance thì nhỏ = gần (ngược với similarity) => Euclidean distance (L2) là đường chim bay (định lý Pythagoras) => L2(A,B) = √(1²+1²+0²) = √2 ≈ 1.4142, còn L2(A,C) = √(2²+3²+2²) = √17 ≈ 4.1231 => nên A gần B hơn (1.41 < 4.12) => cả ba metric đều kết luận A gần B hơn — nhưng chúng đồng ý ở đây là do ví dụ này, không phải luôn luôn

## CHAIN — khi nào 3 metric BẤT ĐỒNG (rẽ từ "không phải luôn luôn")
ba metric đồng ý trong ví dụ đồ chơi nhưng có thể xếp hạng ngược trên dữ liệu thật => vì cosine đo hướng, bỏ qua magnitude (độ lớn) => hai vector cùng hướng dù dài ngắn khác nhau → cosine = 1 (giống hệt) => còn Euclidean và Manhattan đo khoảng cách tuyệt đối, có tính độ lớn => vd document dài và document ngắn cùng chủ đề → cùng hướng (cosine coi rất giống) nhưng khác độ lớn (Euclidean coi xa) => đây chính là lý do cosine được ưa chuộng cho semantic search: quan tâm ý nghĩa (hướng), không quan tâm độ dài văn bản (magnitude)

## CHAIN — normalize → 3 metric là một (rẽ từ hình học)
với vector đã normalize (độ dài 1), Euclidean và cosine cho CÙNG thứ hạng => vì quan hệ chính xác: L2(a,b)² = 2·(1 − cosine(a,b)) khi |a|=|b|=1 => tức Euclidean là hàm đơn điệu giảm theo cosine => nên sắp xếp giống hệt nhau => vì vậy nhiều vector DB normalize hết rồi dùng inner product (dot product) => vì dot product nhanh hơn mà kết quả tương đương cosine => đây là mẹo tối ưu hiệu năng kinh điển => hệ quả: chọn *metric* ít quan trọng hơn chọn *model*

## CHAIN — Đính chính cosine range −1 đến 1 (rẽ, đính chính)
bài gốc nói cosine similarity chạy từ 0 đến 1 => nhưng sai về mặt toán tổng quát => cosine của góc chạy từ −1 đến 1 => cùng hướng = +1, vuông góc = 0, ngược hướng = −1 => chỉ khi mọi thành phần vector đều không âm (như TF-IDF, count vectors) thì cosine mới nằm trong [0, 1] => vì khi đó các vector đều ở "góc phần tám dương" => nhưng embedding từ mạng neural hiện đại CÓ thành phần âm => nên cosine có thể âm (vd hai embedding trái chiều cho −0.944) => nói được "cosine ∈ [−1,1], chỉ [0,1] khi vector không âm" là tín hiệu hiểu bản chất, không học vẹt

## CHAIN — Đính chính "nearest neighbor" hai nghĩa (rẽ, đính chính)
bài gốc nói "nearest neighbor dùng để pixelating và cải thiện chất lượng ảnh" => đây là nhầm hai thứ khác nhau cùng tên "nearest neighbor" => k-NN *search* tìm vector gần nhất trong embedding space (high-dimensional), có liên quan vector DB => còn nearest-neighbor *interpolation* phóng to/thu nhỏ ảnh bằng cách lấy pixel gần nhất (2D lưới pixel), KHÔNG liên quan vector DB (là image scaling) => hai cái chỉ trùng tên, giống bẫy "vector" GIS vs "vector" embedding ở quyển 2

## CHAIN — Cosine similarity vs cosine distance (rẽ)
ChromaDB bên trong dùng cosine distance => cosine distance = 1 − cosine similarity => nên cosine similarity 1.0 (giống hệt) ↔ cosine distance 0.0 (khoảng cách 0) => khi cấu hình `metadata={"hnsw:space":"cosine"}`, kết quả `distances` Chroma trả về là cosine distance (thấp = giống) => đừng nhầm với similarity (cao = giống) => quy tắc nhớ: similarity cao = gần, distance thấp = gần

## CHAIN — Threshold chống retrieve rác (rẽ từ "nearest neighbor")
nearest neighbor LUÔN trả về một điểm gần nhất => nhưng "gần nhất" không có nghĩa là "thật sự giống" => nếu tất cả điểm đều xa thì cái gần nhất vẫn là rác => nên phải đặt một ngưỡng (threshold) => vd "What does the Kangaroo do?" so với 10 câu, câu "The kangaroo is hopping" có cosine 0.7776 (cao nhất), đặt threshold 0.6 => chỉ chấp nhận nếu score ≥ 0.6, dưới ngưỡng → "không có nearest neighbor phù hợp" => threshold chính là cơ chế "nếu không tìm thấy tài liệu đủ liên quan → nói không biết, đừng bịa" => nên bỏ threshold = nguồn gốc của hallucination trong RAG (quyển 3)

## CHAIN — Range search (rẽ từ "threshold")
range search lấy các vector trong một bán kính / ngưỡng khoảng cách quanh query => thay vì lấy đúng top-k gần nhất => vd gợi ý phim "cùng thể loại" (không cần *giống nhất*, chỉ cần *đủ gần*) => bản chất range search là lọc theo threshold khoảng cách (mở rộng của ý threshold)

## CHAIN — Query engine của Chroma là ANN/HNSW (rẽ, đính chính)
bài gốc mô tả "query engine tối ưu nearest neighbor" nhưng ngầm hiểu là tìm chính xác (exact) => ở quy mô thật Chroma KHÔNG quét toàn bộ (exact O(N)) => nó dùng HNSW — một ANN (Approximate Nearest Neighbor) index => nên kết quả gần đúng, đánh đổi recall lấy tốc độ O(log N) => phân biệt: distance metric trả lời "đo gần/xa thế nào" => còn HNSW trả lời "tìm cái gần đó nhanh ra sao" => hai thứ độc lập và đều cần

## CHAIN — 3 lỗi thường gặp về metric (rẽ độc lập)
lỗi 1: lẫn similarity với distance => cosine *similarity* cao = gần, còn Manhattan/Euclidean *distance* thấp = gần => nhầm chiều so sánh → xếp hạng ngược (ChromaDB dùng cosine distance = 1 − similarity) => lỗi 2: không normalize khi dùng cosine => tính bằng dot product mà quên normalize → ra inner product, không phải cosine => lỗi 3: chọn sai metric so với embedding model => model được train cho cosine mà index bằng Euclidean thô → xếp hạng lệch → dùng metric mà model khuyến nghị (đa số text embedding → cosine)

## CHAIN — Edge cases (rẽ độc lập)
vector độ dài 0 (all-zeros) → chia cho 0 trong cosine → NaN → phải loại bỏ/xử lý riêng => chiều không đồng nhất thang đo → một chiều "áp đảo" Euclidean/Manhattan → cân nhắc chuẩn hoá đặc trưng => curse of dimensionality: ở chiều rất cao mọi khoảng cách "co cụm" gần nhau → chọn metric & threshold cẩn thận, đừng tin threshold tuyệt đối cứng => query và document dùng khác model/metric → xếp hạng rác

## CHAIN — Nguồn embedding cập nhật 2026 (rẽ, đính chính)
bài gốc nói hai nguồn text embedding là TensorFlow Hub và OpenAI GPT-3 => nhưng danh sách này đã cũ => GPT-3 embeddings là đồ cổ (2020, đã deprecated), OpenAI hiện dùng text-embedding-3-small / text-embedding-3-large => TensorFlow Hub gần như không còn, nay là Hugging Face / Sentence Transformers (self-host) hoặc API providers => "hai nguồn" là quá hẹp => nhưng câu "Chroma dùng Sentence Transformers mặc định" vẫn đúng => vì default của ChromaDB là all-MiniLM-L6-v2 (384 chiều, một model Sentence Transformers) => để so sánh các model dùng MTEB (Massive Text Embedding Benchmark) => nhưng đừng chọn theo bảng xếp hạng MTEB, hãy benchmark trên chính dữ liệu của bạn => và fine-tune cho domain hẹp (pháp lý/y tế/code) thường +10–30%

## CHAIN — Chọn embedding model ở quy mô lớn (rẽ, staff)
chọn embedding model là quyết định kiến trúc tốn kém nhất => vì số chiều gắn với storage & latency => vd 3072 chiều (Gemini) tốn gấp 8 lần RAM so với 384 chiều (MiniLM) và search chậm hơn => với 1 tỷ vector, chênh lệch này = hàng TB RAM và hàng chục nghìn USD => giảm bằng Matryoshka / MRL embeddings => Matryoshka cho phép cắt bớt chiều (vd 1536 → 256) mất ít chất lượng để tiết kiệm storage khủng => một trục quyết định khác là API vs self-host => API (OpenAI/Cohere) là zero-ops nhưng trả tiền mỗi lần embed và gửi dữ liệu ra ngoài (vấn đề compliance) => self-host (BGE-M3) kiểm soát được, rẻ ở scale lớn, nhưng cần GPU => nên dữ liệu nhạy cảm thường buộc self-host => và chi phí *sinh* embedding (embed 1 tỷ tài liệu bằng GPU) thường đắt hơn cả vector search

## CHAIN — Đổi model/metric = re-embed toàn bộ (rẽ, staff)
vector từ model A và model B không so sánh được với nhau (khác không gian) => nên đổi embedding model → phải re-embed *toàn bộ* corpus và rebuild index => việc này tốn kém, phải lên kế hoạch (blue-green, version hoá embedding) => tương tự, đổi metric (cosine → L2) thường phải build lại index => nên quyết định embedding model + metric là quyết định khó đảo ngược → chọn kỹ từ đầu, benchmark trên dữ liệu thật trước khi cam kết

## CHAIN — Khi nào lo metric / bikeshedding (rẽ, staff)
NÊN cân nhắc kỹ metric khi magnitude mang thông tin (recsys với điểm số) hoặc dữ liệu không normalize => KHÔNG cần quá lo khi dùng text embedding chuẩn đã normalize => vì khi đó cosine/dot/Euclidean cho cùng thứ hạng → chọn cái nào cũng được (chọn dot vì nhanh nhất) => nên staff biết đâu là quyết định quan trọng, đâu là bikeshedding: chọn *embedding model* quan trọng gấp bội chọn *distance metric* (với vector normalized)

## CHAIN — Monitoring & reliability (rẽ, staff)
giám sát phân bố similarity score: nếu điểm nearest-neighbor trung bình tụt dần → có thể drift dữ liệu hoặc embedding model không hợp domain → cần đổi/fine-tune => threshold tuning là việc liên tục: ngưỡng 0.6 không phải hằng số vũ trụ, phụ thuộc model/domain/số chiều → đo trên tập ground-truth và theo dõi tỉ lệ "không tìm thấy trên ngưỡng" => chi phí query: cosine/L2 rẻ nhưng số chiều lớn làm mỗi phép tính đắt hơn tuyến tính → số chiều là đòn bẩy cost trực tiếp

## CHAIN — System design 50M tài liệu đa ngôn ngữ (rẽ, staff)
đề: semantic search 50 triệu tài liệu đa ngôn ngữ (Việt + Anh), ngân sách hạn chế, dữ liệu nhạy cảm không được gửi ra ngoài => bước 1 ràng buộc quyết định trước: dữ liệu nhạy cảm → loại API bên ngoài, buộc self-host => đa ngôn ngữ → cần model multilingual, ngân sách hạn chế → ưu tiên open-source + số chiều vừa phải => bước 2 chọn model: BGE-M3 (open-source, 100+ ngôn ngữ, self-host, hỗ trợ sparse+dense) là ứng viên mạnh, benchmark trên dữ liệu Việt-Anh thật (MTEB không đủ), cân nhắc Matryoshka nếu storage căng => bước 3 metric: normalize vector → dùng cosine (hoặc dot product tương đương, nhanh hơn), 50M vector → HNSW (ANN) trong Chroma/Qdrant/Milvus => bước 4 threshold: đo trên ground-truth để đặt ngưỡng "đủ liên quan", dưới ngưỡng → "không tìm thấy" thay vì rác => bước 5 vận hành: version hoá embedding, kế hoạch re-embed nếu đổi model, monitor phân bố score + recall => bước 6 kết bằng trade-off (self-host GPU cost vs API compliance, số chiều vs storage, cosine vs dot)

---

## BẢNG — Kết quả tính tay 3 metric (A[5,4,2], B[4,3,2], C[3,1,4])
| Cặp | Cosine similarity (cao=giống) | Manhattan L1 (thấp=gần) | Euclidean L2 (thấp=gần) |
|---|---|---|---|
| **A vs B** | 0.9965 | 2 | 1.4142 |
| **A vs C** | 0.7890 | 7 | 4.1231 |

> Cả ba đồng ý "A gần B hơn" — nhưng đó là do ví dụ này; trên data thật cosine (hướng) và L2 (magnitude) có thể xếp hạng ngược.

## BẢNG — Chọn distance metric nào
| Metric | Đo gì | Nhạy magnitude? | Dùng khi | "Gần" = |
|---|---|---|---|---|
| **Cosine** | góc/hướng | Không | semantic search (mặc định text embedding); độ dài văn bản không quan trọng | similarity cao / distance thấp |
| **Euclidean (L2)** | đường chim bay | Có | dữ liệu đồng nhất độ lớn; cần khoảng cách vật lý; embedding đã normalize (≡ cosine) | distance thấp |
| **Manhattan (L1)** | đi theo trục | Có | dữ liệu thưa (sparse), nhiều chiều; ít nhạy outlier hơn L2 | distance thấp |
| **Dot / Inner product** | hướng × độ lớn | Có | vector đã normalize (≡ cosine, nhanh hơn); một số recsys | score cao |

## BẢNG — Landscape embedding model 2026
| Nhóm | Model tiêu biểu | Ghi chú |
|---|---|---|
| **API (đóng)** | OpenAI text-embedding-3, Cohere embed-v4 (đa ngôn ngữ), Voyage voyage-3-large (retrieval & code), Google Gemini Embedding (đa phương thức, 3072 chiều), Amazon Titan V2, Mistral Embed | dễ tích hợp, trả phí, không tự host |
| **Open-source (mở)** | BGE-M3 (dense+sparse+multi-vector, 100+ ngôn ngữ, 8K context), Qwen3-Embedding, Jina v5, NV-Embed-v2 (top MTEB), E5, Nomic, MiniLM (384 chiều, ~46MB) | tự host, kiểm soát, không phí license |

> MiniLM (all-MiniLM-L6-v2, 384 chiều) = default của ChromaDB. Chọn theo: độ chính xác trên data của bạn, ngôn ngữ, chi phí, latency, đa phương thức.

## BẢNG — "Nearest neighbor" hai nghĩa (bẫy trùng tên)
| | k-NN *search* (chủ đề của ta) | Nearest-neighbor *interpolation* |
|---|---|---|
| Làm gì | tìm vector gần nhất trong embedding space | phóng to/thu nhỏ ảnh bằng pixel gần nhất |
| Không gian | high-dimensional (embedding) | 2D lưới pixel |
| Liên quan vector DB? | **Có** | **Không** (là image scaling) |

## BẢNG — Đính chính bài gốc
| Bài gốc nói | Đúng ra là |
|---|---|
| "cosine similarity ranges from 0 to 1" | 🚨 SAI — thực ra **−1 đến 1**; chỉ [0,1] khi vector không âm (TF-IDF); embedding neural có số âm |
| "nearest neighbor để pixelating cải thiện ảnh" | 🚨 nhầm k-NN search với nearest-neighbor interpolation (2 thuật toán trùng tên) |
| nguồn embedding "TensorFlow Hub + GPT-3" | lỗi thời — 2026 là text-embedding-3, Cohere embed-v4, Voyage, BGE-M3, Qwen3, Gemini... |
| "query engine tối ưu nearest neighbor" (ngầm hiểu exact) | thực ra Chroma dùng **HNSW (approximate)**, O(log N), đổi recall lấy tốc độ |
| "ba metric comparable" | chỉ đúng ở ví dụ đồ chơi; cosine vs Euclidean có thể xếp hạng ngược; chỉ tương đương khi đã normalize |
| (không phân biệt) cosine similarity vs distance | Chroma trả về distance (thấp = giống), dễ nhầm chiều |
| ghi cos(A,B)=.996 | giá trị chính xác ~0.9965 (làm tròn 3 số = 0.997) — chỉ là rounding |

---

## TỪ KHÓA MỒI
- Mạch chính: **exact match → embedding → distance metric**
- 3 loại data → task: **image/text/audio**
- Cosine tính tay: **36/36.125 ≈ 0.9965**
- Manhattan & Euclidean: **L1=2/7, L2=1.41/4.12**
- 3 metric bất đồng: **hướng vs magnitude**
- Normalize → 1: **L2² = 2(1−cos)**
- Đính chính cosine range: **−1 đến 1**
- Đính chính nearest neighbor: **search vs interpolation**
- Cosine similarity vs distance: **distance = 1 − similarity**
- Threshold: **0.6, chống bịa**
- Range search: **đủ gần, không cần giống nhất**
- Query engine = ANN: **HNSW gần đúng O(log N)**
- 3 lỗi metric: **lẫn similarity/distance**
- Edge cases: **all-zeros → NaN**
- Nguồn embedding 2026: **MiniLM default, MTEB**
- Chọn model quy mô lớn: **3072 chiều gấp 8× RAM**
- Đổi model = re-embed: **khác không gian**
- Bikeshedding metric: **model > metric**
- Monitoring: **score tụt = drift**
- System design: **50M Việt-Anh, BGE-M3**

## ĐÃ PHỦ
- **Vấn đề & nền tảng:** SQL exact match bất lực với "giống nhau"; embedding = điểm trong không gian nhiều chiều, giống nhau gần nhau ✓
- **Thuật ngữ:** vector embedding, dimension (300 chiều = 300 số), semantic similarity (ý nghĩa ≠ chữ), distance/similarity metric, nearest neighbor search, embedding model, collection ✓
- **3 loại data → task:** image (recognition/dedup/scene/product), text (translation/sentiment/QA/semantic search), audio (anomaly/STT/music/máy hỏng); "embed được → tìm-theo-ý-nghĩa" ✓
- **Cosine tính tay:** công thức (A·B)/(|A||B|), A·B=36, |A|=√45, |B|=√29, cos(A,B)=0.9965, cos(A,C)=0.789 ✓
- **Manhattan L1:** tổng |hiệu|, L1(A,B)=2, L1(A,C)=7, nhỏ=gần ✓
- **Euclidean L2:** Pythagoras, L2(A,B)=√2≈1.4142, L2(A,C)=√17≈4.1231 ✓
- **Bảng kết quả 3 metric** (A vs B, A vs C) ✓
- **3 metric bất đồng:** cosine=hướng (bỏ magnitude), L1/L2=khoảng cách tuyệt đối; document dài/ngắn cùng chủ đề; cosine ưu tiên semantic search ✓
- **Normalize:** L2(a,b)²=2(1−cos) khi độ dài 1, đơn điệu giảm → cùng thứ hạng, dùng dot nhanh hơn, metric < model ✓
- **Đính chính cosine −1..1:** +1/0/−1, chỉ [0,1] khi không âm (TF-IDF/góc phần tám dương), neural có số âm (−0.944) ✓
- **Đính chính nearest neighbor 2 nghĩa** (bảng): k-NN search vs interpolation ✓
- **Cosine similarity vs distance:** distance = 1 − similarity, sim 1.0 ↔ dist 0.0, hnsw:space cosine → distances = cosine distance ✓
- **Threshold:** luôn có NN nhưng ≠ giống, kangaroo 0.7776, threshold 0.6, chống bịa/hallucination ✓
- **Range search:** trong bán kính thay vì top-k, gợi ý phim cùng thể loại, = lọc threshold khoảng cách ✓
- **Query engine = ANN:** Chroma dùng HNSW (approximate), O(log N), đổi recall lấy tốc độ, metric vs HNSW độc lập ✓
- **3 lỗi metric:** lẫn similarity/distance, quên normalize→inner product, sai metric vs model ✓
- **Edge cases:** all-zeros→NaN, chiều lệch thang→chuẩn hoá, curse of dimensionality→threshold không cứng, khác model/metric→rác ✓
- **Nguồn embedding 2026** (bảng + chain): GPT-3/TF Hub lỗi thời, MiniLM default Chroma (384 chiều), MTEB nhưng benchmark data thật, fine-tune domain +10–30% ✓
- **Bảng chọn metric** (cosine/L2/L1/dot) ✓
- **Chọn model quy mô lớn:** số chiều↔storage/latency (3072 gấp 8× RAM, 1 tỷ = TB + chục nghìn USD), Matryoshka/MRL (1536→256), API vs self-host (compliance/GPU), chi phí sinh embedding ✓
- **Đổi model/metric = re-embed:** khác không gian, re-embed toàn bộ + rebuild, blue-green/version, khó đảo ngược ✓
- **Bikeshedding metric:** nên lo khi magnitude quan trọng/không normalize; không lo khi normalized (chọn dot); model > metric ✓
- **Monitoring:** phân bố score tụt = drift/domain, threshold tuning liên tục (0.6 không cố định), số chiều = đòn bẩy cost ✓
- **System design 50M:** ràng buộc (nhạy cảm→self-host, đa ngôn ngữ→multilingual) → BGE-M3 → normalize+cosine+HNSW → threshold → vận hành → trade-off ✓
- **Bảng đính chính bài gốc** (7 điểm) ✓
- *Bỏ có chủ đích (không phải kiến thức):* analogy "phòng đồ chơi"/"bản đồ ý nghĩa", self-check, one-liner tiếng Anh (chỉ diễn đạt lại ý đã phủ).
