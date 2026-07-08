# Chain gối đầu — ChromaDB: Concepts, Architecture & RAG

> Nén theo chain gối đầu: đuôi node trước = đầu node sau. Thuật ngữ giữ nguyên tiếng Anh,
> giải thích tiếng Việt ngắn. Bảng so sánh / đính chính giữ dạng bảng.
> Bài gốc mô tả đúng cơ chế nhưng gọi sai tên (không nói "RAG") + sai "train LLM" + sai "giảm bias"
> → phần đính chính giữ nguyên như mắt xích trong chain.

---

## MẠCH CHÍNH — LLM hỏng ở đâu → RAG vá thế nào
LLM có những hạn chế thật khi dùng cho sản phẩm => hạn chế 1 là knowledge cutoff (chỉ biết tới thời điểm nó được train) => nên hỏi tin hôm nay / chính sách tháng trước thì LLM không biết => hạn chế 2 là hallucination (khi không chắc vẫn trả lời rất tự tin nhưng sai) => hallucination là hạn chế nguy hiểm nhất trong sản phẩm thực tế => hạn chế 3 là không biết dữ liệu riêng (private/proprietary data) => vì LLM chưa từng thấy tài liệu nội bộ, wiki, hợp đồng, codebase của *bạn* => cả ba có một lời giải chung, tên chính thức là RAG => RAG = Retrieval-Augmented Generation => ý tưởng RAG: đừng bắt LLM trả lời từ trí nhớ, hãy đưa cho nó tài liệu liên quan ngay trong câu hỏi => tức khi LLM sắp trả lời, ta retrieve thông tin liên quan từ một kho tài liệu (lưu trong ChromaDB) => rồi augment (nhét) tài liệu đó vào prompt => để model trả lời dựa trên sự thật cụ thể thay vì "bịa" => điểm cốt lõi: RAG hoạt động ở inference time (thời điểm suy luận) => nên trọng số (weights) của LLM KHÔNG hề thay đổi => vì vậy RAG (inference-time) ≠ fine-tuning (training-time) — nói nhầm hai cái là mất điểm phỏng vấn

## CHAIN — Thuật ngữ RAG (rẽ từ "RAG")
trong RAG, bước tìm tài liệu liên quan bằng similarity search gọi là retrieval => retrieval chính là việc của ChromaDB => nhét context retrieved vào prompt gọi là augmentation => neo câu trả lời của LLM vào nguồn cụ thể gọi là grounding => grounding giúp giảm bịa (hallucination) => còn bản thân đoạn văn được biểu diễn thành dãy số gọi là embedding

## CHAIN — ChromaDB & Collection (rẽ từ "ChromaDB")
ChromaDB là một vector database thiên về developer-friendly => nó cực dễ bắt đầu (chỉ `pip install chromadb`) => nên phổ biến cho prototyping và RAG => đơn vị lưu trong ChromaDB là collection => collection giống một "bảng" chứa embeddings + documents + metadata => tên collection phải 3–512 ký tự [a-zA-Z0-9._-] => tạo collection với metadata `{"hnsw:space":"cosine"}` để chọn cosine, và `embedding_function=None` để tự đưa vector => client mặc định là in-memory cho prototyping => còn `PersistentClient(path=...)` để lưu ra đĩa

## CHAIN — CRUD workflow ChromaDB (rẽ từ collection)
vòng đời một collection là CRUD => `add()` nạp ids + embeddings + documents + metadatas => `count()` đếm số bản ghi => `query()` là bước retrieval => query() nhận query_embeddings, n_results, và where (metadata filter) => where lọc theo metadata (filtered search, cực quan trọng ở production) => `update()` cập nhật embedding của một id => `delete()` xóa một id => quan trọng: ChromaDB chỉ lo lưu & tìm vector, nó KHÔNG "trả lời" câu hỏi => vì LLM là một hệ thống riêng đứng sau

## CHAIN — "Architecture" bài gốc thực ra là API workflow (rẽ, đính chính)
bài gốc gọi 5 bước (obtain embeddings → create collections → store → collection operations → query) là "ChromaDB architecture" => nhưng đây KHÔNG phải architecture (kiến trúc hệ thống) => đây là API workflow — trình tự các lệnh bạn gọi => tức "cách bạn *dùng*" ≠ "cách nó *được xây*" => kiến trúc thật (WAL, compaction, segment, HNSW/SPANN, deployment modes) mới là architecture đúng nghĩa

