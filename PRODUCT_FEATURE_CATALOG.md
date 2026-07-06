# ICP — Product Feature Catalog (Bản trình bày cho Nhà đầu tư)

> **ICP — Intelligent Commerce Platform**: nền tảng **thương mại thông minh full-chain cho SME Việt Nam** — người mua · shop bán lẻ · tiệm tạp hoá · nhà phân phối · nhà cho vay · host livestream cùng vận hành trên MỘT hệ, với lõi **AI thương mại** (tìm kiếm thông minh · tối ưu quyết định · dự báo nhu cầu).
>
> **Mục đích tài liệu:** để nhà đầu tư nắm **(1) miền nghiệp vụ** (ai dùng, tính năng làm gì, tại sao có giá trị) + **(2) công nghệ đi kèm** (làm bằng gì, có bền/scale không) → đủ cơ sở đánh giá **có đáng đầu tư**.
> **Nguyên tắc số liệu:** cột **%** là **ước lượng thận trọng** (conservative) — chúng tôi cố tình báo cáo dè dặt, under-promise / over-deliver. Cột **Trạng thái** phân biệt rõ *đã chạy thật (🟢)* vs *lộ trình (🔵/🟠)*, KHÔNG trộn lẫn.
> **Cập nhật:** 2026-07-06 · living document.

---

## 🔑 Quy ước trạng thái

| Ký hiệu | Nghĩa | Diễn giải cho nhà đầu tư |
|---|---|---|
| 🟢 **DONE** | Đã chạy thật, có test tự động + hạ tầng production-grade | Rủi ro thực-thi đã được khử |
| 🟡 **PARTIAL** | Có khung + chạy một phần, còn mảng cần hoàn thiện | Đường đi rõ, không phải nghiên cứu |
| 🔵 **PLANNED** | Đã thiết kế + phụ thuộc dữ liệu sẵn, chưa code | Lộ trình xác định |
| 🟠 **NEEDED** | Bắt buộc cho VN hoặc để ngang Amazon/Shopee, nằm trong tầm nhìn đầy đủ | Backlog thị trường |

*(% = mức hoàn thành so với **tầm nhìn đầy đủ** của tính năng đó khi hoàn chỉnh 100%.)*

---

## 0. Tóm tắt điều hành (Executive Summary)

**ICP không phải "một Shopee nữa".** ICP là **hệ điều hành thương mại thông minh** cho hàng triệu tiệm tạp hoá / SME Việt: nơi shop **bán online + livestream**, **nhập hàng từ nhà phân phối qua nền tảng**, được **AI cố vấn nhập gì – bán giá nào – trữ bao nhiêu**, và **tiếp cận vốn theo doanh số thật**. Marketplace là **nền sinh dữ liệu**; **trí tuệ (AI) là lõi giá trị**.

**3 lý do đáng đầu tư:**

1. **ĐÃ CHẠY THẬT — không phải slide.** Hàng trăm vòng phát triển · kiến trúc **microservices** (đã tách 8 service độc lập, mỗi service một cơ sở dữ liệu riêng) · cách ly dữ liệu giữa các shop **thực thi ở tầng database** · thanh toán VN (Momo/VNPay/ZaloPay/COD) với xác thực chữ ký + chống tính-tiền-hai-lần · tuân thủ pháp lý (đồng ý dữ liệu / quyền chủ thể / nhật ký kiểm toán bất biến) xây sẵn từ ngày đầu · quy trình kiểm-tra-chất-lượng tự động chặn code lỗi. ⇒ **khử rủi-ro-thực-thi** — thứ giết phần lớn startup trước khi ra thị trường.

2. **MOAT = mạng lưới nhiều shop × dữ liệu độc quyền.** ICP nắm **giá vốn + tồn kho + dòng tiền tín dụng** của hàng nghìn shop → sinh 3 "10x" mà không đối thủ nào gộp đủ: **dự cảm nhu cầu toàn mạng** (báo sốt cầu sớm 1–2 tuần) · **đường ray nhập-hàng + bộ não gộp** (bắt giá vốn tự động + trí tuệ chung) · **xếp hạng "vốn-bảo-chứng"** (ưu tiên hàng có vốn hậu thuẫn, không bán quá). *Shopee có data nhưng không có vòng lặp vốn; ngân hàng có vốn nhưng không có data; công cụ đơn lẻ không có mạng lưới — chỉ ICP full-chain có cả ba.*

3. **Nhiều đường doanh thu xếp sẵn trong kiến trúc.** Phí SaaS + **phí cho vay** (theo %-doanh-số) + **quảng cáo nội sàn** + **B2B phân phối** — các đường tiền đã được chừa chỗ trong thiết kế, không phải đập đi xây lại.

**Mức hoàn thành tổng thể** (so với tầm nhìn đầy đủ: ngang Amazon + thuần VN + AI + B2B + cho vay):

```
Nền tảng & Kiến trúc microservices  ██████░░░░░░░░░░░░░░  ~28%   (đã de-risk, rất vững)
Lõi thương mại buyer↔merchant       █████░░░░░░░░░░░░░░░  ~24%
Khám phá AI (search/voice/image)    ████░░░░░░░░░░░░░░░░  ~20%
Trí tuệ — 3 bài toán ML lõi         █░░░░░░░░░░░░░░░░░░░  ~4%   (nghiên cứu + bắt dữ liệu xong, model đang tới)
Data-capture & nhà phân phối (B2B)  ██░░░░░░░░░░░░░░░░░░  ~8%
Tài chính & thanh toán VN           █████░░░░░░░░░░░░░░░  ~22%
Tuân thủ & Quản trị                 ██████░░░░░░░░░░░░░░  ~28%
────────────────────────────────────────────────────────────
TỔNG (toàn bộ tầm nhìn)             ███░░░░░░░░░░░░░░░░░  ~14%
Lõi thương mại đã de-risk (MVP)     █████░░░░░░░░░░░░░░░  ~26%
```

> **Thông điệp:** phần **nền + lõi + kiến trúc + bắt-dữ-liệu** đã thật và vững; phần lớn giá trị còn lại là **lộ trình thực-thi xác định** (không phải rủi ro nghiên cứu). 3 bài toán ML — điểm khác biệt lớn nhất — đã **nghiên cứu sâu + bắt dữ liệu độc quyền**, model là bước kế.

---

## 🧭 Bối cảnh & Bài toán kinh doanh (đọc trước tiên)

> *Mục này viết bằng ngôn ngữ kinh doanh thuần — để nhà đầu tư nắm rõ **ICP giải bài toán gì, cho ai, kiếm tiền ra sao** trước khi đọc chi tiết kỹ thuật.*

**Thị trường:** Việt Nam có **hàng triệu tiệm tạp hoá, shop nhỏ và SME bán lẻ** — xương sống của nền bán lẻ (kênh truyền thống vẫn chiếm ~60% thị trường). Họ vận hành gần như **thủ công**, và đó chính là cơ hội.

**Nỗi đau — 4 vết thương mỗi ngày của một chủ shop:**
1. **Không biết lời-lỗ thật.** Giá vốn ghi sổ tay / hoá đơn giấy → định giá theo cảm tính, không biết món nào thực sự có lãi.
2. **Trữ hàng theo cảm tính.** Nhập dư thì đọng vốn / ế; nhập thiếu thì hết hàng, mất khách. Không có ai dự báo giúp.
3. **Khó tiếp cận vốn.** Ngân hàng đòi thế chấp + lịch sử tín dụng — shop nhỏ không có → phải vay nóng lãi cao.
4. **Bán online rời rạc.** Mỗi sàn (Shopee/Lazada/TikTok) một nơi, chỉ giúp *bán* chứ không giúp *quản trị*; việc *nhập hàng* lại nằm ở kênh tách biệt hoàn toàn (Zalo/điện thoại với nhà phân phối).

**Hôm nay họ chắp vá bằng:** sổ tay + Excel + vài sàn TMĐT (chỉ để bán) + tin nhắn Zalo với nhà phân phối + vay nóng. **Không có "một hệ" nào nối cả chuỗi lại.**

**ICP là gì:** **một hệ điều hành thương mại** số-hoá **cả chuỗi** cho shop nhỏ — **nhập hàng → bán (online + livestream) → giao → thu tiền → vay vốn** — với **AI cố vấn xuyên suốt** (nhập gì · bán giá nào · trữ bao nhiêu · vay bao nhiêu). Không phải "một sàn nữa"; ICP là **lớp vận-hành + trí-tuệ** đặt lên trên cả đời sống kinh doanh của shop.

