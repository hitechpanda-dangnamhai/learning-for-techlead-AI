# Giáo trình: Update & Delete dữ liệu trong ChromaDB

> Biên soạn theo phong cách "cầm tay chỉ việc" — từ zero đến tư duy Staff Engineer.
> Giảng bằng tiếng Việt, giữ nguyên technical terms tiếng Anh để phục vụ phỏng vấn big tech.
>
> **Đây là giáo trình thứ 8 của loạt về vector database.** Bảy quyển trước phủ: cơ chế (HNSW), ứng dụng, RAG, embeddings & distance metrics, tạo collections, similarity search, và case study florist. Quyển này khép **vòng đời CRUD**: sau *Create* (add) và *Read* (query) ở các quyển trước, giờ là **Update và Delete**. Nghe có vẻ tầm thường — nhưng update/delete trong vector database là thứ **dễ tưởng đơn giản mà thực ra khó**, và là chỗ nhiều hệ RAG production "chết âm thầm".
>
> **Cảnh báo ngay từ đầu:** đoạn API trong bài giảng gốc **phần lớn là bịa/không khớp API thật của ChromaDB**. Cụ thể: `updateDocument(collectionName, id, ...)` và `client.collections.delete(collectionName, id)` **không tồn tại** trong Chroma. API thật là `collection.update(...)`, `collection.delete(ids=...)`, và `client.delete_collection(name)` — gọi *trên collection object* hoặc *client*, không phải các hàm wrapper mà bài giảng mô tả. Bài cũng **bỏ hẳn `upsert`** (thao tác quan trọng nhất trong thực tế) và **giấu sự thật khó chịu**: update/delete trong vector DB *không* giống SQL — HNSW không xoá sạch được, phải dùng **soft-delete + compaction**. Tôi giữ phần đúng (async/try-catch), sửa toàn bộ API, và chạy thật. Đánh dấu 🛠️ **[Đính chính]** và ➕ **[Mở rộng ngoài bài gốc]**.

---

## Phần 0 — 🗺️ Bản đồ bài học (Overview)

Dữ liệu không đứng yên: chính sách công ty đổi, sản phẩm hết hàng, người dùng yêu cầu xoá dữ liệu (GDPR). Một knowledge base chỉ *add* mà không *update/delete* sẽ dần trả lời theo thông tin **lỗi thời** — một dạng hallucination "hợp lệ" (quyển 3). Bài giảng gốc dạy hai thao tác: **update** (thay text/metadata của một document theo id) và **delete** (xoá document theo id), cùng các pattern chung (async, try/catch, console.log/error). Ta sẽ đi qua tất cả — nhưng với **API thật 2026**, thêm **upsert**, và đào tới tận cơ chế soft-delete/compaction mà bài gốc không chạm.

**Sau khi học xong, bạn sẽ có thể:**
- Viết code **chạy được** để update (metadata-only vs cả document), **upsert** (insert-or-update idempotent), và delete (theo id / theo `where`) trong ChromaDB.
- Phân biệt **xoá document** với **xoá cả collection** (bài gốc lẫn lộn).
- Hiểu **sự thật sâu**: vì sao update/delete trong vector DB *không* giống SQL — HNSW soft-delete, tombstone, compaction, re-embed.
- Nhận ra & sửa các API bịa trong tài liệu kiểu bài giảng.
- Thiết kế xoá dữ liệu ở quy mô lớn: mass delete, GDPR "right to be forgotten", index degradation & rebuild.

**Mạch kiến thức từ basic → staff:**
- 🟢 **Basic:** vì sao cần update/delete → async/try-catch (đúng) → API update/delete THẬT → hello-world.
- 🟡 **Intermediate:** update (metadata vs document/re-embed) → **upsert** → delete document vs delete collection → lỗi thường gặp.
- 🔴 **Advanced:** sự thật soft-delete/tombstone/compaction, vì sao delete/update đắt, re-embed cost, Big-O, log-structured WAL.
- 🟣 **Staff:** update/delete ở scale — mass delete, GDPR, tombstone bloat, rebuild cadence, consistency/freshness, cost, system design.
- 🎯 **Cheatsheet:** Keywords, core concepts, code phải thuộc, câu hỏi phỏng vấn + one-liners.

---

## Phần 1 — 🟢 BASIC (Nền tảng)