## CHAIN — RAG pipeline 2 giai đoạn (rẽ từ "pipeline")
RAG pipeline thật gồm 2 giai đoạn => giai đoạn A là Indexing (offline, làm 1 lần / cập nhật định kỳ) => indexing: tài liệu thô → CHUNK (cắt nhỏ) → EMBED (embedding model) → STORE (ChromaDB) => giai đoạn B là Serving (online, mỗi câu hỏi) => serving: câu hỏi → EMBED → RETRIEVE (Chroma tìm top-k chunk) → [RE-RANK] → AUGMENT (nhét chunk vào prompt) → GENERATE (LLM) → câu trả lời => điểm mấu chốt: ChromaDB dừng ở bước RETRIEVE => còn AUGMENT + GENERATE là một hệ thống khác (LLM), Chroma không biết gì về LLM

## CHAIN — 3 lỗi RAG pipeline (rẽ độc lập)
lỗi 1: tưởng ChromaDB "trả lời" câu hỏi => không — Chroma chỉ trả về tài liệu, việc tạo câu trả lời là của LLM ở bước sau => lỗi 2: query embedding và document embedding dùng model khác nhau => nếu khác model thì vector nằm khác không gian → retrieve ra rác → phải embed câu hỏi và tài liệu bằng cùng một embedding model => lỗi 3: không chunk / chunk sai kích thước => nhét cả tài liệu 50 trang làm 1 vector → retrieve đúng chủ đề nhưng loãng, còn chunk quá nhỏ thì mất ngữ cảnh

## CHAIN — Kiến trúc thật: log-structured (rẽ)
ChromaDB 2026 đã viết lại lõi bằng Rust (nhanh hơn ~4x, thoát Python GIL) => kiến trúc là log-structured (giống các storage engine hiện đại) => write path: ghi vào WAL (Write-Ahead Log) trước => WAL là một log append-only, bền vững (bản distributed lưu trên S3/object storage) => ghi được xác nhận SAU KHI WAL bền => rồi index được cập nhật bất đồng bộ ở nền => compaction là service nền biến lịch sử WAL thành segment tối ưu cho đọc (read-optimized) => tức compaction "biến ghi gần đây thành đọc nhanh" => read path: query kết hợp segment đã index + WAL gần đây => nhờ đó kết quả luôn mới, không bỏ sót ghi vừa xong => index bên trong segment là HNSW (cho vector search) và SPANN (cho quy mô lớn hơn)

## CHAIN — SPANN (rẽ từ "SPANN")
SPANN là index kiểu phân cụm + posting list => query tìm các cluster center trước => rồi chỉ lục trong posting list của cluster khớp nhất => trực giác này giống IVF (quyển 1) => nhưng SPANN tối ưu để phần lớn dữ liệu nằm trên disk/object storage thay vì RAM => nên rẻ hơn ở scale lớn => tóm lại HNSW cho tốc độ/in-memory, SPANN cho scale/disk-based

## CHAIN — Chunking, bài toán khó (rẽ)
retrieval chỉ tốt khi chunk tốt => nên chunking là khâu quyết định chất lượng RAG (nhưng cực khó) => chunk quá to → 1 vector "trộn" nhiều ý → retrieve đúng chủ đề nhưng loãng (precision thấp) => chunk quá nhỏ → mất ngữ cảnh, câu bị cắt giữa chừng (semantic incoherence) => ranh giới chunk là một vấn đề mở => các chiến lược: fixed-size + overlap, cắt theo câu/đoạn, semantic chunking (cắt tại chỗ ý nghĩa đổi), recursive theo cấu trúc tài liệu (heading → đoạn → câu) => không có "kích thước chunk vàng" cho mọi trường hợp, phải thử nghiệm trên chính dữ liệu của bạn => vì thực tế 80% công sức nằm ở chunking + retrieval quality, không phải ở vector search