**Chuỗi giá trị ICP số-hoá (khép kín — đây là điểm khác biệt cốt lõi):**
```
Nhà phân phối ─[NHẬP]→ SHOP ─[BÁN online/live]→ Người mua ─[GIAO]→ ─[THU TIỀN]→ ─[VAY theo doanh số]→
        ↑______________ AI cố vấn + dữ liệu độc quyền chảy suốt cả vòng ______________↑
```
> Đối thủ chỉ nắm **một mắt xích**: Shopee nắm "bán" (không có nhập/vốn) · ngân hàng nắm "vốn" (không có dữ liệu bán) · công cụ quản-lý-bán-hàng nắm "quản trị một shop" (không có mạng lưới, không có vốn). **Chỉ ICP nắm cả vòng** → dữ liệu độc quyền + hiệu-ứng-mạng-lưới không sao chép được.

**Vì sao là BÂY GIỜ (timing):** smartphone phủ khắp · thanh toán số bùng nổ (VietQR) · **livestream commerce** thành kênh bán #1 · **luật hoá-đơn-điện-tử bắt buộc** đang ép shop số-hoá · khoảng-trống-tín-dụng SME khổng lồ. Cửa sổ để "đặt hệ điều hành" cho lớp bán lẻ này đang mở — và đóng dần khi ai đó chiếm trước.

**Ai trả tiền cho ICP (mô hình doanh thu — nhiều đường, cùng một tập dữ liệu):**
- **Shop** trả **phí SaaS** (công cụ + AI cố vấn) + **take-rate** trên đơn bán.
- **Nhà phân phối** trả **phí B2B** (kênh phân phối số + được thấy cầu hạ nguồn).
- **Nhà cho vay** **chia phí cho vay** (ICP cấp tín-hiệu-rủi-ro từ dữ liệu giao dịch thật).
- **Người bán** trả **phí quảng cáo** nội sàn.

→ **Bốn đường tiền, một tập dữ liệu độc quyền.** Đó là lý do ICP không phải "một Shopee nữa" mà là một **hệ điều hành thương mại thông minh** cho SME Việt.

---

## 1. 👥 Hệ đa-bên — Ai vận hành trên ICP (multi-sided market)

> Nhà đầu tư cần thấy: ICP **không phải marketplace 2 bên đơn giản** (mua–bán). Đây là **hệ nhiều bên** khép kín cả chuỗi cung ứng bán lẻ Việt Nam — từ **nhà phân phối → shop → người mua**, cộng thêm **vốn** và **giao vận**. Càng nhiều bên tham gia, dữ liệu càng độc quyền, moat càng sâu — và mỗi bên là một **đường doanh thu**.

| # | Vai (actor) | Là ai / làm gì | Giá trị họ nhận từ ICP | Đường tiền | Trạng thái |
|---|---|---|---|---|---|
| 1 | **Người mua lẻ (Buyer)** | Người tiêu dùng cuối, tài khoản dùng chung toàn sàn, mua từ mọi shop | Tìm–mua–thanh toán–theo dõi đơn; mua bằng chữ / ảnh / giọng nói | (nền cầu) | 🟢 DONE |
| 2 | **Người mua doanh nghiệp (B2B buyer)** | Chính shop/tạp hoá khi đi **nhập sỉ** từ nhà phân phối | Đặt sỉ, giá theo số lượng, mua-trả-sau (công nợ) | GMV B2B | 🔵 PLANNED |
| 3 | **Chủ shop / Người bán (Merchant)** | Tổ chức bán lẻ (chủ + nhân viên), 1 chủ có thể nhiều shop | Đăng bán, quản đơn, tồn kho, **AI cố vấn nhập/giá**, vay vốn | Phí SaaS + take-rate | 🟢 DONE |
| 4 | **Tiệm tạp hoá O2O** | Cửa hàng offline vừa bán tại quầy vừa lên sàn (bán lẻ truyền thống số-hoá) | POS-lite, đồng bộ tồn offline↔online, cố vấn "lấy gì off truck" | Phí SaaS | 🔵 PLANNED |
| 5 | **SME tự sản xuất** | Vừa sản xuất vừa bán (thương hiệu nhỏ, đặc sản vùng) | Vừa là người bán vừa là nguồn cung, quản lý thành phẩm | Phí SaaS + B2B | 🔵 PLANNED |
| 6 | **Nhà phân phối / Cung cấp sỉ (Supplier)** | Bán sỉ cho shop; **nguồn giá vốn thật** của cả hệ | Kênh phân phối số, đơn nhập định kỳ, thấy cầu hạ nguồn | Take-rate B2B | 🟡 PARTIAL |
| 7 | **Nhà cho vay (Lender)** | Bên cấp vốn (nhiều đối tác, có giấy phép) cho shop theo doanh số | Chấm điểm rủi ro từ dữ liệu giao dịch thật (không cần FICO) | Chia phí cho vay | 🔵 PLANNED |
| 8 | **Đối tác vận chuyển (Logistics/3PL)** | GHN / GHTK / J&T / Ahamove — giao hàng chặng cuối | Luồng đơn giao, in vận đơn, đối soát COD | Phí tích hợp | 🟠 NEEDED |
| 9 | **KOC / Host livestream** | Người bán qua livestream / ảnh hưởng — kênh #1 TMĐT VN | Công cụ live-commerce, giỏ hàng trong live, hoa hồng | Hoa hồng + ads | 🟠 NEEDED |
| 10 | **Nhân viên nền tảng (Platform staff)** | Admin / kiểm toán viên / hỗ trợ / quản-lý-ngành-hàng | Điều phối sàn, kiểm toán chéo tenant, xử lý vi phạm | (vận hành) | 🟢 DONE |

**~24 quan hệ (cạnh) giữa các bên — chuỗi giá trị khép kín:**
- **Mua–bán:** buyer → merchant (đặt đơn, thanh toán, đánh giá, trả hàng) · buyer → KOC (mua trong livestream).
- **Cung ứng:** supplier → merchant (bán sỉ, hoá đơn nhập, công nợ) · merchant → supplier (đặt nhập định kỳ).
- **Vốn:** lender → merchant (cấp vốn theo doanh số) · ICP → lender (cung cấp tín hiệu rủi ro từ dữ liệu thật).
- **Giao vận:** merchant → logistics (tạo đơn giao) · logistics → buyer (giao chặng cuối) · logistics ↔ platform (đối soát COD).
- **Dòng tiền ngược:** hoàn tiền, trả hàng, quyết toán payout cho seller, đối soát.
- **Điều phối:** platform ↔ mọi bên (kiểm duyệt, audit, phân xử tranh chấp, quảng cáo).

> **Ý nghĩa đầu tư:** mỗi bên tham gia làm **dữ liệu độc quyền dày thêm** (cost từ supplier, doanh số cho lender, hành vi từ buyer) và **mở một đường doanh thu**. Đây là lý do full-chain > marketplace thuần: đối thủ B2C không có cost/vốn, đối thủ B2B không có cầu cuối.

---

## 2. ⭐ Kiến trúc Microservices (vì sao bền & scale)

> Điểm nhà đầu tư kỹ thuật cần thấy: ICP **KHÔNG phải một khối phần mềm mong manh (monolith)**. Hệ đã (và đang) được **tách thành các service độc lập theo miền nghiệp vụ**, mỗi service **có database riêng**, giao tiếp qua **sự-kiện + API bền bỉ** — nền để scale từng phần, deploy độc lập, nhiều đội phát triển song song.

**Tiến độ tách — đã kiểm chứng (không phải kế hoạch trên giấy):**
- **Mục tiêu:** ~29 microservice (6 trí-tuệ + ~16 nền nghiệp-vụ + 7 lớp giao-diện). **8 service lõi ĐÃ TÁCH XONG** — định-danh · tổ-chức · khách-hàng · danh-mục-sản-phẩm · niêm-yết-giá · tồn-kho · nhập-hàng · thu-thập-hành-vi — mỗi cái có **database riêng, cách ly dữ liệu giữa shop, và bộ test tự động trên database thật**. Dịch vụ điều-phối-AI đã **chuyển sang nền async hiện đại** (hiệu năng cao hơn cho luồng chat/stream).
- **Bộ công cụ chuẩn hoá ("paved-road"):** tách 1 service mới = 1 lệnh dựng khung (giám-sát / log / gọi-chéo-bền-bỉ / cách-ly-tenant / đóng-gói container / CI / migration database) → mở rộng **rẻ + đồng nhất**, không phải viết tay lại từng service.
- **Kỷ luật kỹ thuật:** mọi quyết định lớn đều được **ghi lại thành văn bản** kèm lý do + đánh đổi + phương án bị loại → minh bạch tuyệt đối cho quá trình thẩm định (due-diligence) kỹ thuật. Mọi thay đổi chạm dữ liệu đều có test cách-ly-tenant "báo đỏ khi sai".