### 1.1. "Tại sao" — dữ liệu thay đổi, và điều đó nguy hiểm

Bạn index "chính sách nghỉ phép: 12 ngày/năm" vào ChromaDB cho RAG (quyển 3). Sáu tháng sau công ty đổi thành 15 ngày. Nếu bạn **không update**, chatbot vẫn tự tin trả lời "12 ngày" — *có căn cứ* (từ tài liệu retrieved) nhưng **sai sự thật hiện tại**. Tương tự, hoa hết hàng vĩnh viễn cần **delete** khỏi kho gợi ý (quyển 7), và người dùng có quyền yêu cầu **xoá dữ liệu cá nhân** (luật). → Update/delete không phải "tính năng phụ"; nó là điều kiện để hệ thống *đúng theo thời gian*.

### 1.2. Analogy đời thường

Nghĩ về một **tủ hồ sơ có dán nhãn** (collection — quyển 5):
- **Update** = rút một hồ sơ ra, *sửa nội dung* rồi bỏ lại (theo đúng số hồ sơ = id).
- **Delete document** = *rút bỏ hẳn một hồ sơ* khỏi ngăn.
- **Delete collection** = *vứt cả ngăn tủ*.

Nhưng có một cú twist mà tủ hồ sơ giấy không có: trong vector DB, "rút bỏ" thường chỉ là **dán nhãn 'đã xoá'** (tombstone), hồ sơ vẫn nằm đó tới khi có người "dọn tủ" (compaction). Chi tiết này (Phần 3) là thứ phân biệt người hiểu sâu.

### 1.3. Thuật ngữ nền tảng (giải thích ngay lần đầu)

- **CRUD:** Create (add), Read (query/get), **Update**, **Delete** — vòng đời dữ liệu.
- **id:** khoá định danh duy nhất của mỗi document trong collection; update/delete *dựa vào id*.
- **update:** sửa document/metadata/embedding của một id **đã tồn tại** (fail nếu id chưa có, tùy version).
- **upsert:** "update or insert" — sửa nếu id đã có, thêm mới nếu chưa. **Idempotent** (chạy lại nhiều lần vẫn ra cùng kết quả).
- **delete (document):** xoá một/nhiều document khỏi collection (theo id hoặc theo điều kiện metadata).
- **delete collection:** xoá *toàn bộ* collection (khác hẳn xoá document!).
- **async / await:** từ khoá JS cho thao tác *bất đồng bộ* — hàm "chờ" Chroma trả lời trước khi tiếp tục.
- **try/catch:** khối xử lý lỗi; `try` chạy thao tác, `catch` bắt lỗi nếu có.
- **re-embed:** sinh lại vector khi *nội dung document* đổi (không cần nếu chỉ đổi metadata).
- **soft-delete / tombstone:** đánh dấu "đã xoá" thay vì xoá vật lý ngay; dọn sau bằng compaction.

### 1.4. Async & try/catch — phần bài gốc nói ĐÚNG (giữ nguyên)

Bài gốc giải thích tốt phần này, tôi giữ:
- **`async`** trước một hàm → hàm có thể *chờ* (await) tác vụ hoàn tất (vd chờ Chroma phản hồi) trước khi chạy tiếp.
- **`try { ... } catch (err) { ... }`** → `try` thử chạy thao tác; nếu lỗi (mất kết nối Chroma, embed hỏng...), nhảy vào `catch`.
- **`console.log`** báo thành công; **`console.error`** log lỗi để debug.

🛠️ *Lưu ý:* đây là **JS chung**, không đặc thù Chroma. Đúng, nhưng đừng nhầm nó là "API của Chroma".

### 1.5. 🛠️ API update/delete THẬT (sửa hàm bịa của bài gốc)

Bài gốc mô tả `updateDocument(collectionName, id, newDocument, newMetadata)` và `client.collections.delete(collectionName, id)`. **Hai hàm này KHÔNG tồn tại trong ChromaDB.** Chúng là *helper tự viết* trong một tutorial nào đó, không phải API thật. API thật:

| Việc | API bịa (bài gốc) | API THẬT (2026) |
|---|---|---|
| Update document | `updateDocument(collectionName, id, doc, meta)` | `collection.update({ ids, documents, metadatas })` |
| Delete document | `client.collections.delete(collectionName, id)` | `collection.delete({ ids })` hoặc `{ where }` |
| Delete collection | (không phân biệt) | `client.deleteCollection({ name })` |