## CHAIN — Re-ranking (rẽ)
ANN retrieval nhanh nhưng thô => nên thêm một tầng re-ranking bằng cross-encoder (cải tiến ROI cao nhất) => bước 1: Chroma retrieve top-20 (nhanh, recall cao, hơi thô) => bước 2: một cross-encoder (Cohere Rerank, ms-marco-MiniLM) chấm điểm từng cặp (câu hỏi, chunk) cùng nhau => chấm theo cặp chính xác hơn nhiều so với dot product => rồi chọn ra top-3 thật sự liên quan để đưa vào prompt => đây chính là mô hình 2 tầng retrieval → ranking (giống recommendation quyển 2): vector DB lọc thô (recall), cross-encoder chấm tinh (precision)

## CHAIN — Hybrid search trong ChromaDB (rẽ)
ChromaDB bản mới hợp nhất trong một query interface => gồm dense vector + sparse vector (BM25) + full-text + regex + metadata filter => vì hybrid (semantic + keyword) thắng pure vector ở production => đặc biệt với tên riêng, mã lỗi, ID, version number mà embedding hay bỏ sót => nên với RAG doanh nghiệp (tài liệu kỹ thuật đầy mã sản phẩm), hybrid gần như bắt buộc

## CHAIN — Big-O & trade-off log-structured (rẽ)
retrieval của ChromaDB: HNSW ~O(log N), SPANN ~O(số cluster khám phá + kích thước posting list) => write: append WAL = O(1) amortized => nhưng index cập nhật bất đồng bộ → có độ trễ freshness (ghi xong chưa chắc searchable ngay với index tối ưu) => đây là trade-off log-structured kinh điển: write nhanh & bền, đổi lại read-optimization bị hoãn => so với vector DB khác, ChromaDB mạnh nhất ở dễ dùng & prototype => điểm yếu: không cho billion-scale, không có GPU acceleration, latency p99 cao hơn (100–200ms bản cũ) so với Qdrant/Milvus, và không có ACID

## CHAIN — Đại phẫu ví dụ "thìa/bút" (rẽ, đính chính)
bài gốc kể LLM suy luận sai "thìa dùng viết lên giấy" rồi vector DB "sửa" nhờ các function dimension => nhưng đây là lỗi suy luận logic (bad syllogism), không phải lỗi thiếu dữ liệu để retrieve => mà RAG chỉ giải quyết thiếu kiến thức/sự kiện, KHÔNG sửa lỗi suy luận => vì embedding similarity ≠ logical reasoning => vector "thìa" và "bút" khác nhau ở chiều "function" chỉ giúp tìm tài liệu, không dạy LLM luật suy diễn => nên vector DB KHÔNG phải "cỗ máy lý luận common-sense" => và LLM 2026 không còn mắc lỗi ngây ngô này, ví dụ đã lỗi thời

## CHAIN — Đại phẫu "giảm bias" (rẽ, đính chính)
bài gốc nói "vector DB giảm bias" nhưng không có căn cứ => RAG giảm hallucination (tỷ lệ thuận với retrieval quality) nhưng KHÔNG tự giảm bias => trái lại RAG kế thừa bias & quyền hạn của kho tài liệu nguồn => nên nếu corpus lệch thì câu trả lời lệch theo => thậm chí bị *khuếch đại* vì giờ nó "có vẻ có căn cứ" => "nhét dữ liệu đa dạng vào ChromaDB để giảm bias" là lẫn giữa chất lượng data nguồn (data governance) với bản thân vector DB (chỉ là kho lưu trữ)

## CHAIN — Edge cases RAG (rẽ độc lập)
"lost in the middle": LLM hay bỏ sót thông tin nằm *giữa* context dài → đặt chunk quan trọng nhất ở đầu/cuối, đừng nhồi quá nhiều chunk => retrieve ra rác nhưng LLM vẫn "tự tin trả lời" → cần ngưỡng similarity hoặc bước "không tìm thấy → nói không biết" => tài liệu cập nhật/xóa: chính sách công ty đổi → phải update/delete trong Chroma, nếu không RAG trả lời theo bản cũ (một dạng hallucination "hợp lệ") => chunk trùng lặp đẩy các câu trả lời đa dạng ra khỏi top-k → cần dedup / diversity trong retrieval