**Nguyên tắc kiến trúc (đã áp dụng thật):**
- **Chia theo miền nghiệp vụ** (Domain-Driven) — mỗi service một database, migration riêng, tiến hoá độc lập.
- **Cách ly dữ liệu giữa các shop tuyệt đối:** thực thi ở **tầng database (Row-Level Security)** — 1 shop **không bao giờ** đọc/ghi được dữ liệu shop khác, kiểm chứng bằng test "báo đỏ khi rò". Đây là điều kiện SỐNG-CÒN để bán SaaS đa-shop và để tuân thủ pháp lý.
- **3 "đường ray" hạ tầng dùng chung:** (A) **Xương sống sự-kiện** — dữ liệu chảy giữa các service đáng tin cậy (đặt hàng → trừ kho → nuôi AI), dùng chuẩn công nghiệp (Debezium + Kafka/Redpanda); (B) **Ngữ cảnh tenant** — mọi request mang danh tính shop; (C) **Gọi-chéo-service bền bỉ** — có timeout + thử lại + ngắt-mạch (circuit-breaker) để một service lỗi không kéo sập dây chuyền.
- **Chiến lược "bóp nghẹt" (strangler):** tách dần khỏi khối cũ qua **lớp trung gian bật/tắt được** — an toàn, có thể quay lui, không downtime, không "đập đi xây lại".

**Bản đồ service (lộ trình 5 pha — ưu tiên dữ liệu trước):**

| Pha | Service | Vai trò nghiệp vụ | Công nghệ | Trạng thái |
|---|---|---|---|---|
| **P1** | **Định danh (identity)** | Xác thực, đăng nhập, token, 1 tài khoản nhiều shop | NestJS · Postgres · JWT · mã hoá mật khẩu memory-hard | 🟢 tách xong |
| **P1** | **Tổ chức (organization)** | Shop, thành viên, phân quyền owner/staff/admin | NestJS · Postgres · cách ly cấp-dòng | 🟢 tách xong |
| **P1** | **Khách hàng (customer)** | Hồ sơ người mua: sổ địa chỉ, đã-lưu, sở thích | NestJS · Postgres · cách ly cấp-dòng | 🟢 tách xong |
| **P1** | **Danh mục SP (catalog)** | Product master toàn cục dùng chung (kiểu ASIN của Amazon) | NestJS · Postgres · hướng-sự-kiện | 🟢 tách xong |
| **P1** | **Niêm yết & Giá (offer)** | Listing của từng người bán + lịch sử giá theo thời gian + Buy-Box | NestJS · Postgres · cách ly cấp-dòng | 🟢 tách xong |
| **P2** | **Nhập hàng (procurement)** | **Đọc hoá đơn nhà phân phối bằng AI → lịch sử giá vốn thật** | NestJS · Postgres · AI-vision (Gemini) đọc hoá đơn | 🟢 tách xong |
| **P2** | **Thu thập hành vi (tracking)** | Ghi nhận hành vi + lượt-hiển-thị quy mô lớn (nuôi AI) | NestJS · Postgres phân-mảnh theo tháng · thu-100% | 🟢 tách xong |
| **P2** | **Tồn kho (inventory)** | Tồn + sổ-cái-kho + sự-kiện-hết-hàng | NestJS · Postgres · khoá nguyên-tử chống bán-quá | 🟢 tách xong |
| **P3** | **ML-runtime (apps/ml)** | Nền chạy model: dự báo / xếp hạng / chấm điểm | Python · numpy/pandas · statsforecast · Chronos-2 · PyMC | 🔵 vừa khởi động (thiết kế) |
| **P3** | **ML-platform** | Kho đặc-trưng đúng-thời-điểm · sổ model · phát-hiện-lệch · thử-nghiệm bóng | Python · MLOps stack | 🔵 planned |
| **P3** | **Tìm kiếm & Gợi ý (BT01)** | Truy hồi + xếp hạng + livestream + tiếng Việt | Vespa · embedding riêng · học-xếp-hạng (LTR) | 🔵 planned |
| **P3** | **Dự báo (BT03)** | Dự báo cầu theo xác suất (dải P10/P50/P90) | Chronos-2 · AutoGluon · Bayes phân-tầng | 🔵 planned |
| **P3** | **Quyết định (BT02)** | Copilot cố vấn nhập / giá / khuyến mãi cho shop | Toán tồn-kho (newsvendor) · ML nhân-quả · tối ưu | 🔵 planned |
| **P3** | **Chấm điểm rủi ro** | Nền chung: tín dụng + gian-lận + độ-chắc-giao-hàng | Gradient-boosting · hiệu-chỉnh xác suất · đồ-thị | 🔵 planned |
| **P4** | **Giao dịch lõi** (giỏ · đơn · thanh toán · quyết toán · trả hàng · giao vận · cho vay) | Trái tim giao dịch nhiều bước (saga) — khó nhất, làm sau | NestJS · saga · thanh toán chống-trùng | 🟡 lõi có trong khối cũ, đang tách dần |
| **P5** | **Lớp lá & kênh** (đánh giá · thông báo · các giao-diện buyer/seller/Zalo/live) | Trải nghiệm đầu-cuối cho từng bên | NestJS · Next.js · Zalo API | 🟠 planned |

**Nền chạy chung:** cổng-vào (BFF/edge, luồng thời-gian-thực) · điều-phối-AI (async, LangGraph) · tool-server cho AI · giao diện web (Next.js) · các worker nền (đồng bộ sự-kiện · thanh toán · dọn dẹp · kiểm toán).

| Tính năng nền tảng | Trạng thái | % | Mô tả nghiệp vụ (business) | Công nghệ |
|---|---|---|---|---|
| Cách ly dữ liệu giữa shop (thực thi tầng database, mặc-định-đóng) | 🟢 DONE | 32% | Mỗi shop là 1 "khoang" dữ liệu kín — điều kiện sống-còn để bán SaaS đa-shop + tuân thủ pháp lý; sai một dòng là rò dữ liệu cho đối thủ | Postgres Row-Level Security · ngữ cảnh tenant · test cách-ly đầu-cuối |
| Tách microservices (mỗi service 1 database, tách dần an toàn) | 🟢 DONE | 23% | Scale/độc lập từng miền, nhiều đội làm song song, deploy không downtime — nền lên quy mô hàng triệu shop | NestJS · Postgres per-service · Debezium/Kafka · lớp-trung-gian bật/tắt |
| Xác thực & phân quyền (đăng nhập, 1 tài khoản nhiều shop, vai trò) | 🟢 DONE | 28% | Đăng nhập, chuyển giữa các shop, vai trò owner/staff/admin/supplier | JWT · mã hoá mật khẩu memory-hard · guard theo vai trò |
| Chống tính-tiền-hai-lần / lặp sự-kiện (idempotency) | 🟢 DONE | 32% | Không bao giờ tính tiền 2 lần dù mạng lỗi/thử-lại — niềm tin thanh toán | Redis · khoá chống-trùng · xử-lý-đúng-một-lần |
| Xương sống sự-kiện (đặt hàng → trừ kho → nuôi AI) | 🟢 DONE | 28% | "Hệ thần kinh" truyền sự-kiện giữa service đáng tin cậy | Ghi-sự-kiện cùng-giao-dịch · Debezium · Kafka/Redpanda |
| Giám sát xuyên-service (log / trace / metric) | 🟢 DONE | 28% | Nhìn xuyên suốt 1 request qua nhiều service — vận hành production nghiêm túc | OpenTelemetry · bộ giám-sát Grafana |
| Hợp đồng API FE↔BE tự-sinh (chống lệch) | 🟢 DONE | 32% | FE và BE không bao giờ lệch hợp đồng — giảm bug tích hợp | OpenAPI tự-sinh · kiểm-lệch trong CI |
| Cổng chất lượng CI/CD (lint/test/typecheck/coverage) | 🟢 DONE | 30% | Chặn code sai/thiếu test lên nhánh chính — chất lượng có kỷ luật | GitHub Actions · ngưỡng coverage |
| Nền tảng sinh dữ-liệu-demo (first-class) | 🟢 DONE | 23% | **Tự sinh dữ liệu demo thật** để showcase hệ thống cho đối tác/nhà đầu tư + test mọi tính năng không-giả-lập | Bộ sinh dữ liệu đăng-ký · giao diện admin · sinh/xoá/khôi-phục |
| Chịu tải cao & mở rộng ngang (giai đoạn scale) | 🔵 PLANNED | 3% | Gộp kết nối database · cụm tìm-kiếm nhiều-node · nhiều bản-sao | Bộ gộp kết nối · Vespa cluster |