Điểm mấu chốt: update/delete document gọi **trên collection *object*** (mà bạn đã lấy bằng `getCollection`), *không* truyền `collectionName` như tham số của một hàm rời.

### 1.6. Code "hello world" — CRUD thật (Python, đã chạy thật)

```python
import chromadb
client = chromadb.Client()
col = client.create_collection("crud_demo", embedding_function=None,
                               metadata={"hnsw:space": "cosine"})

# CREATE
col.add(ids=["d1","d2"], embeddings=[[1,0,0],[0,1,0]],
        documents=["red rose","yellow tulip"], metadatas=[{"stock":5},{"stock":0}])

# UPDATE (gọi TRÊN collection object, theo id) — sửa document của d2
col.update(ids=["d2"], embeddings=[[0.1,0.9,0]], documents=["golden tulip"])
print(col.get(ids=["d2"])["documents"])   # -> ['golden tulip']

# DELETE document theo id
col.delete(ids=["d1"])
print(col.count())                         # -> 1
```
Giải thích: `col.update(ids=["d2"], ...)` sửa document mang id `d2`; `col.delete(ids=["d1"])` xoá document `d1`. Cả hai gọi **trên `col`** (collection object), đúng API thật.

### 1.7. ✅ Self-check (Basic)

1. Vì sao một RAG knowledge base *chỉ add mà không update* lại nguy hiểm?
2. `updateDocument(collectionName, ...)` trong bài gốc có phải API thật của Chroma không? API đúng là gì?
3. Xoá một document khác xoá cả collection thế nào?

---

## Phần 2 — 🟡 INTERMEDIATE (Vận dụng)

### 2.1. Update: metadata-only vs cả document (có/không re-embed)

Có hai kiểu update, và chúng **tốn kém khác nhau**:

- **Update metadata-only:** đổi nhãn (vd `stock`), *không đụng vector* → **KHÔNG re-embed** → rẻ.
- **Update document (nội dung):** đổi text → **phải re-embed** lại vector bằng EF (quyển 5) → đắt hơn (tốn compute/API).

```python
# Chỉ đổi metadata -> KHÔNG re-embed (rẻ)
col.update(ids=["d2"], metadatas=[{"stock": 10}])

# Đổi nội dung document -> Chroma phải re-embed (nếu EF gắn vào collection)
col.update(ids=["d2"], documents=["a bright golden tulip"])
```
**Đã chạy thật:** update metadata → `stock` đổi từ 0 thành 10 mà vector giữ nguyên; update document → nội dung + vector đổi. Nguyên tắc staff: **đổi trạng thái (tồn kho, cờ) → metadata update; đổi nội dung → re-embed** (cân nhắc batch nếu nhiều).

### 2.2. ⭐ upsert — thao tác bài gốc BỎ QUÊN (quan trọng nhất thực tế)

🛠️ Bài gốc chỉ nói `update` (fail nếu id chưa tồn tại). Nhưng trong pipeline thật, thứ bạn dùng nhiều nhất là **`upsert`** = *update nếu có, insert nếu chưa*:

```python
col.upsert(ids=["d4"], embeddings=[[0.5,0.5,0]], documents=["orange gerbera"])  # id mới -> INSERT
col.upsert(ids=["d1"], embeddings=[[0.9,0.1,0]], documents=["crimson rose"])     # id cũ -> UPDATE
```
Vì sao upsert quan trọng? Vì pipeline re-index (quyển 5) thường **chạy lại** (retry, cron). Dùng `add` → trùng id lỗi; dùng `update` → id mới lỗi. Dùng `upsert` → **idempotent**: chạy 1 lần hay 10 lần đều ra cùng trạng thái. → Với id ổn định (`docid:chunk`), upsert là "vũ khí" chống trùng lặp & an toàn khi re-run.

### 2.3. Delete: theo id, theo `where`, và xoá cả collection

**Delete document** — hai cách:
```python
col.delete(ids=["d3"])              # xoá theo id cụ thể
col.delete(where={"stock": 0})      # xoá theo ĐIỀU KIỆN metadata (mọi hoa hết hàng)
```
🛠️ Bài gốc *chỉ* nói xoá theo id; thực tế **xoá theo `where`** (điều kiện metadata) cực hữu ích (vd "xoá mọi tài liệu của tenant đã rời đi", "xoá mọi SKU hết hàng").