## CHAIN — Bottleneck RAG ở scale lớn (rẽ, staff)
bottleneck của RAG KHÔNG phải vector search mà là RETRIEVAL QUALITY => vì "garbage in, garbage out": retrieve sai/thiếu thì LLM chắc chắn trả lời sai dù model xịn tới đâu => nghiên cứu 2026 chỉ rõ hallucination trong RAG hầu hết đến từ retrieval (thiếu context, bỏ sót tài liệu top-ranked, lost-in-the-middle) => nên câu hỏi staff đầu tiên là "làm sao đo & tăng retrieval quality" => bottleneck chi phí: LLM generation thống trị chi phí mỗi query (không phải vector search) => giảm cost ROI cao nhất: chunk nhỏ gọn, prompt ngắn, giới hạn độ dài câu trả lời => bottleneck latency: cộng dồn embed query + retrieve + (re-rank) + LLM generate (chậm nhất) → cache câu hỏi lặp (semantic cache) => bottleneck scale: vài triệu chunk thì Chroma OK, tới billion-scale thì Chroma không phải lựa chọn (chuyển Milvus/Vespa)

## CHAIN — Evaluation RAG (rẽ)
không đo thì không cải thiện được RAG => đo retrieval bằng recall@k và MRR (Mean Reciprocal Rank — vị trí trung bình của chunk đúng đầu tiên) => đo generation bằng Faithfulness (RAGAS) — câu trả lời có chỉ dựa trên context được retrieve không => faithfulness là chỉ số quan trọng nhất để đo hallucination => cùng với answer relevance — có trả lời đúng câu hỏi không => và LLM-as-judge: dùng một model khác (Claude/GPT) chấm điểm 1–5 => LLM-as-judge phổ biến nhất 2026 nhưng phải cảnh giác judge bias

## CHAIN — Bảo mật & governance (rẽ, staff)
RAG mở ra các lỗ hổng mới => lỗ hổng 1 là prompt injection qua tài liệu retrieved => kẻ xấu nhét câu lệnh độc ("bỏ qua hướng dẫn trước, làm X") vào một tài liệu => khi tài liệu đó được retrieve và đưa vào prompt, LLM có thể *tuân theo* => nên cần sanitize / tách context và giới hạn quyền của LLM => lỗ hổng 2: RAG kế thừa quyền truy cập của nguồn => nên user chỉ được retrieve tài liệu họ *được phép xem* => cần access control theo user trên collection (multi-tenancy) => nếu không RAG có thể rò rỉ dữ liệu nhạy cảm (lương, hợp đồng) cho người không có quyền => đây là failure mode nghiêm trọng nhất trong RAG doanh nghiệp => nên phải filter ACL *TRONG* retrieval, không post-filter

## CHAIN — Khi nào NÊN / KHÔNG dùng RAG (rẽ)
NÊN dùng RAG khi cần trả lời dựa trên kiến thức thay đổi thường xuyên hoặc dữ liệu riêng => hoặc khi cần trích nguồn (citation) => hoặc khi ngân sách không cho phép fine-tune liên tục (RAG rẻ & linh hoạt hơn fine-tuning khi kiến thức đổi) => KHÔNG NÊN RAG khi cần model đổi hành xử/văn phong (không phải cần dữ kiện mới) → dùng fine-tuning => KHÔNG NÊN khi kiến thức nhỏ, tĩnh, vừa context window → nhét thẳng vào prompt (long-context), khỏi cần vector DB => KHÔNG NÊN khi bài toán là suy luận/tính toán, không phải tra cứu → RAG không giúp

## CHAIN — Khi nào chọn ChromaDB (rẽ)
chọn ChromaDB khi cần prototype nhanh, team nhỏ, corpus vừa phải, muốn dev-experience tốt nhất => KHÔNG chọn Chroma khi cần billion-scale → dùng Milvus => KHÔNG chọn khi cần filtered query siêu nhanh ở scale lớn → dùng Qdrant => KHÔNG chọn khi cần ACID/JOIN với data quan hệ → dùng pgvector

