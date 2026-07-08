# Chain gối đầu — Update & Delete dữ liệu trong ChromaDB (CRUD)

> Nén theo chain gối đầu: đuôi node trước = đầu node sau. Thuật ngữ giữ nguyên tiếng Anh,
> giải thích tiếng Việt ngắn. Bảng API / Big-O giữ dạng bảng.
> Bài gốc bịa API (updateDocument / client.collections.delete không tồn tại) + bỏ upsert +
> giấu sự thật soft-delete/compaction → phần đính chính giữ nguyên như mắt xích trong chain.

---

## MẠCH CHÍNH — vì sao cần update/delete → CRUD
dữ liệu không đứng yên (chính sách đổi, sản phẩm hết hàng, user yêu cầu xoá GDPR) => nên một knowledge base chỉ *add* mà không update/delete sẽ dần trả lời theo thông tin lỗi thời => vd index "nghỉ phép 12 ngày" cho RAG, 6 tháng sau công ty đổi thành 15 ngày => nếu không update, chatbot vẫn tự tin trả "12 ngày" — *có căn cứ* nhưng sai sự thật hiện tại => đây là một dạng hallucination "hợp lệ" (quyển 3) => nên update/delete là điều kiện để hệ thống *đúng theo thời gian* => hai thao tác này khép vòng đời CRUD => CRUD = Create (add), Read (query/get), Update, Delete => và mọi update/delete đều dựa vào id (khoá định danh duy nhất của mỗi document)

## CHAIN — Update: metadata-only vs cả document (rẽ)
update là sửa document/metadata/embedding của một id **đã tồn tại** => có hai kiểu update, tốn kém khác nhau => update metadata-only đổi nhãn (vd `stock`), không đụng vector => nên metadata-only KHÔNG re-embed → rẻ => update document (nội dung) đổi text => nên đổi text thì phải re-embed lại vector bằng EF (quyển 5) => nên update document đắt hơn (tốn compute/API) => nguyên tắc staff: đổi trạng thái (tồn kho, cờ) → metadata update; đổi nội dung → re-embed (batch nếu nhiều)

## CHAIN — upsert: thao tác bài gốc bỏ quên (rẽ)
bài gốc chỉ nói `update` (fail nếu id chưa tồn tại) => nhưng trong pipeline thật, thứ dùng nhiều nhất là `upsert` => upsert = update nếu id đã có, insert nếu id chưa có => upsert quan trọng vì pipeline re-index thường chạy lại (retry, cron) => nếu dùng `add` thì trùng id lỗi => nếu dùng `update` thì id mới lỗi => còn `upsert` thì idempotent: chạy 1 lần hay 10 lần đều ra cùng một trạng thái => nên với id ổn định (`docid:chunk`), upsert chống trùng lặp & an toàn khi re-run

## CHAIN — Delete: theo id / theo where / cả collection (rẽ)
delete document có hai cách => cách 1: `col.delete(ids=["d3"])` xoá theo id cụ thể => cách 2: `col.delete(where={"stock":0})` xoá theo điều kiện metadata (vd mọi hoa hết hàng) => delete theo `where` cực hữu ích (xoá mọi tài liệu của tenant đã rời đi, xoá mọi SKU hết hàng) => nhưng delete document KHÁC hẳn delete collection => `client.delete_collection("...")` xoá TOÀN BỘ collection (không thể hoàn tác) => xoá document = bỏ vài hồ sơ, còn xoá collection = vứt cả ngăn tủ => nên nhầm hai việc này là thảm hoạ (xoá nhầm cả kho thay vì một bản ghi)

## CHAIN — API thật vs API bịa (rẽ, đính chính)
bài gốc mô tả `updateDocument(collectionName, id, ...)` và `client.collections.delete(collectionName, id)` => nhưng hai hàm này KHÔNG tồn tại trong ChromaDB => chúng là helper tự viết trong một tutorial nào đó, không phải API thật => API thật gọi TRÊN collection object (lấy bằng `getCollection`) => tức không truyền `collectionName` như tham số của một hàm rời (chi tiết ở bảng)

## CHAIN — Async & try/catch (rẽ, bài gốc đúng)
`async` trước một hàm cho phép hàm *chờ* (await) tác vụ hoàn tất trước khi chạy tiếp => vd chờ Chroma phản hồi => `try { } catch (err) { }` thì `try` thử chạy thao tác => nếu lỗi (mất kết nối Chroma, embed hỏng) thì nhảy vào `catch` => `console.log` báo thành công, `console.error` log lỗi để debug => lưu ý: đây là JS chung, không đặc thù Chroma (đừng nhầm là "API của Chroma")

