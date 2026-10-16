## 4.1 Bảng so sánh: Thủ công (Traditional) vs DevOps (CI/CD Pipeline)

| Tiêu chí | Cách THỦ CÔNG (Traditional) | Cách DEVOPS (CI/CD Pipeline) |
|----------|---------------------------|------------------------------|
| **Làm sao để deploy?** | Mở FTP/SSH → copy `index.html` lên server → sửa cấu hình web server (Apache/Nginx) → khởi động lại dịch vụ. Làm tay từng bước một. | Push code lên GitHub → pipeline tự chạy, không cần thao tác gì thêm. |
| **Mất bao lâu?** | 15–30 phút cho 1 lần deploy (tải file, upload, cấu hình, kiểm tra). Càng nhiều file càng lâu. | **~10 giây** (đo thực tế trong lab: run `38223054494` hoàn tất trong 9s). |
| **Ai làm deploy?** | Chính bạn (hoặc admin) — phải có mặt, phải nhớ đúng quy trình. | **GitHub Actions** (máy tự làm). Bạn chỉ cần `git push`. |
| **Có bị sai sót không?** | **Có** — dễ quên bước, copy nhầm file, upload thiếu file, sai đường dẫn, sửa nhầm server. Lỗi con người là không tránh khỏi. | **Không** — pipeline chạy đúng kịch bản YAML, nhất quán 100% ở mọi lần chạy. |
| **Rollback nếu lỗi?** | Phải **nhớ** bản cũ là bản nào, còn giữ file cũ hay không, rồi copy ngược lại bằng tay. Nếu mất file backup thì gần như bó tay. | `git revert <commit>` + push → pipeline tự deploy lại bản cũ. Lịch sử nằm trong Git nên **không bao giờ mất bản cũ**. |
| **Làm sao biết deploy thành công?** | Tự mở browser kiểm tra bằng mắt. Không có log, không ai báo lỗi nếu bạn không tự phát hiện. | **GitHub Actions** hiển thị ✅/❌ cho từng step + gửi notification qua email. Xem được log chi tiết. |

### Bảng mở rộng — các tiêu chí bổ sung

| Tiêu chí | Cách THỦ CÔNG (Traditional) | Cách DEVOPS (CI/CD Pipeline) |
|----------|---------------------------|------------------------------|
| **Có log / lịch sử deploy?** | Không có, hoặc chỉ ghi tay vào file text. Muốn biết ai deploy lúc nào phải hỏi nhau. | Có đầy đủ: mỗi lần chạy là 1 record trong tab **Actions**, kèm commit, người push, thời gian, log từng step. |
| **Nhiều người cùng deploy?** | Dễ **ghi đè lên nhau** — hai người upload cùng lúc thì người sau thắng, không ai biết. | An toàn — Git quản lý phiên bản, pipeline chạy tuần tự, không ghi đè mất code. |
| **Chi phí hạ tầng?** | Phải thuê VPS/server + tốn thời gian cài đặt, bảo trì. | **Miễn phí** — GitHub Actions (2.000 phút/tháng cho repo private, không giới hạn cho repo public) + GitHub Pages. |
| **Có tự kiểm tra code không?** | Không — deploy thẳng, lỗi thì người dùng phát hiện trước. | Có — step `Validate HTML` chặn deploy nếu thiếu file; có thể thêm test/Lighthouse CI. |
| **Khả năng lặp lại (repeatability)** | Mỗi người làm một kiểu, kết quả khác nhau. | Cùng 1 file YAML → kết quả **giống hệt nhau** ở mọi lần chạy. |

---

## 4.2 Sơ đồ DevOps Workflow đã trải nghiệm