## CHAIN — Ảnh hưởng tổ chức (rẽ, stakeholder)
giải thích cho stakeholder non-technical: ta không "dạy lại" AI (vừa đắt vừa chậm) => thay vào đó cho nó tra cứu tài liệu công ty ngay lúc trả lời (như một nhân viên được phép mở sổ tay) => nhờ vậy nó trả lời đúng theo dữ liệu *mới nhất* và trích được nguồn => đánh đổi: chất lượng phụ thuộc vào việc *tìm đúng tài liệu* => nên phần lớn công sức là tổ chức & làm sạch tài liệu, cộng kiểm soát ai được xem gì

## CHAIN — System design chatbot 10M trang (rẽ, khung staff)
đề: chatbot hỏi-đáp trên 10 triệu trang tài liệu nội bộ, có phân quyền, trả lời phải trích nguồn => bước 1 làm rõ: đây là tra cứu dữ kiện → RAG chứ không fine-tune, citation & phân quyền là ràng buộc thiết kế chính => bước 2 indexing: tài liệu → chunk (semantic/recursive) → embed (một model nhất quán) → store vào vector DB kèm metadata (doc_id, acl/quyền, updated_at, nguồn) => bước 3 chọn store: 10M chunk thì Chroma có thể, cần scale/latency chặt hơn thì Qdrant/Milvus, đã có Postgres thì pgvector => bước 4 serving: query → embed → hybrid retrieve (vector + BM25) top-20 kèm filter theo quyền user → re-rank (cross-encoder) top-3 → augment prompt có kèm nguồn → LLM generate → trả lời đính kèm citation (doc_id trong metadata) => bước 5 governance/security: filter ACL *trong* retrieval (không post-filter để tránh rò rỉ), chống prompt injection từ tài liệu, log truy vấn => bước 6 freshness: tài liệu cập nhật → update/delete trong store, lịch re-embed khi đổi embedding model => bước 7 evaluation & monitoring: faithfulness (RAGAS) + recall@k + LLM-as-judge, giám sát "không tìm thấy → trả lời không biết" thay vì bịa => bước 8 kết bằng trade-off (RAG vs fine-tune vs long-context, chi phí LLM generation, Chroma vs alternatives)

---

## BẢNG — 4 chế độ triển khai ChromaDB (cùng một codebase)
| Mode | Cách chạy | Lưu trữ | Khi nào dùng |
|---|---|---|---|
| **In-memory / embedded** | `chromadb.Client()` (trong tiến trình Python) | RAM | prototype, notebook, test |
| **Persistent single-node** | `PersistentClient(path=...)` | SQLite + HNSW ra đĩa | app nhỏ, 1 máy, cần lưu trữ |
| **Client-server** | `chroma run` rồi `HttpClient(...)` | server | nhiều app cùng truy cập 1 Chroma |
| **Distributed / Chroma Cloud** | tách frontend / WAL / compaction / query workers | object storage | production, serverless, scale |

## BẢNG — LLM hạn chế nào RAG vá được?
| Hạn chế LLM | RAG có vá? | Cơ chế |
|---|---|---|
| Knowledge cutoff (tin mới) | ✅ Có | retrieve tài liệu mới → đưa vào prompt |
| Không biết dữ liệu riêng | ✅ Có | index tài liệu nội bộ vào Chroma |
| Hallucination | ⚠️ Giảm (không hết) | grounding vào context; phụ thuộc retrieval quality |
| Bias | ❌ Không (có thể tệ hơn) | kế thừa bias của corpus; là vấn đề data governance |
| Lỗi suy luận logic | ❌ Không | RAG cấp *dữ kiện*, không cấp *khả năng suy diễn* |

## BẢNG — RAG vs Fine-tuning vs Long-context (chọn cái nào)
| Cách | Khi nào chọn | Bản chất |
|---|---|---|
| **RAG** | cần dữ kiện mới/riêng + citation | inference-time; đưa tài liệu vào prompt; weights không đổi |
| **Fine-tuning** | cần đổi *hành vi/văn phong* của model | training-time; đổi weights |
| **Long-context** | kiến thức nhỏ, tĩnh, vừa context window | nhét thẳng vào prompt, khỏi vector DB |