**Delete collection** — khác hẳn:
```python
client.delete_collection("crud_demo")   # XOÁ TOÀN BỘ collection (không thể hoàn tác!)
```
🛠️ Bài gốc lẫn lộn hai việc này. **Xoá document** = bỏ vài hồ sơ; **xoá collection** = vứt cả ngăn tủ. Nhầm là thảm hoạ (xoá nhầm cả kho thay vì một bản ghi).

### 2.4. Code JS 2026 đúng chuẩn (thay hàm bịa của bài gốc)

```javascript
// collection = await client.getCollection({ name, embeddingFunction }); // lấy object trước
async function updateFlower(collection) {
  try {
    await collection.update({                         // gọi TRÊN collection object
      ids: ["flower_2"],
      documents: ["a bright golden tulip"],           // đổi document -> re-embed
      metadatas: [{ stock: 10 }],                     // và/hoặc metadata
    });
    console.log("Update thành công");
  } catch (err) { console.error("Update lỗi:", err); }
}

async function deleteFlower(collection) {
  try {
    await collection.delete({ ids: ["flower_1"] });   // xoá document (KHÔNG phải client.collections.delete)
    console.log("Delete thành công");
  } catch (err) { console.error("Delete lỗi:", err); }
}
// Xoá cả collection: await client.deleteCollection({ name: "my_flower_collection" });
```

### 2.5. 3 lỗi thường gặp (và cách tránh)

1. **Copy hàm bịa `updateDocument`/`client.collections.delete` từ tài liệu cũ.** → Dùng `collection.update(...)`, `collection.delete({ids})`, `client.deleteCollection({name})`.
2. **Dùng `add` trong pipeline re-run → trùng id lỗi.** → Dùng **`upsert`** cho idempotency.
3. **Nhầm xoá document với xoá collection.** → `collection.delete(ids=...)` xoá bản ghi; `client.delete_collection(name)` xoá cả kho. Kiểm tra kỹ trước khi chạy.

### 2.6. ✅ Self-check (Intermediate)

1. Khi nào update *không* cần re-embed? Khi nào *cần*?
2. `upsert` khác `add` và `update` ở điểm nào? Vì sao pipeline production thích upsert?
3. Viết lệnh xoá mọi document có `tenant="acme"` khỏi collection.

---

## Phần 3 — 🔴 ADVANCED (Chuyên sâu)

### 3.1. 🚨 Sự thật: update/delete trong vector DB KHÔNG giống SQL

Đây là phần quan trọng nhất — và bài gốc hoàn toàn im lặng.

Nhớ từ quyển 1: **HNSW không có thao tác "xoá node" sạch sẽ.** Đồ thị HNSW được xây bằng các cạnh nối; gỡ một node giữa đồ thị có thể *phá vỡ khả năng điều hướng* của các node khác. Vì vậy đa số vector DB (gồm ChromaDB) dùng:

- **Soft-delete (tombstone):** khi bạn `delete`, vector *không bị xoá vật lý ngay* — nó được **đánh dấu "đã xoá"** (tombstone). Query bỏ qua nó, nhưng nó **vẫn chiếm bộ nhớ và vẫn nằm trong đồ thị**.
- **Compaction / garbage collection:** định kỳ, một tiến trình nền **dọn dẹp** — xây lại segment/index sạch, loại bỏ tombstone thật sự. ChromaDB (kiến trúc log-structured — quyển 3) dùng **WAL → compaction → segment** và có **garbage collector** với "three-phase deletion" (tách xoá *logic* khỏi xoá *vật lý*).

→ Hệ quả staff phải biết:
1. **Delete không giải phóng bộ nhớ ngay.** Xoá 1 triệu vector không tức thì giảm RAM; phải chờ compaction.
2. **Update = delete + insert.** Đổi một vector ≈ tombstone cái cũ + chèn cái mới → đồ thị phình tombstone.
3. **Workload update/delete nặng → index thoái hoá:** nhiều tombstone → recall giảm, search chậm dần → cần **rebuild định kỳ**. Đây là lý do "chat memory prune liên tục" là workload khó (quyển 1 & 3).

### 3.2. Re-embed — chi phí ẩn của update nội dung