## CHAIN — Sự thật: update/delete KHÔNG giống SQL (rẽ, đính chính lớn)
update/delete trong vector DB KHÔNG giống SQL => vì HNSW không có thao tác "xoá node" sạch sẽ (quyển 1) => đồ thị HNSW được xây bằng các cạnh nối => gỡ một node giữa đồ thị có thể phá vỡ khả năng điều hướng của các node khác => nên đa số vector DB (gồm ChromaDB) dùng soft-delete (tombstone) => khi `delete`, vector không bị xoá vật lý ngay mà được đánh dấu "đã xoá" => query bỏ qua nó, nhưng nó vẫn chiếm bộ nhớ và vẫn nằm trong đồ thị => để dọn thật sự cần compaction (garbage collection) => compaction là tiến trình nền định kỳ xây lại segment/index sạch, loại bỏ tombstone thật => ChromaDB (log-structured, quyển 3) dùng WAL → compaction → segment, có garbage collector "three-phase deletion" (tách xoá *logic* khỏi xoá *vật lý*)

## CHAIN — 3 hệ quả của soft-delete (rẽ)
hệ quả 1: delete không giải phóng bộ nhớ ngay => xoá 1 triệu vector không tức thì giảm RAM, phải chờ compaction => hệ quả 2: update = delete + insert => đổi một vector ≈ tombstone cái cũ + chèn cái mới → đồ thị phình tombstone => hệ quả 3: workload update/delete nặng → index thoái hoá => nhiều tombstone → recall giảm, search chậm dần → cần rebuild định kỳ => đây là lý do "chat memory prune liên tục" là workload khó (quyển 1 & 3)

## CHAIN — Re-embed: chi phí ẩn của update nội dung (rẽ)
khi update *document* (không phải metadata), Chroma phải re-embed => với API EF (OpenAI) → tốn tiền + latency mạng mỗi lần => với local EF → tốn GPU/CPU => nên update hàng loạt document thì nên batch re-embed => và nhớ: nếu đổi cả embedding model thì đó KHÔNG phải "update" => mà là re-embed toàn bộ collection (quyển 4–5), việc khác hẳn

## CHAIN — Consistency & freshness của update/delete (rẽ)
kiến trúc log-structured ghi (gồm update/delete) vào WAL trước, index cập nhật bất đồng bộ => nên sau khi `delete` có thể có độ trễ nhỏ trước khi document *thật sự* biến khỏi kết quả query => vì index nền chưa cập nhật xong => với "right to be forgotten" (GDPR), độ trễ này có ý nghĩa pháp lý => nên cần biết SLA xoá thật là bao lâu, và khi nào dữ liệu thật sự rời khỏi WAL/backup

## CHAIN — Edge cases (rẽ độc lập)
update một id không tồn tại → `update` có thể lỗi/no-op tùy version → dùng `upsert` nếu muốn "có thì sửa, không thì tạo" => delete id không tồn tại → thường no-op (không lỗi), nhưng đừng dựa vào đó để suy "đã xoá thành công" => tombstone bloat: sau nhiều delete/update index đầy tombstone → recall/latency xấu → lên lịch rebuild/compaction => delete theo `where` khớp 0 bản ghi → không lỗi nhưng tưởng đã xoá → luôn verify bằng `count()` => xoá nhầm collection → không hoàn tác → cần backup/confirmation trước lệnh `delete_collection`

## CHAIN — Mass delete & rebuild cadence (rẽ, staff)
với tỷ vector, update/delete là bài toán vận hành => mass delete (xoá 100 triệu vector của một tenant rời đi): đánh tombstone thì nhanh => nhưng RAM không giảm tới khi compaction => và compaction O(N) rất nặng => nên lên lịch compaction ngoài giờ cao điểm => và theo dõi tombstone ratio làm tín hiệu rebuild => index thoái hoá: workload update/delete cao → recall trôi (quyển 1) → rebuild định kỳ (coi như release artifact) => staff phải đặt cadence rebuild dựa trên tỉ lệ mutation, không để index "mục" âm thầm => bottleneck của update nội dung là re-embed (GPU/API), không phải chèn index

## CHAIN — GDPR / right to be forgotten (rẽ, staff)
vector là *dẫn xuất* của dữ liệu gốc => nên xoá đúng GDPR phải gồm cả vector, document, metadata, VÀ mọi bản sao (WAL, backup, replica, cache) => soft-delete chưa đủ: tombstone nghĩa là dữ liệu vẫn còn vật lý tới compaction => nên với GDPR có thể chưa đạt yêu cầu → cần đảm bảo compaction/purge thật sự loại bỏ và xử lý backup => embedding có thể rò rỉ thông tin: vector là biểu diễn của nội dung gốc → về lý thuyết có thể đảo ngược một phần => nên xoá vector là bắt buộc, không chỉ xoá text => tóm lại "xoá dữ liệu người dùng" là một quy trình đầu-cuối, không phải một lệnh `delete`