---

## 3. ⭐⭐ Trí tuệ — 3 bài toán ML lõi (khác biệt cạnh tranh lớn nhất)

> **Đây là trái tim giá trị của ICP.** Ba bài toán này đã được **nghiên cứu rất sâu** (đúng chuẩn kỹ thuật Amazon/bigtech, đã sửa hết các sai lầm blueprint ngây thơ) và **đang bắt dữ liệu độc quyền không-thể-lấy-lại**. Model là bước kế — nhưng **dữ liệu + nghiên cứu = phần khó và phần moat**, và ICP đã nắm.

**Vì sao "bắt dữ liệu là việc phải làm TRƯỚC":** những tín hiệu như **cầu bị từ chối do hết hàng**, **giá vốn thật từ hoá đơn nhập**, **giá theo thời gian**, **lịch chiến dịch (Tết / 9.9 / ngày lương)**, **lượt hiển thị + xu hướng click**, **sự-kiện livestream**, **tầng ngôn ngữ tiếng Việt** — *không bắt hôm nay là mất vĩnh viễn*. ICP đã dựng đường ống bắt các tín hiệu này TRƯỚC khi có model. Đây là lý do các service **tồn-kho / nhập-hàng / thu-thập-hành-vi** được tách ra và hoàn thiện sớm.

### BT01 — Tìm kiếm & Gợi ý thông minh (Smart Search + Recommendation)

*Ai được lợi: người mua tìm đúng hàng nhanh hơn → chuyển đổi cao hơn; người bán có hàng đúng-vốn-đúng-tồn được ưu tiên.*

| Hạng mục | Trạng thái | % | Mô tả nghiệp vụ (business) | Công nghệ |
|---|---|---|---|---|
| Tìm kiếm lai (chữ + ảnh + ngữ nghĩa) | 🟢 DONE | 28% | Người mua tìm bằng gõ chữ hoặc ảnh, trả kết quả liên quan | Vespa · so-khớp từ-khoá + ngữ-nghĩa hình-ảnh + tái-xếp-hạng |
| Tìm/mua bằng **giọng nói** (voice-commerce) | 🟡 PARTIAL | 18% | Nói "mua 2 chai dầu ăn" → hệ hiểu + thêm giỏ; hợp chủ tạp hoá ít gõ phím | Nhận-dạng-giọng-nói (Gemini) · LangGraph · luồng thời-gian-thực |
| Gợi ý bằng **ảnh** (visual reco) | 🟡 PARTIAL | 18% | Chụp sản phẩm → tìm hàng tương tự | AI-vision · ngữ-nghĩa hình-ảnh · tìm lân-cận Vespa |
| Gợi ý đa-tín-hiệu ("mua cùng", "liên quan", xu-hướng) | 🟡 PARTIAL | 15% | Tăng giá-trị mỗi giỏ hàng bằng gợi ý đúng ngữ cảnh | Trộn 3 tín hiệu · bảng tổng-hợp precompute |
| Gợi ý 2 tầng (sinh ứng-viên → **xếp hạng đa mục tiêu**) | 🔵 PLANNED | 3% | Chuẩn Amazon thật: sinh ứng viên rồi xếp hạng theo nhiều mục tiêu (liên quan + tỉ-lệ-click + tỉ-lệ-mua) kèm ràng buộc đa dạng | Embedding riêng · học-xếp-hạng · thuật-toán bandit |
| ⭐ **Tầng ngôn ngữ tiếng Việt hạng nhất** | 🔵 PLANNED | 3% | Nơi "hiểu đúng ý" thật sự khó — moat 10 năm của Shopee: hiểu teencode, thiếu dấu ("ao thun" → "áo thun"), từ đồng nghĩa vùng miền | Mô hình ngôn ngữ Việt (PhoBERT / e5-đa-ngữ) · tách-từ tiếng Việt |
| ⭐⭐ **Xếp hạng "vốn-bảo-chứng" (fulfillment-certainty)** | 🔵 PLANNED | 3% | **10x MOAT**: ICP cấp vốn + tài trợ tồn kho → biết SKU nào "có-vốn-hậu-thuẫn + còn-hàng, không bán-quá" → ưu tiên hàng chắc-chắn-giao-được; livestream drop hàng "vốn bảo chứng" | Tín hiệu chấm-điểm-rủi-ro + tài-trợ-tồn-kho |
| Trợ lý mua sắm Gen-AI (kiểu "Rufus" của Amazon) | 🟠 NEEDED | 0% | Hỏi-đáp mua sắm bằng AI hội thoại (đa-LLM) | LLM · truy-hồi-tăng-cường (RAG) |
| Ghi nhận lượt-hiển-thị + xu-hướng click (nền công-bằng-hoá) | 🟢 DONE | 27% | Bắt "đã hiển thị gì cho ai" — bắt buộc để về sau đánh giá gợi ý công bằng, không thiên lệch | Service thu-thập-hành-vi |

### BT02 — Cố vấn quyết định cho shop (Optimization / Decision Copilot)

> **Khung đúng = "cố vấn tiền-mặt-và-kệ-hàng", KHÔNG phải "tối ưu biên-lợi-nhuận".** Một màn hình trả lời: *"Thứ Năm này lấy gì off truck, hết bao nhiêu tiền, món gì đang ế trên kệ?"*

*Ai được lợi: chủ shop nhỏ ra quyết định nhập/giá tốt như chuỗi lớn — nhờ trí tuệ gộp từ toàn mạng.*

| Hạng mục | Trạng thái | % | Mô tả nghiệp vụ (business) | Công nghệ |
|---|---|---|---|---|
| ⭐ **Bắt giá vốn từ hoá đơn nhập (bằng AI-vision)** | 🟢 DONE | 20% | **Bài toán hạng-nhất** — mọi công thức lãi/lỗ vô nghĩa nếu không biết giá vốn thật. ICP đọc hoá đơn nhập bằng AI → lịch sử giá vốn → nền cho margin & cho vay | Service nhập-hàng · AI-vision đọc hoá đơn · lịch sử giá vốn theo thời gian |
| Cố vấn nhập hàng (số lượng tối ưu + tồn an toàn) | 🔵 PLANNED | 5% | "Lấy ~24 thùng, xấu nhất mất bao nhiêu, vì sao" — toán tồn kho ĐÚNG (tính cả mất-doanh-số khi hết hàng + biến-động thời-gian-giao) | Toán newsvendor · tồn-an-toàn có biến-động lead-time |
| Cố vấn giá (co-giãn cầu, nhìn đối thủ khu vực) | 🔵 PLANNED | 5% | Đề xuất giá đua với **shop kế bên** (không phải Shopee): giá-đề-xuất của nhà phân phối + chuẩn khu vực | Ước-lượng co-giãn giá · nhân-quả gộp |
| Cố vấn khuyến mãi (uplift thật, trừ ăn-thịt-lẫn-nhau) | 🔵 PLANNED | 3% | Đo tác động thật của khuyến mãi (trừ phần "dời mua tương lai"), kèm chặn pháp lý khuyến-mãi ≤50% | ML nhân-quả (uplift) · chặn theo luật |
| ⭐⭐ **Đường ray nhập-hàng + bộ não gộp** | 🔵 PLANNED | 3% | **10x MOAT**: đơn nhập chảy qua ICP → (1) bắt cost tự động (shop khỏi nhập liệu), (2) thấy cầu across hàng nghìn shop → tặng shop nhỏ dự-báo/co-giãn nó không tự tính được, (3) biến gợi ý thành **HÀNH ĐỘNG** (bấm Xác-nhận → đặt đơn + rút vốn) | Đường-ray đặt-lại-hàng · nhân-quả gộp cross-shop |
| Sổ niềm tin (dự-đoán vs thực-tế vs shop-làm-theo) | 🔵 PLANNED | 2% | "Tháng trước tôi khuyên lấy 24, bạn bán 22 — rất sát" → xây niềm tin bằng thành-tích thật | Nguyên-thuỷ theo-dõi-niềm-tin |
| Chuẩn hoá "đối tượng quyết định" (hành động · lợi-ích-kỳ-vọng · độ-tin-cậy · ràng-buộc) | 🔵 PLANNED | 3% | Đầu ra nhất quán: khuyến-nghị-trước rồi mới tự-động, tính lãi-ròng không phải doanh-thu-gộp | Service quyết-định |