➕ Khi update *document* (không phải metadata), Chroma phải **re-embed** (quyển 5). Với API EF (OpenAI) → tốn tiền + latency mạng mỗi lần; với local EF → tốn GPU/CPU. Update hàng loạt document → **nên batch** re-embed. Và nhớ: nếu bạn *đổi cả embedding model*, đó không phải "update" — đó là **re-embed toàn bộ collection** (quyển 4–5), việc khác hẳn.

### 3.3. Big-O & cấu trúc

| Thao tác | Chi phí | Ghi chú |
|---|---|---|
| `add` / `upsert` (insert) | embed (đắt) + chèn HNSW ~O(log N) | insert incremental được (quyển 1) |
| `update` metadata-only | ~O(1) cập nhật metadata | không re-embed, không đụng đồ thị |
| `update` document | re-embed + (tombstone cũ + insert mới) | đắt nhất; phình tombstone |
| `delete` theo id | O(1) đánh tombstone | **không** giải phóng RAM ngay |
| `delete` theo `where` | O(số match) đánh tombstone | tiện cho mass delete |
| `delete_collection` | ~O(1) logic, dọn nền | vứt cả kho |
| **compaction** (nền) | O(N) xây lại segment | dọn tombstone, khôi phục recall |

### 3.4. Consistency & freshness của update/delete

➕ Kiến trúc log-structured (quyển 3): ghi (gồm update/delete) vào **WAL trước**, index cập nhật **bất đồng bộ**. Hệ quả: sau khi `delete`, có thể có **độ trễ nhỏ** trước khi document *thật sự* biến khỏi kết quả query (index nền chưa cập nhật xong). Với "right to be forgotten" (GDPR), độ trễ này có ý nghĩa pháp lý — cần biết SLA xoá thật là bao lâu, và khi nào dữ liệu *thật sự* rời khỏi WAL/backup.

### 3.5. Edge cases phải xử lý

- **Update một id không tồn tại** → `update` có thể lỗi/no-op tùy version; dùng `upsert` nếu muốn "có thì sửa, không thì tạo".
- **Delete id không tồn tại** → thường no-op (không lỗi), nhưng đừng dựa vào đó để suy ra "đã xoá thành công".
- **Tombstone bloat** → sau nhiều delete/update, index đầy tombstone → recall/latency xấu → lên lịch rebuild/compaction.
- **Delete theo `where` khớp 0 bản ghi** → không lỗi, nhưng bạn tưởng đã xoá → luôn verify bằng `count()`.
- **Xoá nhầm collection** → không hoàn tác; có backup/confirmation trước lệnh `delete_collection`.

---

## Phần 4 — 🟣 STAFF LEVEL (Tư duy hệ thống & lãnh đạo kỹ thuật)

### 4.1. Update/delete ở quy mô lớn — bottleneck & rebuild cadence

➕ Với vài document, update/delete tức thì. Với **tỷ vector**, đây là bài toán vận hành:
- **Mass delete** (xoá 100 triệu vector của một tenant rời đi): đánh tombstone thì nhanh, nhưng **RAM không giảm tới khi compaction** — và compaction O(N) rất nặng. → Lên lịch compaction ngoài giờ cao điểm; theo dõi **tombstone ratio** làm tín hiệu rebuild.
- **Index thoái hoá:** workload update/delete cao → recall trôi (quyển 1) → cần **rebuild định kỳ** (coi như release artifact — quyển 1). Staff phải đặt **cadence rebuild** dựa trên tỉ lệ mutation, không để index "mục" âm thầm.
- **Bottleneck update nội dung = re-embed** (GPU/API), không phải chèn index (quyển 2 & 7).

### 4.2. GDPR / Right to be forgotten — bài toán staff kinh điển

➕ Bài gốc coi delete là "xoá một document". Ở tầm hệ thống, **"xoá dữ liệu người dùng" là bài toán khó & có rủi ro pháp lý**:
- Vector là *dẫn xuất* của dữ liệu gốc → xoá phải gồm **cả vector, document, metadata, VÀ mọi bản sao** (WAL, backup, replica, cache — quyển 3).
- **Soft-delete chưa đủ:** tombstone nghĩa là dữ liệu *vẫn còn vật lý* tới compaction → với GDPR có thể **chưa đạt yêu cầu**; cần đảm bảo compaction/purge thật sự loại bỏ, và xử lý backup.
- **Embedding có thể rò rỉ thông tin:** vector là biểu diễn của nội dung gốc → về lý thuyết có thể *đảo ngược một phần* → xoá vector là bắt buộc, không chỉ xoá text.
→ Staff phải thiết kế **quy trình xoá đầu-cuối** (không chỉ một lệnh `delete`) và biết SLA/pháp lý.