## CHAIN — Update in-place vs rebuild (rẽ, staff)
update in-place rẻ cho ít thay đổi, nhưng tích luỹ tombstone → thoái hoá => rebuild định kỳ (blue-green) dựng index mới sạch từ nguồn rồi swap => tốn compute nhưng khôi phục recall & giải phóng RAM => nên với collection mutation cao, blue-green rebuild theo lịch thường lành mạnh hơn update in-place vô hạn => nhưng nếu collection gần như tĩnh (ít update/delete) thì update in-place là đủ, đừng vẽ vời rebuild

## CHAIN — Reliability, monitoring, cost (rẽ, staff)
monitor: tombstone ratio, recall drift sau mutation, độ trễ xoá (add→invisible), chi phí re-embed, dung lượng collection (không giảm dù đã delete → dấu hiệu cần compaction) => idempotency: upsert + id ổn định để pipeline retry an toàn => failure modes: delete "thành công" nhưng dữ liệu còn trong backup (GDPR fail), tombstone bloat làm search chậm, update document quên re-embed, xoá nhầm collection => cost: update nội dung = re-embed (đắt), mass delete + compaction = burst compute → lên kế hoạch, đừng để bất ngờ

## CHAIN — System design xoá GDPR 2 tỷ vector (rẽ, khung staff)
đề: user yêu cầu xoá toàn bộ dữ liệu khỏi hệ RAG 2 tỷ vector, 500 tenant, đúng GDPR => bước 1 làm rõ: SLA xoá (72h?), "dữ liệu" gồm vector + document + metadata + bản sao, có backup/replica không => bước 2 định vị dữ liệu: id/metadata gắn `user_id` → `collection.delete(where={"user_id":X})` => nếu tách collection-per-tenant (quyển 5) thì gọn hơn => bước 3 soft-delete → hard purge: `delete` đánh tombstone ngay (người dùng hết thấy) → kích hoạt compaction/purge để xoá vật lý trong SLA → xử lý WAL & backup (điểm hay bị quên) => bước 4 xác minh: kiểm tra `count`/query để chắc dữ liệu không còn truy hồi được, log để audit => bước 5 scale: mass delete → tombstone nhanh, compaction O(N) lên lịch, theo dõi tombstone ratio & recall => bước 6 rủi ro embedding: xoá cả vector (không chỉ text) vì vector có thể rò rỉ nội dung => bước 7 kết bằng trade-off: soft-delete (nhanh, chưa purge) vs hard-delete tức thì (đắt); collection-per-tenant giúp xoá dễ

---

## BẢNG — API bịa (bài gốc) vs API THẬT (2026)
| Việc | API bịa (bài gốc) | API THẬT |
|---|---|---|
| Update document | `updateDocument(collectionName, id, doc, meta)` | `collection.update({ ids, documents, metadatas })` |
| Delete document | `client.collections.delete(collectionName, id)` | `collection.delete({ ids })` hoặc `{ where }` |
| Delete collection | (không phân biệt) | `client.deleteCollection({ name })` / `client.delete_collection(name)` |
| Upsert (bài gốc bỏ quên) | — | `collection.upsert({ ids, documents, metadatas })` |

> Update/delete document gọi **trên collection object** (lấy bằng `getCollection`), KHÔNG truyền `collectionName` vào hàm rời.

## BẢNG — Big-O các thao tác CRUD
| Thao tác | Chi phí | Ghi chú |
|---|---|---|
| `add` / `upsert` (insert) | embed (đắt) + chèn HNSW ~O(log N) | insert incremental được |
| `update` metadata-only | ~O(1) cập nhật metadata | KHÔNG re-embed, không đụng đồ thị |
| `update` document | re-embed + (tombstone cũ + insert mới) | đắt nhất; phình tombstone |
| `delete` theo id | O(1) đánh tombstone | **không** giải phóng RAM ngay |
| `delete` theo `where` | O(số match) đánh tombstone | tiện cho mass delete |
| `delete_collection` | ~O(1) logic, dọn nền | vứt cả kho |
| **compaction** (nền) | O(N) xây lại segment | dọn tombstone, khôi phục recall |

---