### Sơ đồ tổng quát (6 giai đoạn vòng đời DevOps)

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │                     VÒNG ĐỜI DEVOPS — LAB 1                          │
  └──────────────────────────────────────────────────────────────────────┘

   ① PLAN            ② CODE            ③ BUILD           ④ TEST
  ┌─────────┐      ┌─────────┐       ┌─────────┐       ┌─────────┐
  │  Đọc đề │─────▶│VS Code  │──────▶│Git commit│─────▶│Validate │
  │  lab    │      │index.html│      │git push  │      │ HTML    │
  └─────────┘      └─────────┘       └─────────┘       └─────────┘
   GitHub            Viết code         Đóng gói          Kiểm tra
   Issues/           HTML+CSS          phiên bản         file tồn tại
   README                              (Git)             + kích thước

                                         │
                                         ▼
  ┌──────────────────────────────────────────────────────────────────────┐
  │  ⑤ DEPLOY  —  GitHub Actions Runner (ubuntu-latest) — MIỄN PHÍ       │
  │                                                                      │
  │   Trigger: on: push → branches: [main]                               │
  │      │                                                               │
  │      ├─▶ Step 1: actions/checkout@v4      → lấy code về runner       │
  │      ├─▶ Step 2: Validate HTML            → kiểm tra index.html      │
  │      └─▶ Step 3: peaceiris/actions-gh-pages@v3                       │
  │                    github_token: GITHUB_TOKEN                        │
  │                    publish_dir: ./  →  publish_branch: gh-pages      │
  │                              │                                       │
  │                              ▼  (git push origin gh-pages)           │
  │                    ┌───────────────────────┐                         │
  │                    │  Nhánh gh-pages       │                         │
  │                    │  (bản public của web) │                         │
  │                    └───────────┬───────────┘                         │
  └────────────────────────────────┼─────────────────────────────────────┘
                                   ▼
                     ┌──────────────────────────────┐
                     │   GitHub Pages build         │
                     │   (build_type: legacy)       │
                     └──────────────┬───────────────┘
                                    ▼
              🌐  https://hienvanla5.github.io/devops-lab1-portfolio/
                        (Ai trên thế giới cũng truy cập được)

  ┌──────────────────────────────────────────────────────────────────────┐
  │  ⑥ MONITOR — Tab Actions: ✅/❌ từng step, log chi tiết, email notify │
  │              → Phát hiện lỗi → sửa → push lại → vòng lặp tiếp tục    │
  └──────────────────────────────────────────────────────────────────────┘
```

### Sơ đồ so sánh 2 luồng

```
  ❌ THỦ CÔNG (15–30 phút, nhiều rủi ro)
  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
  │ Sửa code │──▶│ Mở FTP   │──▶│ Upload   │──▶│ Cấu hình │──▶│ Restart  │
  │ bằng tay │   │ bằng tay │   │ file tay │   │ server   │   │ dịch vụ  │
  └──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
       😰            😰             😰             😰             😰
   Dễ quên bước · Copy nhầm · Không log · Rollback khó · 1 người làm

  ✅ DEVOPS (10 giây, tự động, nhất quán)
  ┌──────────┐   ┌──────────────────────────────────────────────┐
  │git push  │──▶│        GitHub Actions tự động chạy           │
  │  main    │   │  checkout → validate → deploy → gh-pages     │
  └──────────┘   └───────────────────────┬──────────────────────┘
       😎                                ▼
                                   🌐 Website LIVE
  1 lệnh duy nhất · Nhất quán 100% · Có log · Rollback = git revert