### 4.3. Trade-off: khi nào update tại chỗ vs rebuild lại

- **Update tại chỗ (in-place):** rẻ cho ít thay đổi, nhưng tích luỹ tombstone → thoái hoá.
- **Rebuild định kỳ (blue-green):** dựng index mới sạch từ nguồn, swap — tốn compute nhưng khôi phục recall & giải phóng RAM. → Với collection **mutation cao**, blue-green rebuild theo lịch thường *lành mạnh hơn* update in-place vô hạn.
- **Khi nào KHÔNG lo:** collection gần như tĩnh (ít update/delete) → update in-place là đủ, đừng vẽ vời rebuild.

### 4.4. Reliability, monitoring, cost

- **Monitor:** tombstone ratio, recall drift sau mutation, độ trễ xoá (add→invisible), chi phí re-embed, dung lượng collection (không giảm dù đã delete → dấu hiệu cần compaction).
- **Idempotency:** upsert + id ổn định để pipeline retry an toàn.
- **Failure modes:** delete "thành công" nhưng dữ liệu còn trong backup (GDPR fail), tombstone bloat làm search chậm, update document quên re-embed (nếu bạn tự quản vector), xoá nhầm collection.
- **Cost:** update nội dung = re-embed (đắt); mass delete + compaction = burst compute. Lên kế hoạch, đừng để bất ngờ.

### 4.5. Giải thích cho stakeholder & 🎤 System design mẫu

**Cho stakeholder non-technical:** *"Xoá trong hệ AI không tức thì như xoá một dòng Excel: hệ thống đánh dấu 'đã xoá' ngay (người dùng không còn thấy), nhưng dọn sạch vật lý cần một đợt 'dọn kho' chạy nền. Với yêu cầu pháp lý như GDPR, ta phải đảm bảo đợt dọn đó thực sự xoá cả bản sao lưu — nên 'xoá' là một quy trình, không phải một nút bấm."*

> **Đề: "Người dùng yêu cầu xoá toàn bộ dữ liệu của họ khỏi một hệ RAG có 2 tỷ vector, 500 tenant. Thiết kế quy trình xoá đúng GDPR."**

Hướng trả lời staff:
1. **Làm rõ trước:** SLA xoá (72h?); "dữ liệu" gồm vector + document + metadata + bản sao; có backup/replica không?
2. **Định vị dữ liệu:** id/metadata gắn `user_id` → `collection.delete(where={"user_id": X})` (xoá theo điều kiện). Nếu tách collection-per-tenant (quyển 5) thì gọn hơn.
3. **Soft-delete → hard purge:** delete đánh tombstone ngay (người dùng hết thấy); **kích hoạt compaction/purge** để xoá vật lý trong SLA; xử lý **WAL & backup** (điểm hay bị quên).
4. **Xác minh:** kiểm tra `count`/query để chắc dữ liệu không còn truy hồi được; log để audit.
5. **Scale:** mass delete → tombstone nhanh, compaction O(N) lên lịch; theo dõi tombstone ratio & recall.
6. **Rủi ro embedding:** xoá cả vector (không chỉ text) vì vector có thể rò rỉ nội dung.
7. **Kết bằng trade-off:** soft-delete (nhanh, chưa purge) vs hard-delete tức thì (đắt); collection-per-tenant giúp xoá dễ. → *đánh đổi + tuân thủ*.

---

## Phần 5 — 🎯 CHỐT LẠI ĐỂ ĐI PHỎNG VẤN (Interview Cheatsheet)

### 5.1. Keywords bắt buộc nhớ (định nghĩa 1 dòng)