## TỪ KHÓA MỒI
- Mạch chính: **add-only → trả lời lỗi thời**
- Update metadata vs document: **metadata rẻ, nội dung đắt**
- upsert: **idempotent, chạy lại an toàn**
- Delete id/where/collection: **document ≠ collection**
- API thật vs bịa: **collection.update trên object**
- Async & try/catch: **JS chung, không đặc thù Chroma**
- Sự thật soft-delete: **HNSW không xoá node sạch**
- 3 hệ quả: **update = delete + insert**
- Re-embed: **đổi model ≠ update**
- Consistency & freshness: **WAL trước, index async**
- Edge cases: **verify bằng count()**
- Mass delete & rebuild: **tombstone ratio**
- GDPR: **quy trình đầu-cuối, không phải 1 lệnh**
- In-place vs rebuild: **blue-green khi mutation cao**
- Reliability & cost: **dung lượng không giảm = cần compaction**
- System design: **2 tỷ vector, delete where user_id**

## ĐÃ PHỦ
- **Vấn đề & CRUD:** dữ liệu đổi (nghỉ phép 12→15 ngày), add-only → hallucination hợp lệ, update/delete khép CRUD (Create/Read/Update/Delete), dựa vào id ✓
- **Thuật ngữ:** CRUD, id, update, upsert (idempotent), delete document, delete collection, async/await, try/catch, re-embed, soft-delete/tombstone ✓
- **Update metadata vs document:** metadata-only không re-embed (rẻ), document re-embed (đắt), đổi trạng thái→metadata, đổi nội dung→re-embed batch ✓
- **upsert:** update-or-insert, quan trọng cho pipeline re-run (retry/cron), add trùng id lỗi/update id mới lỗi, idempotent, id ổn định docid:chunk ✓
- **Delete:** theo id, theo where (tenant rời/SKU hết hàng), delete collection (không hoàn tác), document ≠ collection ✓
- **API thật vs bịa** (bảng): updateDocument/client.collections.delete bịa; collection.update/delete, client.deleteCollection thật; gọi trên object ✓
- **Async & try/catch:** async chờ await, try/catch bắt lỗi (mất kết nối/embed hỏng), console.log/error, JS chung không đặc thù Chroma ✓
- **Sự thật soft-delete:** HNSW không xoá node sạch (cạnh nối, gỡ node phá điều hướng), tombstone (chiếm RAM + nằm trong đồ thị), compaction/GC, WAL→compaction→segment, three-phase deletion ✓
- **3 hệ quả:** delete không giải phóng RAM ngay (chờ compaction), update = delete+insert (phình tombstone), mutation cao → index thoái hoá → rebuild (chat memory prune khó) ✓
- **Re-embed chi phí ẩn:** update document → re-embed, API EF (tiền+latency) / local (GPU/CPU), batch, đổi model ≠ update (re-embed toàn bộ) ✓
- **Consistency & freshness:** WAL trước index async, độ trễ delete→invisible, GDPR ý nghĩa pháp lý, SLA xoá + WAL/backup ✓
- **Bảng Big-O:** add/upsert, update metadata O(1), update document (re-embed+tombstone), delete id O(1) tombstone, delete where, delete_collection, compaction O(N) ✓
- **Edge cases:** update id không tồn tại (lỗi/no-op→upsert), delete id không tồn tại (no-op), tombstone bloat, delete where 0 match (verify count()), xoá nhầm collection (backup/confirm) ✓
- **Mass delete & rebuild cadence:** 100M vector tombstone nhanh nhưng RAM chờ compaction O(N), compaction ngoài giờ cao điểm, tombstone ratio, cadence rebuild theo mutation, bottleneck = re-embed ✓
- **GDPR/right to be forgotten:** vector là dẫn xuất → xoá cả vector+document+metadata+WAL+backup+replica+cache, soft-delete chưa đủ, embedding rò rỉ → xoá vector bắt buộc, quy trình đầu-cuối ✓
- **Update in-place vs rebuild:** in-place rẻ nhưng tích tombstone, blue-green rebuild khôi phục recall+RAM, mutation cao → rebuild, tĩnh → in-place đủ ✓
- **Reliability/monitoring/cost:** tombstone ratio/recall drift/độ trễ xoá/re-embed cost/dung lượng, idempotency upsert+id, failure modes (backup GDPR fail/bloat/quên re-embed/xoá nhầm), cost re-embed + burst compaction ✓
- **System design GDPR 2 tỷ:** làm rõ SLA/phạm vi → định vị delete where user_id/collection-per-tenant → soft-delete→hard purge+WAL/backup → xác minh count+audit → scale compaction → xoá cả vector → trade-off ✓
- *Bỏ có chủ đích (không phải kiến thức):* analogy "tủ hồ sơ", self-check, one-liner tiếng Anh (chỉ diễn đạt lại ý đã phủ).