### BT03 — Dự báo nhu cầu theo xác suất (Demand Forecast)

*Ai được lợi: shop trữ đúng lượng (không ế / không hụt), nhà phân phối biết đẩy hàng gì, nhà cho vay biết cấp vốn đúng lúc.*

| Hạng mục | Trạng thái | % | Mô tả nghiệp vụ (business) | Công nghệ |
|---|---|---|---|---|
| ⭐ **Bắt cầu bị mất/từ chối (hết hàng + xem/thêm-giỏ không mua)** | 🟢 DONE | 27% | Không có tín hiệu này thì "hiệu chỉnh cầu ẩn" vô nghĩa — ICP bắt NGAY (hành-vi + sự-kiện-hết-hàng) | Service thu-thập-hành-vi · sự-kiện-hết-hàng · sự-kiện-cầu |
| Hạ tầng biến-số (giá theo thời gian · lịch chiến dịch) | 🟢 DONE | 22% | Đường ống biến-số nuôi model (giá đổi theo ngày, lịch Tết/mega-sale/ngày-lương) | Lịch sử giá · nhập-hàng · lịch chiến-dịch |
| Dự báo xác suất (dải P10/P50/P90, nhiều kịch-bản) | 🔵 PLANNED | 2% | Không đoán 1 con số — đoán **dải rủi ro**; 1 kết quả dùng chung cho cả nhập-hàng + cho-vay + đối-soát | Chronos-2 (không cần train) · AutoGluon · độ-đo pinball/CRPS |
| Chọn model theo độ giàu dữ liệu | 🔵 PLANNED | 2% | Shop mới ít data → foundation-model không-cần-train; đủ data → model cổ điển; đủ khối lượng → model gộp | Chronos-2/TimesFM · statsforecast · bộ định-tuyến |
| Học chung giữa các shop (partial pooling) | 🔵 PLANNED | 2% | "Mượn" dữ liệu SKU→ngành→toàn-mạng → giải bài toán shop mới thiếu lịch sử + bảo vệ riêng-tư | Bayes phân-tầng (PyMC) |
| ⭐⭐ **Dự cảm nhu cầu TOÀN MẠNG (báo sốt cầu sớm)** | 🔵 PLANNED | 2% | **10x MOAT**: gộp tín hiệu đầu-phễu + vĩ-mô-VN (đếm-ngược-Tết / ngày-lương / thời-tiết) → báo sốt cầu **1–2 tuần TRƯỚC** khi hiện ở 1 shop → nối nhà-cho-vay **duyệt trước gói vốn nhập-hàng đúng lúc sốt** | Dự-cảm toàn-mạng · nối tín-hiệu cho-vay |

**Vận hành ML nghiêm túc ("60% còn lại" khi ship model thật):** kho đặc-trưng đúng-thời-điểm · sổ model + truy-vết nguồn-gốc · phát-hiện-lệch-dữ-liệu · chống lệch train↔serve · **chạy-bóng → thử-nghiệm-hẹp → tự-quay-lui** · giải-thích được (bắt buộc pháp lý cho cho-vay) · hiệu-chỉnh xác suất. → 🔵 PLANNED (~2%). *Đây là lý do ICP làm "quản-trị-từ-đầu", không lắp-thêm-sau.*

---

## 4. Người mua — Trải nghiệm mua sắm (ngang Amazon/Shopee)

| Tính năng | Trạng thái | % | Mô tả nghiệp vụ (business) | Công nghệ |
|---|---|---|---|---|
| Gian hàng + trang chi tiết sản phẩm (chọn biến thể) | 🟢 DONE | 30% | Trang sản phẩm đầy đủ: ảnh, mô tả, chọn màu/size, giá, tồn | Ghép danh-mục + niêm-yết · Next.js |
| Giỏ hàng (thêm / sửa số-lượng / xoá / biến-thể / khuyến-mãi / thuế) | 🟢 DONE | 32% | Giỏ đầy đủ: từng dòng theo biến thể, tính thuế, tự honor giá thấp hơn khi giá đổi | NestJS · Postgres · cách ly cấp-dòng |
| Lưu để mua sau (save-for-later) | 🟢 DONE | 30% | Danh sách riêng, chuyển qua-lại với giỏ | Postgres · cách ly cấp-dòng |
| Thanh toán (sổ địa chỉ + phí ship + COD/chuyển-khoản/ví) | 🟢 DONE | 30% | Luồng đặt hàng hoàn chỉnh: chọn địa chỉ, phí ship, phương thức thanh toán | NestJS · khoá chống tranh-chấp tồn |
| Đặt hàng → tạo đơn + giữ/trừ kho | 🟢 DONE | 28% | Đơn tạo ra trừ kho đúng, chống bán-quá; nhiều bước phối hợp an toàn | Saga giao-dịch · sự-kiện |
| Theo dõi đơn thời-gian-thực | 🟢 DONE | 28% | Trạng thái đơn cập nhật trực tiếp không cần refresh | Luồng thời-gian-thực · Redis |
| Trang chủ người mua + các "kệ" khám phá (xu-hướng / bán-chạy / mới-về / vừa-xem) | 🟡 PARTIAL | 17% | Trang chủ với các kệ gợi ý theo độ phổ biến — tăng khám phá & chuyển đổi; công thức phổ-biến có trọng-số mua + suy-giảm theo thời gian | Bảng tổng-hợp phổ-biến · điểm xu-hướng Vespa |
| Duyệt theo danh mục (lọc / sắp-xếp / phân-trang) | 🟡 PARTIAL | 15% | Duyệt hàng theo ngành hàng với bộ lọc, sắp xếp, phân trang | Danh mục · duyệt + gom-nhóm Vespa |
| Sản phẩm vừa xem (recently-viewed) | 🟡 PARTIAL | 10% | Xem lại nhanh sản phẩm vừa lướt (khách vãng-lai lưu máy, khách đăng-nhập lưu server) | Đọc-lại hành-vi · cách ly cấp-dòng |
| **Đánh giá & xếp hạng (reviews/ratings)** | 🟡 PARTIAL | 17% | **Cốt lõi niềm tin mua hàng** + nội-dung-người-dùng nuôi AI: có đánh giá/sao với **gate "đã-mua-mới-được-đánh-giá"** (chống review giả), tự tính điểm trung bình | Công-cụ-ghi đánh-giá · gate đã-mua · tự-tính điểm |
| Hỏi-đáp sản phẩm (Q&A) | 🟡 PARTIAL | 13% | Hỏi-đáp ngay trên trang SP (người bán hoặc người-đã-mua trả lời) | Công-cụ-ghi hỏi-đáp · kiểm-soát quyền-ghi |
| Trả hàng / hoàn tiền (yêu cầu từ người mua) | 🟡 PARTIAL | 13% | Luồng trả hàng cơ bản; cần hoàn thiện đầy đủ | Trả-hàng · cách ly cấp-dòng |
| Gợi-ý-gõ khi tìm kiếm (autocomplete) | 🟠 NEEDED | 0% | Baseline Amazon/Shopee — gõ tới đâu gợi ý tới đó | — |
| Mua lại nhanh (reorder từ đơn cũ) | 🟠 NEEDED | 0% | Tạp hoá mua lặp hằng tuần — tăng giữ-chân | — |
| Danh sách yêu thích + so sánh sản phẩm | 🟠 NEEDED | 0% | Khám phá + cân nhắc trước khi mua | — |
| Thông báo đơn (đẩy / SMS / Zalo) | 🟠 NEEDED | 0% | Người Việt kỳ vọng tin nhắn cập nhật đơn qua Zalo/SMS | Service thông-báo · Zalo |
| Hồ sơ & trang tài khoản người mua (đơn · địa chỉ · sổ) | 🟡 PARTIAL | 10% | Trung tâm quản lý đơn hàng, địa chỉ, thông tin cá nhân | Next.js · service khách-hàng |