- **CRUD:** Create/Read/Update/Delete — vòng đời dữ liệu.
- **update:** sửa document/metadata/embedding của id **đã có**; API: `collection.update(...)`.
- **upsert:** update-or-insert, **idempotent**; API: `collection.upsert(...)`. Bài gốc bỏ quên.
- **delete (document):** `collection.delete(ids=...)` hoặc `delete(where=...)`.
- **delete collection:** `client.delete_collection(name)` — xoá cả kho, KHÁC xoá document.
- **re-embed:** sinh lại vector khi *document* đổi; không cần khi chỉ đổi metadata.
- **soft-delete / tombstone:** đánh dấu "đã xoá" thay vì xoá vật lý ngay.
- **compaction / garbage collection:** tiến trình nền dọn tombstone, xây lại index sạch.
- **tombstone bloat:** index đầy tombstone → recall/latency xấu → cần rebuild.
- **idempotent:** chạy lại nhiều lần vẫn ra cùng kết quả (upsert).
- **async / await / try-catch:** JS chung cho thao tác bất đồng bộ + xử lý lỗi (không đặc thù Chroma).
- **right to be forgotten (GDPR):** nghĩa vụ xoá thật sự cả vector + bản sao.

### 5.2. Core concepts (nếu chỉ nhớ mấy điều này)

1. **API thật:** `collection.update/upsert/delete` (trên object), `client.delete_collection` (cả kho). Các hàm `updateDocument`/`client.collections.delete` của bài gốc là **bịa**.
2. **upsert > update** cho pipeline: idempotent, an toàn khi re-run.
3. **Update metadata rẻ (không re-embed); update document đắt (re-embed).**
4. **Delete document ≠ delete collection.** Nhầm là thảm hoạ.
5. **Vector DB không xoá như SQL:** HNSW → **soft-delete + tombstone + compaction**; delete không giải phóng RAM ngay.
6. **Update = delete + insert** → phình tombstone → **rebuild định kỳ**.
7. **GDPR = quy trình xoá đầu-cuối** (vector + document + metadata + WAL + backup), không phải một lệnh.
8. **Ở scale:** theo dõi tombstone ratio & recall drift; lên lịch compaction/rebuild; re-embed là chi phí ẩn.

### 5.3. Ideas / mental models

- **"Xoá = dán nhãn 'đã xoá', dọn kho tính sau"** (soft-delete → compaction).
- **"Update = xoá cũ + thêm mới"** → hiểu vì sao phình tombstone.
- **"upsert là nút an toàn cho pipeline chạy lại"** (idempotent).
- **"Metadata rẻ, nội dung đắt"** (re-embed).
- **"Xoá document là rút một hồ sơ; xoá collection là vứt cả ngăn tủ."**
- **"GDPR-delete là quy trình, không phải nút bấm"** (cả bản sao lưu).
- **"Index cũng mục"** — mutation cao thì phải rebuild, đừng để thoái hoá âm thầm.

### 5.4. Code cần thuộc lòng

**(1) CRUD đầy đủ — Python (interviewer hay bắt viết):**
```python
col.add(ids=["1"], documents=["v1"], metadatas=[{"s":1}])       # Create
col.update(ids=["1"], metadatas=[{"s":2}])                       # Update metadata (không re-embed)
col.update(ids=["1"], documents=["v2"])                          # Update document (re-embed)
col.upsert(ids=["2"], documents=["v3"])                          # Upsert (idempotent)
col.delete(ids=["1"])                                            # Delete document theo id
col.delete(where={"s": 2})                                       # Delete theo điều kiện
client.delete_collection("name")                                 # Delete cả collection
```

**(2) Sửa API bịa của bài gốc:**
```javascript
// SAI:  updateDocument(collectionName, id, doc, meta)
// ĐÚNG: await collection.update({ ids:[id], documents:[doc], metadatas:[meta] })
// SAI:  client.collections.delete(collectionName, id)
// ĐÚNG: await collection.delete({ ids:[id] })          // xoá document
//       await client.deleteCollection({ name })        // xoá collection
```

**(3) Upsert idempotent (pattern pipeline):**
```python
for chunk in chunks:                                    # chạy lại an toàn
    col.upsert(ids=[f"{doc_id}:{chunk.i}"], documents=[chunk.text],
               metadatas=[{"doc_id": doc_id, "updated_at": now}])
```

### 5.5. Câu hỏi phỏng vấn thường gặp + gợi ý trả lời (có câu bẫy)