> Có thể kết hợp cả ba.

## BẢNG — Đính chính bài gốc (đọc lại nguyên văn sẽ học sai)
| Bài gốc nói | Đúng ra là |
|---|---|
| (mô tả cơ chế nhưng) không gọi tên "RAG" | thiếu từ khoá trung tâm nhất của chủ đề |
| "train LLM using ChromaDB" / "supplement LLM training" | 🚨 SAI — RAG là inference-time, KHÔNG train, weights không đổi |
| "vector DB giảm bias" | 🚨 gây hiểu nhầm — RAG kế thừa bias corpus; giảm bias là data governance; RAG chỉ giảm *hallucination* |
| ví dụ "thìa/bút" | lỗi suy luận logic, RAG không sửa; embedding similarity ≠ reasoning; đã lỗi thời |
| gọi "workflow API" là "architecture" | kiến trúc thật = log-structured (WAL/compaction/segment), HNSW/SPANN, 4 deployment modes |
| (bỏ qua) chunking, re-ranking, hybrid, evaluation, lost-in-the-middle, prompt injection, access control | gần như toàn bộ phần khó & quan trọng của RAG thực chiến |
| (không nhắc) chi phí LLM generation | đó mới là bottleneck chi phí thật của RAG |

---

## TỪ KHÓA MỒI
- Mạch chính: **LLM 3 hạn chế → RAG inference-time**
- Thuật ngữ RAG: **retrieve / augment / grounding**
- ChromaDB & collection: **developer-friendly, bảng**
- CRUD workflow: **query = retrieval, where = filter**
- Architecture đính chính: **workflow ≠ architecture**
- RAG pipeline: **indexing offline + serving online**
- 3 lỗi pipeline: **Chroma không trả lời**
- Kiến trúc thật: **WAL → compaction → segment**
- SPANN: **cluster + posting list, disk-based**
- Chunking: **quá to loãng, quá nhỏ mất ngữ cảnh**
- Re-ranking: **cross-encoder top-20 → top-3**
- Hybrid search: **dense + BM25 + regex + filter**
- Big-O & trade-off: **WAL O(1), freshness trễ**
- Đại phẫu thìa/bút: **similarity ≠ reasoning**
- Đại phẫu bias: **kế thừa bias, không giảm**
- Edge cases: **lost in the middle**
- Bottleneck: **retrieval quality, không phải vector search**
- Evaluation: **Faithfulness (RAGAS)**
- Bảo mật: **prompt injection + rò rỉ ACL**
- Nên/không RAG: **fine-tune vs long-context**
- Chọn ChromaDB: **prototype vs Milvus/Qdrant/pgvector**
- Tổ chức: **không dạy lại, cho mở sổ tay**
- System design: **10M trang, phân quyền, citation**