---

## 5. Người bán — Quản trị gian hàng (Seller-Central)

| Tính năng | Trạng thái | % | Mô tả nghiệp vụ (business) | Công nghệ |
|---|---|---|---|---|
| Quản lý sản phẩm (thêm/sửa/ẩn + biến thể cha-con + gallery ảnh) | 🟢 DONE | 30% | Đăng/sửa/ẩn sản phẩm, biến thể kiểu Amazon, thư viện ảnh trên CDN | Service danh-mục · kho-ảnh/CDN |
| Danh mục / phân loại hàng hoá | 🟢 DONE | 30% | Sắp xếp sản phẩm theo ngành hàng | Service danh-mục |
| Nhập hàng loạt bằng file CSV | 🟢 DONE | 28% | Upload 1 file → đăng hàng trăm sản phẩm (kèm biến thể) | Nhập-loạt · đồng-bộ chỉ-mục tìm-kiếm |
| Vòng đời sản phẩm (bán / tạm-ẩn / lưu-trữ / nháp) | 🟢 DONE | 30% | Ẩn/hiện đảo được, vẫn giữ liên-kết với đơn cũ | Xoá-mềm |
| Tồn kho 1 điểm (nguyên-tử, chống bán-quá) | 🟢 DONE | 30% | Trừ kho nguyên-tử, không bao giờ bán quá số hàng có | Service tồn-kho · khoá nguyên-tử |
| Sổ-cái-kho (mọi biến động: nhập/xuất/giữ/huỷ) | 🟢 DONE | 25% | Nhật ký mọi thay đổi tồn — nền đối-soát + nuôi dự-báo | Sổ-cái tồn-kho |
| Sự-kiện hết-hàng (bắt cầu bị mất) | 🟢 DONE | 25% | Ghi lại mỗi lần khách muốn mua mà hết hàng → tín hiệu vàng cho dự-báo | Sự-kiện-hết-hàng |
| Đa cửa hàng + nhân viên (phân vai) | 🟢 DONE | 27% | 1 chủ nhiều shop, phân vai nhân viên với quyền khác nhau | Service tổ-chức · phân-quyền |
| Bật/tắt gian hàng | 🟢 DONE | 28% | Đóng/mở shop an toàn (vd nghỉ Tết) | State-machine trạng-thái gian-hàng |
| Giữ hàng khi khách đang thanh toán (reserve/commit/release) | 🟡 PARTIAL | 10% | Giữ tồn tạm khi khách checkout, nhả ra nếu bỏ giỏ — chống bán-quá lúc cao-điểm | Service tồn-kho |
| Cảnh báo tồn thấp / điểm-đặt-lại-hàng | 🔵 PLANNED | 3% | Tự nhắc "sắp hết, nên nhập" theo ngưỡng | Tồn-kho + quyết-định |
| Khuyến mãi (mã giảm giá, free-ship, tặng-kèm) | 🟡 PARTIAL | 20% | Cơ bản; thiếu công cụ chiến-dịch/voucher đầy đủ | Khuyến-mãi · Postgres |
| Quản lý đơn (màn người bán) | 🟡 PARTIAL | 17% | Có đơn/giao dịch; cần màn quản-lý-đơn đầy đủ (xử-lý, đóng-gói, giao) | Đơn · cách ly cấp-dòng |
| Bảng phân tích kinh doanh (analytics) | 🟡 PARTIAL | 17% | Doanh-số/lượt-xem/chuyển-đổi theo sản phẩm & ngành hàng | Bảng tổng-hợp · biểu-đồ |
| **Cố vấn AI: nhập / giá / khuyến-mãi** | 🔵 PLANNED | 5% | Xem BT02 §3 — copilot cố vấn; **phần bắt giá-vốn đã có** | Service quyết-định |
| **Dự báo nhu cầu cho shop** | 🔵 PLANNED | 2% | Xem BT03 §3 — đường-ống-dữ-liệu đã có, model đang tới | Service dự-báo |
| Quyết toán & chi-trả (payout) cho người bán | 🟡 PARTIAL | 10% | Đối-soát doanh-thu; luồng chi-trả cho seller chưa đủ | Quyết-toán · sổ-cái |
| Phân tích giá đối thủ | 🟡 PARTIAL | 10% | Dữ-liệu mẫu; công-cụ thu-thập giá thật chưa code | Thu-thập-giá (planned) |
| Quản lý trả hàng (phía người bán) | 🟡 PARTIAL | 13% | Xử lý yêu cầu trả hàng cơ bản | Service trả-hàng |
| Kiểm duyệt đánh giá (gắn-cờ review) | 🟡 PARTIAL | 8% | Người bán gắn-cờ review nghi-vấn (mô hình chống-thao-túng); admin duyệt ẩn/hiện = lộ trình | Mô hình gắn-cờ chống-thao-túng |
| Quảng cáo / đẩy tin sản phẩm (sponsored) | 🟠 NEEDED | 0% | **Đường tiền quảng cáo** (như Shopee/Lazada) | Service quảng-cáo (planned) |
| Bán hàng livestream (live commerce) | 🟠 NEEDED | 0% | Livestream chiếm >60% tương-tác TMĐT VN — ưu tiên hạng nhất | Giao-diện live-commerce |
| Sức khoẻ tài khoản người bán (account health) | 🟠 NEEDED | 0% | Điểm vi-phạm/hiệu-suất, cảnh báo trước khi bị khoá | Chấm-điểm rủi-ro |

---

## 6. Nhà phân phối — Supplier / B2B (đường ray bắt dữ liệu)

> B2B không chỉ là "thêm một actor" — nó là **đường ray bắt giá vốn** (nuôi BT02) và **nguồn cầu nhiều tầng** (nuôi BT03). Đây là nơi ICP khác biệt hẳn so với marketplace thuần B2C.

| Tính năng | Trạng thái | % | Mô tả nghiệp vụ (business) | Công nghệ |
|---|---|---|---|---|
| **Bắt giá vốn từ hoá đơn nhập** | 🟢 DONE | 20% | Đọc hoá đơn nhà phân phối bằng AI → lịch sử giá vốn thật (đã tách service) | Service nhập-hàng · AI-vision · lịch-sử-giá theo thời gian |
| Vai trò nhà phân phối (phân quyền supplier) | 🟡 PARTIAL | 8% | Khung tài khoản + phân quyền cho nhà phân phối | Service tổ-chức · loại-tenant |
| Luồng đặt hàng sỉ (supplier → shop → người mua) | 🔵 PLANNED | 0% | Shop đặt sỉ từ nhà phân phối ngay trên nền tảng | Service nhập-hàng |
| **Công nợ B2B (mua-trả-sau)** | 🟠 NEEDED | 0% | Tập quán VN: phân-phối ↔ shop bán-chịu/công-nợ — chưa có | Quyết-toán · cho-vay |
| Bảng giá sỉ + số-lượng-tối-thiểu + giá theo bậc | 🟠 NEEDED | 0% | Giá sỉ giảm dần theo số lượng đặt | Niêm-yết sỉ |
| Tồn kho nhiều tầng (kho phân-phối → shop) | 🔵 PLANNED | 0% | Tối ưu tồn across nhiều tầng cung ứng | Tồn-kho · quyết-định |
| **Hoá đơn điện tử B2B** | 🟠 NEEDED | 0% | **Bắt buộc theo luật (NĐ 123/2024)** — chặn hợp-pháp-hoá giao dịch B2B | Tích-hợp hoá-đơn-điện-tử |
| Đặt hàng định kỳ (theo lịch xe tải) | 🟠 NEEDED | 0% | Tự đặt-lại-hàng theo chu kỳ giao của nhà phân phối | Đường-ray đặt-lại-hàng |
| Nhận hàng & đối chiếu (goods-receipt) | 🔵 PLANNED | 0% | Xác nhận hàng nhập khớp đơn + khớp hoá đơn | Service nhập-hàng |

---

## 7. Thanh toán & Tài chính (rails VN + cho vay)