1. **"Update và delete trong vector DB có giống SQL không?"** *(câu bẫy cốt lõi)* → **Không.** HNSW không xoá node sạch → **soft-delete (tombstone) + compaction**. Delete không giải phóng RAM ngay; update = delete + insert → phình tombstone → cần rebuild.

2. **"`updateDocument(collectionName, id, ...)` — đúng API Chroma không?"** *(câu bẫy — bài gốc bịa)* → Không. Đúng là `collection.update(ids=..., documents=..., metadatas=...)`, gọi trên collection object.

3. **"upsert khác update thế nào? Khi nào dùng?"** → upsert = update-or-insert, **idempotent** → an toàn khi pipeline re-run/retry; update thuần fail nếu id chưa có.

4. **"Update một document có tốn re-embed không?"** *(câu bẫy)* → Tuỳ: đổi **metadata** → không; đổi **nội dung document** → **có** (re-embed, tốn GPU/API).

5. **"Xoá 1 tỷ vector — RAM giảm ngay chứ?"** *(câu bẫy)* → Không. Delete chỉ đánh tombstone; RAM giảm sau **compaction** (O(N), nặng). Theo dõi tombstone ratio.

6. **"Thiết kế xoá dữ liệu người dùng đúng GDPR."** *(câu bẫy staff)* → Quy trình đầu-cuối: xoá vector+document+metadata+WAL+backup; soft-delete → hard purge trong SLA; xoá cả vector (rò rỉ nội dung); audit log. Không phải một lệnh `delete`.

7. **"Delete document vs delete collection?"** → `collection.delete(ids)` bỏ bản ghi; `client.delete_collection(name)` vứt cả kho. Nhầm = mất toàn bộ dữ liệu.

### 5.6. One-liner đắt giá

- *"Trong vector DB, **delete là dán tombstone, không phải xoá vật lý** — RAM chỉ giảm sau compaction; nên tôi theo dõi tombstone ratio để biết khi nào rebuild."*
- *"**Update = delete + insert** trong HNSW — mutation cao thì index mục dần, phải rebuild định kỳ như một release."*
- *"Pipeline của tôi luôn dùng **upsert với id ổn định** — idempotent, chạy lại mười lần vẫn đúng."*
- *"Update **metadata rẻ, update nội dung đắt** vì phải re-embed — nên tôi tách hai loại thay đổi."*
- *"GDPR-delete là **một quy trình** (vector + document + backup + purge), không phải một nút — và phải xoá cả vector vì nó rò rỉ nội dung gốc."*
- *"`updateDocument`/`client.collections.delete` là **hàm bịa** — API thật gọi trên collection object: `.update()`, `.delete()`, còn `client.delete_collection()` mới xoá cả kho."*

---

### 📌 Phụ lục: những chỗ bài giảng gốc sai / thiếu

1. **🚨 `updateDocument(collectionName, id, ...)`** → API bịa; đúng là **`collection.update(ids, documents, metadatas)`** trên collection object.
2. **🚨 `client.collections.delete(collectionName, id)`** → API bịa; đúng là **`collection.delete({ids})`** (xoá document) hoặc **`client.deleteCollection({name})`** (xoá collection).
3. **🚨 Bỏ hẳn `upsert`** — thao tác idempotent quan trọng nhất cho pipeline production.
4. **Không phân biệt update metadata (không re-embed) vs update document (re-embed).**
5. **Lẫn lộn xoá document với xoá collection.**
6. **Giấu sự thật soft-delete/tombstone/compaction:** update/delete trong vector DB *không* giống SQL; delete không giải phóng RAM ngay; mutation cao → cần rebuild.
7. **Không nhắc delete theo `where`**, GDPR/right-to-be-forgotten, hay chi phí re-embed.

> **Phần đúng & giá trị của bài gốc:** giải thích **`async`/`await`** (hàm chờ tác vụ hoàn tất) và **`try/catch` + `console.log`/`console.error`** (xử lý & log lỗi) là **chính xác** và là thói quen tốt; nhấn mạnh **id là khoá để update/delete** đúng bản chất; khung "chuẩn bị dữ liệu → gọi thao tác → log kết quả" hợp lý. Giữ những phần đó, thay toàn bộ API bịa bằng API thật 2026, thêm `upsert` + phân biệt metadata/document + delete-by-where, và bổ sung cơ chế soft-delete/compaction + tư duy GDPR/scale mà bài gốc bỏ trống.