## ĐÃ PHỦ
- **Vấn đề & RAG:** 3 hạn chế thật (knowledge cutoff, hallucination, private data), RAG = retrieve→augment→generate, inference-time, weights không đổi, RAG ≠ fine-tuning ✓
- **Thuật ngữ:** LLM, hallucination, knowledge cutoff, RAG, embedding, retrieval, grounding, augmentation, collection, ChromaDB ✓
- **RAG bằng số:** retrieve (cosine, q gần P1) → augment (dựng prompt CONTEXT) → generate (LLM đọc context) ✓
- **ChromaDB & collection:** developer-friendly, pip install, collection = bảng (embeddings+documents+metadata), tên 3–512 ký tự, hnsw:space cosine, embedding_function=None, Client in-memory vs PersistentClient ✓
- **CRUD workflow:** add / count / query (query_embeddings, n_results, where) / update / delete; Chroma không trả lời, LLM là hệ riêng ✓
- **"Architecture" đính chính:** 5 bước là API workflow ≠ architecture; "dùng" ≠ "được xây" ✓
- **RAG pipeline 2 giai đoạn:** Indexing offline (chunk→embed→store) + Serving online (embed→retrieve→[rerank]→augment→generate); Chroma dừng ở retrieve ✓
- **3 lỗi pipeline:** tưởng Chroma trả lời, query/doc khác model, không chunk/chunk sai ✓
- **Kiến trúc thật:** Rust ~4x thoát GIL, log-structured, WAL (append-only, S3), write xác nhận sau WAL bền, index async, compaction→segment read-optimized, read path segment+WAL, HNSW/SPANN ✓
- **SPANN:** cluster + posting list, tìm center rồi lục posting, giống IVF, disk/object storage rẻ hơn, HNSW in-memory vs SPANN disk ✓
- **4 deployment modes** (bảng): in-memory/embedded, persistent single-node (SQLite+HNSW), client-server, distributed/Chroma Cloud ✓
- **Chunking:** quyết định chất lượng, quá to loãng (precision thấp), quá nhỏ mất ngữ cảnh (semantic incoherence), chiến lược fixed+overlap/câu-đoạn/semantic/recursive, không có kích thước vàng, 80% công sức ✓
- **Re-ranking:** ANN thô → cross-encoder ROI cao, top-20 → chấm cặp (Cohere Rerank, ms-marco-MiniLM) → top-3, giống recsys 2 tầng ✓
- **Hybrid search:** dense + sparse (BM25) + full-text + regex + metadata filter, thắng pure vector với tên riêng/mã/ID/version, RAG doanh nghiệp bắt buộc ✓
- **Big-O & trade-off:** HNSW O(log N), SPANN O(cluster+posting), WAL O(1) amortized, độ trễ freshness, write nhanh-bền vs read-opt hoãn; điểm yếu Chroma (không billion-scale, không GPU, p99 100–200ms, không ACID) ✓
- **Đại phẫu thìa/bút:** lỗi suy luận logic không phải thiếu data, embedding similarity ≠ reasoning, không phải cỗ máy common-sense, lỗi thời ✓
- **Đại phẫu bias:** RAG giảm hallucination (tỷ lệ thuận retrieval quality) không giảm bias, kế thừa & khuếch đại bias corpus, data governance vs vector DB ✓
- **Bảng LLM hạn chế nào RAG vá** (knowledge cutoff ✅, private ✅, hallucination ⚠️, bias ❌, lỗi suy luận ❌) ✓
- **Edge cases:** lost in the middle (chunk quan trọng đầu/cuối), retrieve rác + ngưỡng similarity/nói không biết, tài liệu cập nhật/xóa (hallucination hợp lệ), chunk trùng → dedup/diversity ✓
- **Bottleneck:** retrieval quality (garbage in/out, nguyên nhân hallucination), chi phí LLM generation, latency (generate chậm nhất, semantic cache), scale corpus (billion → Milvus/Vespa) ✓
- **Evaluation:** recall@k, MRR, Faithfulness (RAGAS - đo hallucination), answer relevance, LLM-as-judge + judge bias ✓
- **Bảo mật & governance:** prompt injection qua tài liệu (sanitize/giới hạn quyền LLM), kế thừa quyền nguồn → access control/multi-tenancy → rò rỉ dữ liệu nhạy cảm, filter ACL trong retrieval ✓
- **Nên/không RAG:** nên (kiến thức đổi/riêng, citation, rẻ hơn fine-tune); không (đổi hành vi→fine-tune, kiến thức nhỏ tĩnh→long-context, suy luận/tính toán) ✓
- **RAG vs fine-tune vs long-context** (bảng) ✓
- **Chọn ChromaDB:** prototype/team nhỏ/corpus vừa/dev-experience; không khi billion-scale (Milvus), filtered nhanh scale lớn (Qdrant), ACID/JOIN (pgvector) ✓
- **Tổ chức/stakeholder:** không dạy lại (đắt chậm), cho mở sổ tay, data mới nhất + citation, công sức tổ chức tài liệu + kiểm soát quyền ✓
- **System design 10M trang:** 8 bước từ làm rõ → indexing kèm metadata/acl → chọn store → serving hybrid+rerank+citation → governance ACL/injection → freshness → evaluation → trade-off ✓
- **Bảng đính chính bài gốc** (7 điểm) ✓
- *Bỏ có chủ đích (không phải kiến thức):* analogy "thi mở sách"/"kệ sách", self-check, one-liner tiếng Anh (chỉ diễn đạt lại ý đã phủ).