| Tính năng | Trạng thái | % | Mô tả nghiệp vụ (business) | Công nghệ |
|---|---|---|---|---|
| VNPay / Momo / ZaloPay (thanh toán online) | 🟢 DONE | 28% | Cổng thanh toán VN, xác-thực-chữ-ký + chống tính-tiền-hai-lần | Bộ-nối cổng · ký & xác-thực HMAC |
| COD + chuyển khoản ngân hàng | 🟢 DONE | 30% | Phương thức offline phổ biến nhất VN | Postgres |
| Đối soát + tự-quét IPN thất-lạc | 🟢 DONE | 30% | Lưới an toàn khi tín-hiệu-thanh-toán rớt giữa đường | Đối-soát tự-động |
| Hoàn tiền (refund) | 🟡 PARTIAL | 20% | Momo đã xong môi-trường-thử; VNPay/ZaloPay chờ chạy-thật | Vòng hoàn-tiền |
| **VietQR / QR ngân hàng tức thì** | 🟠 NEEDED | 0% | **Rail phổ biến nhất VN hiện nay** — chưa có | VietQR / napas |
| **Hoá đơn điện tử** | 🟠 NEEDED | 0% | Bắt buộc theo luật (NĐ 123/2024) | Tích-hợp hoá-đơn-điện-tử |
| Thuế / VAT tự động | 🟠 NEEDED | 0% | Tính & khai VAT tự động | Bộ-tính-thuế |
| **Sổ cái quyết toán (kế toán kép per-shop)** | 🔵 PLANNED | 0% | Ghi thu/chi/phí/hoàn kiểu kế-toán-kép — nền cho payout + đối-soát + cho-vay; **không giữ tiền của khách** | Service quyết-toán · kế-toán-kép |
| Số dư nội sàn / hoàn vào ví-tiêu-nội-bộ | 🔵 PLANNED | 0% | Hoàn tiền vào số dư tiêu trong sàn (không cần giấy phép ví) | Sổ-cái quyết-toán |
| **Cho vay / mua-trả-sau (theo %-doanh-số)** | 🔵 PLANNED | 2% | **Đòn bẩy doanh thu lớn** + lấp khoảng-trống tín-dụng SME VN; mô hình kiểu Stripe/Square-Capital (chấm điểm trên dữ-liệu-giao-dịch, không cần điểm-tín-dụng-truyền-thống) | Chấm-điểm-rủi-ro · service cho-vay |
| Ví / ký-quỹ giữ-tiền-khách (open-loop) | 🔵 PLANNED | 0% | Cần giấy phép trung-gian-thanh-toán → đi đường hợp-tác đối-tác-có-phép | Đối-tác ngân-hàng-dịch-vụ |
| Đo mức sử dụng để tính phí SaaS | 🟡 PARTIAL | 13% | Ghi mức dùng theo shop; chưa nối hoá đơn | Bảng đo-mức-dùng |

---

## 8. Logistics & Vận chuyển (VN)

| Tính năng | Trạng thái | % | Mô tả nghiệp vụ (business) | Công nghệ |
|---|---|---|---|---|
| Sổ địa chỉ + tính phí ship cơ bản | 🟢 DONE | 27% | Nhập địa chỉ + tính phí theo phương thức | Service khách-hàng · bảng-cước |
| **Tích hợp hãng giao (GHN/GHTK/J&T/Ahamove)** | 🟠 NEEDED | 0% | **Bắt buộc để giao hàng ở VN** — tạo đơn giao + in vận đơn + tracking | API hãng-giao · service giao-vận |
| Đồng bộ trạng thái vận chuyển | 🟠 NEEDED | 0% | Cập nhật hành trình đơn từ hãng giao | Webhook hãng-giao |
| Đối soát tiền COD với hãng vận chuyển | 🟠 NEEDED | 0% | Khớp tiền COD hãng-giao thu hộ ↔ doanh-thu shop | Quyết-toán |
| Tính phí ship thời-gian-thực theo vùng | 🟠 NEEDED | 0% | Báo phí ship chính xác theo địa chỉ lúc đặt | API hãng-giao |
| Đa kho / chọn kho gần nhất | 🔵 PLANNED | 0% | Giao từ kho gần khách nhất — nhanh & rẻ hơn | Tồn-kho nhiều-tầng |

---

## 9. Marketing & Tăng trưởng

| Tính năng | Trạng thái | % | Mô tả nghiệp vụ (business) | Công nghệ |
|---|---|---|---|---|
| Mã giảm giá / free-ship / tặng-kèm | 🟢 DONE | 23% | Công cụ khuyến mãi cơ bản | Bộ khuyến-mãi |
| Voucher / công cụ chiến-dịch | 🟠 NEEDED | 0% | Chiến dịch có điều kiện áp (giảm theo giỏ, theo ngành hàng…) | Service chiến-dịch |
| Flash sale / deal theo khung giờ | 🟠 NEEDED | 0% | Sự kiện bán giá sốc theo giờ | — |
| Tích điểm / khách thân thiết (loyalty) | 🟠 NEEDED | 0% | Giữ chân khách mua lặp | — |
| Giới thiệu bạn (referral) | 🟠 NEEDED | 0% | Tăng trưởng lan-truyền | — |
| **Quảng cáo nội sàn (đấu-giá vị-trí)** | 🟠 NEEDED | 0% | **Đường tiền quảng cáo** (đấu-giá-giá-nhì), tách bạch rõ với kết-quả tự-nhiên | Service quảng-cáo |
| Marketing qua Email / SMS / Zalo | 🟠 NEEDED | 0% | Kênh nhắc-lại / chăm khách | Service thông-báo |
| Affiliate / KOC (bán qua người ảnh hưởng) | 🟠 NEEDED | 0% | Kênh #1 tăng-trưởng TMĐT VN | Affiliate |
| **Live commerce (bán livestream)** | 🟠 NEEDED | 0% | Livestream — kênh bán hạng nhất thị trường VN | Giao-diện live-commerce |

---

## 10. Tin cậy, Tuân thủ & Quản trị (Luật VN + GDPR)

> Tuân thủ xây sẵn từ ngày đầu = lợi thế: đối thủ thường lắp-thêm-sau ở năm-3 (đắt + rủi ro). ICP đã có.

| Tính năng | Trạng thái | % | Mô tả nghiệp vụ (business) | Công nghệ |
|---|---|---|---|---|
| Đồng ý dữ liệu theo mục đích (phân tích/AI/marketing) | 🟢 DONE | 30% | Người dùng đồng ý theo từng mục đích — cổng chặn mọi thu thập dữ liệu | Cổng-đồng-ý theo mục-đích |
| Quyền chủ thể dữ liệu (truy cập + xoá) | 🟢 DONE | 28% | Người dùng đòi xuất/xoá dữ liệu của mình — nghĩa vụ pháp lý (Luật BVDLCN 2025 / GDPR) | Xuất đa-bảng · xoá nguyên-tử |
| Chính sách lưu trữ / tự-xoá dữ liệu quá hạn | 🟢 DONE | 27% | Tự xoá dữ liệu hết thời-hạn theo từng loại | Bộ dọn-dẹp theo lịch |
| Nhật ký kiểm toán bất biến (chuỗi-băm + neo chống-sửa) | 🟢 DONE | 30% | Nhật ký không-thể-sửa — bằng chứng pháp lý/kiểm toán chống cả admin nội bộ sửa lén | Chuỗi-băm SHA256 · chữ-ký-số Ed25519 |
| Phát hiện vi phạm dữ liệu + quy trình báo-cáo-72h | 🟢 DONE | 27% | Quy trình báo cáo sự-cố-rò-rỉ đúng luật | Ghi-nhận sự-cố |
| Che dữ liệu cá nhân trước khi gửi AI nước ngoài | 🟢 DONE | 28% | Ẩn thông tin cá nhân trước khi gửi qua LLM cross-border (mặc-định-đóng) | Che-dữ-liệu + cổng-đồng-ý |
| Vòng đời shop (đóng-mềm + xoá dữ liệu) | 🟢 DONE | 28% | Đóng shop an toàn, chuyển-quyền, xoá dữ liệu đúng thứ tự | State-machine |
| Hồ sơ pháp lý (DPA / RoPA) | 🟡 PARTIAL | 17% | Tài liệu pháp lý; chốt trước khi go-live | Living doc |
| Chống gian lận giao dịch | 🟠 NEEDED | 0% | Chấm điểm rủi ro giao dịch (nền chung với cho-vay) | Chấm-điểm-rủi-ro |
| Kiểm duyệt nội dung (review/SP/UGC vi phạm) | 🟠 NEEDED | 0% | Lọc nội-dung-người-dùng vi phạm | Kiểm-duyệt |
| Chặn file upload độc hại (kiểm loại + kích thước) | 🟢 DONE | 28% | Kiểm "vân tay" file ảnh + giới hạn dung lượng TRƯỚC khi xử lý → chặn tấn công qua upload | Bộ kiểm vân-tay-file |