```

> **Ảnh sơ đồ:** xem `screenshots/07-devops-workflow.png`
> **Ảnh bảng so sánh:** xem `screenshots/06-comparison-table.png`

---

## 4.3 Trả lời câu hỏi

### Câu 1: DevOps workflow tự động hóa những bước nào mà cách thủ công phải làm bằng tay?

**Trả lời:** DevOps tự động hóa **toàn bộ chuỗi từ sau khi code xong đến khi web lên sóng**, cụ thể là 4 nhóm việc:

| Bước | Thủ công phải làm | DevOps làm thay |
|------|-------------------|-----------------|
| **Build / đóng gói** | Nén file, chuẩn bị bộ file để đưa lên server | `actions/checkout@v4` lấy code, runner tự chuẩn bị môi trường |
| **Test / kiểm tra** | Tự mở browser xem có lỗi không | Step `Validate HTML` tự kiểm tra `index.html` tồn tại + in kích thước, **chặn deploy nếu thiếu file** (`exit 1`) |
| **Deploy** | Mở FTP/SSH, upload từng file, cấu hình server, restart dịch vụ | `peaceiris/actions-gh-pages@v3` tự commit và push sang nhánh `gh-pages` |
| **Publish / phát hành** | Chờ server cập nhật, tự kiểm tra URL | GitHub Pages tự build và public ra URL `*.github.io` |
| **Monitor / giám sát** | Không có gì, tự kiểm tra bằng mắt | Tab Actions hiện ✅/❌ từng step + gửi email notification |

**Điểm mấu chốt:** phần *quyết định* vẫn là con người (viết code, viết file YAML một lần), nhưng phần *thực thi lặp đi lặp lại* đã giao hết cho máy. Đúng tinh thần DevOps: **automate the repetitive, keep the judgment**.

---

### Câu 2: Nếu deploy sai (website bị lỗi), làm sao để quay lại phiên bản cũ?

**Trả lời:** Có 3 cách, xếp theo mức độ an toàn:

**Cách 1 — `git revert` (AN TOÀN NHẤT, khuyến nghị):**
```bash
git revert <commit_bi_loi>     # tạo commit MỚI đảo ngược thay đổi cũ
git push origin main           # pipeline tự chạy → deploy lại bản cũ
```
Ưu điểm: **không xóa lịch sử**, giữ nguyên dấu vết để sau này biết đã từng lỗi gì. Đây là cách đúng chuẩn cho nhánh đã chia sẻ với người khác.

**Cách 2 — `git reset --hard` + force push (dùng cẩn thận):**
```bash
git reset --hard <commit_tot>  # tua ngược nhánh về commit cũ
git push --force origin main   # BẮT BUỘC force vì đã viết lại lịch sử
```
⚠️ **Cảnh báo:** cách này **xóa vĩnh viễn** các commit sau đó khỏi nhánh và viết lại lịch sử. Nếu có người khác đang làm trên cùng nhánh, họ sẽ bị lỗi khi pull. Chỉ nên dùng khi làm một mình và vừa push nhầm trong vài giây trước.

**Cách 3 — Sửa trực tiếp trên GitHub UI:** Vào commit bị lỗi → **Revert** → GitHub tạo Pull Request revert → Merge → pipeline chạy. Phù hợp khi không có sẵn máy.

**Điểm hay của DevOps ở đây:** vì mọi phiên bản đều nằm trong Git và pipeline chạy hoàn toàn tự động, **rollback chỉ là 1 lệnh + 10 giây**, không cần backup thủ công và không sợ mất bản cũ. Đây chính là lợi ích lớn nhất so với cách truyền thống.

> **Liên hệ thực tế trong lab:** pipeline lần đầu của em **thất bại** với lỗi
> `remote: Permission to hienvanla5/devops-lab1-portfolio.git denied to github-actions[bot]. 403`.
> Em đã sửa bằng cách thêm `permissions: contents: write` vào `deploy.yml`, push lại
> (commit `7cc0344`) và pipeline chạy thành công trong 9 giây. Đây là minh chứng cho
> vòng lặp **Monitor → phát hiện lỗi → sửa → deploy lại** của DevOps.

---

### Câu 3: GitHub Actions pipeline đã giúp tiết kiệm bao nhiêu thời gian so với cách thủ công?

**Trả lời:**

| Việc | Thủ công | DevOps |
|------|----------|--------|
| Chuẩn bị file để deploy | 2–3 phút | 0 (tự động) |
| Upload lên server (FTP/SSH) | 5–10 phút | 0 (tự động) |
| Cấu hình web server + restart | 5–10 phút | 0 (tự động) |
| Kiểm tra kết quả | 3–5 phút | ~10 giây (tự động) |
| **TỔNG 1 LẦN DEPLOY** | **15–30 phút** | **~10 giây** |

**Con số cụ thể của em:** lệnh `git push` mất khoảng **2 giây**, sau đó pipeline chạy
**9 giây** (run `38223054494`) → tổng **~11 giây** là website đã cập nhật. So với
**~20 phút** nếu làm thủ công ⇒ **nhanh hơn khoảng 100 lần**.

**Tiết kiệm theo thời gian (giả sử 1 lần deploy/ngày, 22 ngày làm việc/tháng):**

| | Thủ công | DevOps | Tiết kiệm |
|---|---|---|---|
| 1 ngày | 20 phút | 0,2 phút | ~20 phút |
| 1 tháng | ~440 phút (7,3 giờ) | ~4,4 phút | **~7,3 giờ/tháng** |
| 1 năm | ~88 giờ (11 ngày làm việc) | ~53 phút | **~87 giờ/năm** |

**Nhưng thời gian không phải lợi ích lớn nhất.** Giá trị thật sự nằm ở chỗ:

1. **Không còn lỗi con người** — pipeline chạy đúng 100% mọi lần, không quên bước.
2. **Deploy được bất cứ lúc nào** — kể cả 2 giờ sáng, không cần ai thức.
3. **Dám deploy thường xuyên** — vì deploy rẻ và nhanh, nên có thể sửa nhỏ rồi deploy ngay, thay vì dồn lại thành "big bang release" đầy rủi ro.
4. **Ai cũng deploy được** — thành viên mới chỉ cần `git push`, không cần biết cấu hình server.
5. **Tự động hóa là tài sản** — file `deploy.yml` viết **1 lần**, dùng **mãi mãi**.