---

## 11. Kênh & Mobile

| Tính năng | Trạng thái | % | Mô tả nghiệp vụ (business) | Công nghệ |
|---|---|---|---|---|
| Web responsive (ưu tiên điện thoại) | 🟢 DONE | 28% | Giao diện web tối ưu cho màn hình điện thoại | Next.js · Tailwind |
| **App mobile iOS / Android** | 🟠 NEEDED | 0% | **80%+ giao dịch TMĐT VN là mobile** | React Native / Flutter |
| **Zalo (đăng nhập + chat OA + tin ZNS)** | 🟠 NEEDED | 0% | **50M+ người dùng VN** — kênh copilot cho chủ shop | Zalo OA / ZNS |
| PWA (cài như app, dùng offline nhẹ) | 🟠 NEEDED | 0% | Bước đệm rẻ trước khi làm app native | PWA |
| Thông báo SMS (mã OTP + cập nhật đơn) | 🟠 NEEDED | 0% | Xác thực + nhắn tin đơn hàng | Cổng SMS |
| Giao diện tiếng Việt hoàn toàn | 🟠 NEEDED | 0% | Một số trang còn tiếng Anh | Đa-ngôn-ngữ |
| Đồng bộ sàn ngoài (Facebook / TikTok Shop) | 🟠 NEEDED | 0% | Bán một chỗ, đồng bộ nhiều sàn | Đồng-bộ-kênh |

---

## 12. Vận hành nền tảng (Platform Ops)

| Tính năng | Trạng thái | % | Mô tả nghiệp vụ (business) | Công nghệ |
|---|---|---|---|---|
| Giám sát hệ thống (log / trace / metric) | 🟢 DONE | 28% | Theo dõi sức khoẻ hệ thống ở mức production | OpenTelemetry · Grafana |
| Worker nền lõi (đồng-bộ · thanh-toán · dọn-dẹp · kiểm-toán · lưu-trữ) | 🟢 DONE | 25% | Tác vụ chạy ngầm: đồng bộ dữ liệu, đối soát, dọn dẹp | NestJS workers |
| Hạ tầng triển khai (dựng toàn-bộ-service + cấp database mỗi service) | 🟡 PARTIAL | 15% | Dựng cả hệ để chạy/test đầu-cuối | Docker-compose · script cấp-phát |
| Worker phụ (tồn-kho · thông-báo · thu-thập-giá) | 🟡 PARTIAL | 7% | Khung có, đang hoàn thiện | Workers |
| Cảnh báo + ngân-sách-lỗi (SLO) + bảng-điều-khiển-as-code | 🟡 PARTIAL | 10% | Báo động khi vượt ngưỡng lỗi/độ-trễ | Bộ cảnh-báo (planned) |
| Bảng điều khiển admin (vi-phạm / lưu-trữ / kiểm-toán / shop / seed) | 🟡 PARTIAL | 18% | Trung tâm vận hành nội bộ | Next.js admin |
| Sao lưu & khôi phục (backup/DR) | 🟠 NEEDED | 3% | Sao lưu định kỳ + khôi phục sau sự cố | Sao-lưu Postgres · dựng-lại chỉ-mục tìm-kiếm |
| Đo tải & hiệu năng (load test) | 🟡 PARTIAL | 17% | Đo sức chịu tải REST/stream | k6 |

---

## 13. 🗺️ Lộ trình (sequencing)

> Nguyên tắc: **hoàn thiện lõi đã chạy → hoàn tất tách microservices → bật moat trí-tuệ (3 bài toán ML) → mở rộng B2B/cho-vay/scale.** Triết lý ML: **bò** (toán kinh tế minh bạch) → **đi** (ML cổ điển) → **chạy** (deep/foundation model).

**NGAY — "bán được thật ở VN" (0–3 tháng):** hoàn tất tách các service giao-dịch (giỏ/đơn/thanh-toán) · **VietQR** · **hoá đơn điện tử** · **tích hợp hãng giao (GHN/GHTK)** · **Zalo (đăng nhập + tin ZNS)** · **đánh giá/xếp hạng** đầy đủ · trả-hàng/hoàn-tiền đầy đủ · thông báo đơn · chạy-thật thanh toán.

**KẾ — "moat + kiếm tiền" (3–9 tháng):** **3 bài toán ML lên chạy thật** (dự-báo · cố-vấn nhập/giá · tìm-kiếm-gợi-ý 2-tầng + tiếng Việt) · **thí điểm cho vay** (theo %-doanh-số) · quảng cáo nội sàn · công-cụ voucher/chiến-dịch · chi-trả seller · vận-hành-ML cơ bản · dự-cảm-nhu-cầu-toàn-mạng.

**SAU — "nền tảng & mở rộng" (9–18 tháng):** B2B phân phối đầy đủ (đặt sỉ + **công nợ** + tồn nhiều tầng) · live-commerce · chịu-tải-cao/mở-rộng-ngang · quản-trị-ML đầy đủ (giải-thích/chuẩn-mực-kế-toán-tổn-thất) · xếp-hạng vốn-bảo-chứng · embedding riêng · mở rộng Đông Nam Á.

---

## 14. 📊 Bảng tổng hợp (dashboard)

| # | Mảng | Trạng thái | % |
|---|---|---|---|
| 1 | Hệ đa-bên (actors) & nền tảng kiến trúc microservices | 🟢 | ~28% |
| 2 | Trí tuệ — 3 bài toán ML lõi (BT01/02/03) | 🔵 | ~4% |
| 3 | Người mua (buyer) | 🟡 | ~24% |
| 4 | Người bán (merchant) | 🟡 | ~20% |
| 5 | Nhà phân phối (B2B / bắt dữ liệu) | 🟡 | ~8% |
| 6 | Thanh toán & Tài chính | 🟡 | ~22% |
| 7 | Logistics VN | 🟠 | ~4% |
| 8 | Marketing & Tăng trưởng | 🟠 | ~5% |
| 9 | Tin cậy & Tuân thủ | 🟢 | ~28% |
| 10 | Kênh & Mobile | 🟠 | ~5% |
| 11 | Vận hành nền tảng | 🟡 | ~20% |
| — | **TỔNG (toàn bộ tầm nhìn)** | 🟡 | **~14%** |
| — | *(Lõi thương mại + nền + kiến trúc — đã de-risk)* | 🟢 | *~26%* |

---

## 15. 📎 Ghi chú cho nhà đầu tư

- **Cách đọc:** cột **Trạng thái** cho biết đâu là **thật (🟢)**, đâu là **lộ trình (🔵/🟠)** — ICP không trộn lẫn hai thứ. Cột **%** là **ước lượng thận trọng** (chúng tôi báo cáo dè dặt để cam kết chắc chắn).
- **Giá trị cốt lõi nằm ở §1 (hệ đa-bên) + §2 (kiến trúc microservices — đã de-risk) + §3 (3 bài toán ML — moat).** Marketplace là *nền sinh dữ liệu*; trí tuệ + full-chain (mua → bán → nhập → vay) là *khác biệt không sao chép được*.
- **Vì sao % trí-tuệ thấp mà vẫn là moat:** phần khó của ML không phải viết model — mà là **(a) bắt được dữ liệu độc quyền không-thể-lấy-lại** (ICP đã dựng đường ống + tách sẵn các service tồn-kho/nhập-hàng/thu-thập-hành-vi) và **(b) nghiên cứu đúng bài toán** (đã làm sâu, chuẩn bigtech). Model là bước có-đường-đi.
- **Càng nhiều bên tham gia, moat càng sâu:** mỗi actor mới (§1) làm dữ liệu độc quyền dày thêm và mở một đường doanh thu — đây là hiệu-ứng-mạng-lưới mà marketplace 2-bên không có.
- **Số liệu thị trường (TAM/ARR/LTV):** cố ý KHÔNG đưa vào tài liệu này — sẽ trình bằng dữ liệu traction từ pilot thật ở pitch-deck, không phải ước đoán.

> *Living document — cập nhật khi trạng thái đổi. Lần cuối: 2026-07-06.*
