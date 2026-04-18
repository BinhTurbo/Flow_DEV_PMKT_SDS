
## Phan 10: Huong dan cho Developer - Quy trinh lam viec voi SonarQube Scan

### 10.1. Nguyên tắc cơ bản

**Tất cả các nhánh mới đều phải checkout từ nhánh `prod`:**

```bash
git checkout prod
git pull origin prod
git checkout -b [TEN_NHANH_MOI]
```

**Không được phép:**
- Pull code từ nhánh `dev` hoặc `dev-99` về nhánh của mình
- Pull code từ nhánh của người khác
- Chỉ được pull code từ nhánh `prod`

**Quy tắc đặt tên nhánh:**

| Loại | Format | Ví dụ |
|------|--------|-------|
| Tính năng mới | `feature/[ten_nguoi]/[ten_tinh_nang]` | `feature/hieund/tai-san-co-dinh` |
| Fix bug | `fixbug/[ten_nguoi]/[ma_bug]` | `fixbug/hieund/EBW-6655` |
| Hotfix (merge ngay, không qua test) | `hotfix/[ten_nguoi]/[mo_ta]` | `hotfix/hieund/xoa-token` |

**Commit message:**

```bash
git commit -m "[Fix bug] EBW-2793"
git commit -m "[Feature] Them chuc nang tai san co dinh"
```

---

### 10.2. Điều quan trọng nhất: Code trên nhánh của bạn PHẢI COMPILE ĐƯỢC khi clone từ đầu

**Đây là nguyên tắc sống còn khi có SonarQube scan.**

Khi bạn tạo Pull Request, service scan sẽ thực hiện CHÍNH XÁC các bước sau:

1. **Clone repo từ Bitbucket** (tương đương `git clone --branch [nhanh_cua_ban]`) - lấy toàn bộ code trên nhánh của bạn
2. **Build core JARs** (với dự án 200133): clone 4 repo core (`core-model`, `core-config`, `core-voucher`, `core-report`), build từng cái, copy vào `libs/`
3. **Chạy `mvn compile`** - compile toàn bộ source Java
4. **Chạy sonar-scanner** - phân tích code và đẩy kết quả lên SonarQube
5. **Kiểm tra Quality Gate** - pass hay fail
6. **Post kết quả** lên PR (comment) + gửi Discord (tag tên bạn)

**Vấn đề thường gặp:**

Nhiều dev khi làm việc ở local sẽ sửa một số file để chạy được trên máy mình (đổi đường dẫn, đổi config, comment bớt một số dòng...), nhưng khi push code lên Bitbucket lại **chỉ push những file mình cần sửa, còn những file sửa để chạy local thì không push**. Điều này dẫn đến:

- Trên local của bạn: compile OK (vì bạn đã sửa để chạy được)
- Trên server scan: **COMPILE FAIL** (vì server clone nguyên nhánh của bạn từ đầu, không có những thay đổi local của bạn)

**Cách làm đúng:**

Nhánh `prod` là nhánh gốc. Khi bạn checkout từ `prod` và clone về, project **PHẢI** compile thành công mà không cần sửa bất cứ gì. Các repo core (`core-model`, `core-config`, `core-voucher`, `core-report`) ở nhánh `prod` cũng phải compile thành công.

Nếu bạn checkout từ `prod` mà không compile được, đó là lỗi của nhánh `prod`, cần báo ngay để fix trước khi làm tiếp.

Khi bạn sửa code trên nhánh của mình, hãy **push TẤT CẢ các thay đổi** lên Bitbucket, bao gồm cả những file config, `pom.xml`, hay bất kỳ file nào bạn đã sửa. Không được giữ lại thay đổi ở local mà không push, vì server scan sẽ không có những thay đổi đó.

**Kiểm tra trước khi tạo PR:**

```bash
# Trên nhánh của mình, chạy compile xem có lỗi không
mvn compile -DskipTests

# Nếu OK mới tạo Pull Request
```

---

### 10.3. Core repos và cách service build core

Các dự án `200133ebwebbe` và `200133EbQueue` phụ thuộc vào 4 core repos:
- **core-model** (`200133EBCoreModel`)
- **core-config** (`200133EBCoreConfig`)
- **core-voucher** (`200133EBCoreVoucher`)
- **core-report** (`200133EBCoreReport`)

**Service scan resolve core branch theo logic sau:**

```
Cho mỗi core repo:
  1. Tìm nhánh cùng tên với nhánh PR của bạn trong core repo
     Ví dụ: PR của bạn là fixbug/hieund/EBW-6655
            -> Tìm nhánh fixbug/hieund/EBW-6655 trong core-model
  2. Nếu TÌM THẤY -> dùng nhánh đó để build core
  3. Nếu KHÔNG TÌM THẤY -> fallback về nhánh "prod" của core repo
```

**Điều này có nghĩa là:**

- **Trường hợp 1 - Bạn chỉ sửa code backend, không sửa core:**
  Service sẽ dùng nhánh `prod` của các core repo (mặc định).
  Bạn không cần làm gì thêm với core.

- **Trường hợp 2 - Bạn cần sửa cả code core lẫn backend:**
  Tạo nhánh **CÙNG TÊN** trong cả repo backend và repo core.
  Ví dụ: nhánh `fixbug/hieund/EBW-6655` phải tồn tại trong cả
  `200133ebwebbe` và `200133EBCoreModel` (hoặc bất kỳ core nào bạn sửa).
  Khi đó service sẽ tự động dùng nhánh của bạn trong core để build.

- **Trường hợp 3 - Nhánh core của bạn không compile được:**
  Service sẽ FAIL ở bước "Build core dependencies" (bước 3/6).
  Bạn sẽ nhận comment lỗi trên PR và thông báo Discord.
  Cách fix: vào repo core, checkout nhánh của bạn, chạy `mvn package -DskipTests` trên local,
  fix lỗi compile rồi push lên.

**Build core trên local (khi cần):**

```bash
cd libs/
mvn install:install-file -Dfile=core-model.jar -DgroupId=com.core -DartifactId=model -Dversion=1.0 -Dpackaging=jar
mvn install:install-file -Dfile=core-config.jar -DgroupId=com.core -DartifactId=config -Dversion=1.0 -Dpackaging=jar
mvn install:install-file -Dfile=core-voucher.jar -DgroupId=com.core -DartifactId=voucher -Dversion=1.0 -Dpackaging=jar
mvn install:install-file -Dfile=core-report.jar -DgroupId=com.core -DartifactId=report -Dversion=1.0 -Dpackaging=jar
```

Với dự án 200 nếu bị lỗi `jaspersoft.jfrog.io`:
```bash
mvn install:install-file -Dfile=libs\TimesNewRoman.jar -DgroupId=local.jasperFontOverrides -DartifactId=local.jasperFontOverrides -Dversion=1.0 -Dpackaging=jar
```

---

### 10.4. Các nhánh được scan và khi nào scan chạy

Service chỉ scan PR khi **nhánh đích** (nhánh merge vào) nằm trong danh sách `target_branches`:

| Project | Nhánh đích được scan |
|---------|----------------------|
| 200133ebwebbe | `dev-99`, `dev-99-mix-200`, `prod` |
| 200133ebwebfe | `dev-99`, `prod` |
| 200133EbQueue | `dev-99-4`, `prod` |
| 88EBWEB | `dev-2`, `prod` |
| 88ebqueue | `dev`, `prod` |

**Ví dụ:**
- PR từ `fixbug/hieund/EBW-6655` vào `dev-99` -> **ĐƯỢC SCAN**
- PR từ `fixbug/hieund/EBW-6655` vào `dev-88` -> **KHÔNG SCAN** (`dev-88` không trong danh sách)
- PR từ `fixbug/hieund/EBW-6655` vào `prod` -> **ĐƯỢC SCAN**

Scan tự động chạy khi:
- Bạn **tạo PR mới** vào nhánh đích
- Bạn **push commit mới** vào PR đã tồn tại
- Service **poll mỗi 5 phút**, phát hiện commit mới thì queue scan

---

### 10.5. Khi scan lỗi thì làm gì

Sau khi scan xong (thành công hay thất bại), bạn sẽ nhận thông báo qua 2 kênh:

1. **Comment trên PR** - kết quả Quality Gate, chi tiết lỗi (nếu có), 30 dòng cuối của log
2. **Discord** - tag tên bạn, kèm file log đầy đủ

**Nếu lỗi ở bước compile (5/6):**
- Đọc log trong comment PR hoặc file log Discord
- Tìm dòng `[ERROR]` để biết file nào lỗi
- Nguyên nhân thường gặp:
  - Thiếu dependency (chưa push file `pom.xml` đã sửa)
  - Code trên nhánh không compile được (chỉ chạy trên local vì có thay đổi chưa push)
  - Core repo không có nhánh tương ứng (xem mục 9.3)
- Fix: sửa code trên nhánh của bạn, push lên, service sẽ tự động scan lại

**Nếu lỗi ở bước core build (3/6):**
- Kiểm tra nhánh core có tồn tại và compile được không
- Nhánh `prod` của core repo PHẢI luôn compile thành công

**Nếu Quality Gate FAIL (bước 6/6):**
- Mở link SonarQube trong comment PR
- Xem các vấn đề: bug, code smell, vulnerability, duplicate code
- Fix code theo chỉ dẫn của SonarQube, push lên, scan lại

---

### 10.6. Xử lý conflict khi merge vào dev

**Trường hợp 1: Conflict ít, biết rõ code của mình**

1. Checkout vào nhánh dev:
   ```bash
   git checkout dev-99
   git pull origin dev-99
   ```
2. Merge nhánh của mình vào dev:
   ```bash
   git merge fixbug/hieund/EBW-6655
   ```
3. Sửa conflict trong IDE (giữ code của mình + code của người khác, KHÔNG xóa code người khác)
4. Commit và push:
   ```bash
   git add .
   git commit -m "[Fix conflict] EBW-6655 merge vao dev-99"
   git push origin dev-99
   ```
5. **CHẠY THỬ LẠI NHÁNH DEV SAU KHI FIX CONFLICT** - tránh làm hỏng dev

**Trường hợp 2: Conflict nhiều, do chưa pull `prod` về nhánh trước khi tạo PR**

1. Reset dev về trạng thái trước:
   ```bash
   git checkout dev-99
   git reset --hard origin/dev-99
   ```
2. Về nhánh của mình, merge `prod` về trước:
   ```bash
   git checkout fixbug/hieund/EBW-6655
   git pull origin prod
   # Sửa conflict ở đây (trên nhánh của mình, an toàn hơn)
   git add .
   git commit -m "[Merge prod] Pull prod ve nhanh fix"
   git push
   ```
3. Tạo lại Pull Request từ nhánh của mình vào dev
4. Nếu vẫn conflict, làm theo trường hợp 1

**Lưu ý quan trọng:** Luôn **bình tĩnh** khi gặp conflict. Conflict xảy ra do 2 người sửa cùng 1 file. Không hoảng, đọc kỹ code 2 bên rồi giữ cả 2 nếu cần.

---

### 10.7. Xử lý conflict khi merge vào prod

1. Merge `prod` về nhánh fix của mình:
   ```bash
   git checkout fixbug/hieund/EBW-6655
   git pull origin prod
   # Sửa conflict
   git add .
   git commit -m "[Merge prod] Resolve conflict truoc khi merge prod"
   git push
   ```
2. Tạo Pull Request từ nhánh fix vào `prod`
3. Đợi admin review và merge (nhánh `prod` KHÔNG auto-merge, cần admin duyệt)

---

### 10.8. Các bước tạo Pull Request trên Bitbucket — quy trình chuẩn cho dev

Đây là quy trình **đầy đủ** từ lúc bắt đầu code tới lúc scan xong. Làm đúng thứ tự này để tránh scan fail vì lý do thao tác.

**Bước 1 — Chuẩn bị local: checkout từ prod**

```bash
# Luôn luôn checkout từ prod, KHÔNG checkout từ dev hay nhánh người khác
git checkout prod
git pull origin prod

# Tạo nhánh mới đúng format
git checkout -b fixbug/[ten_ban]/[ma_bug]
# hoặc
git checkout -b feature/[ten_ban]/[ten_tinh_nang]
# hoặc (merge ngay không qua test)
git checkout -b hotfix/[ten_ban]/[mo_ta]
```

**Bước 2 — Sửa code và verify compile trên local**

```bash
# Sửa code, chạy thử tính năng trên local
# Với Java backend:
mvn compile -DskipTests

# Với frontend:
npm ci && npm run build

# Bắt buộc compile OK trên local trước khi push.
# Nếu compile fail trên local thì chắc chắn fail trên server scan.
```

**Bước 3 — Push TẤT CẢ thay đổi (không giữ lại bất kỳ file nào ở local)**

```bash
git add .
git commit -m "[Fix bug] EBW-xxxx"
git push -u origin fixbug/[ten_ban]/[ma_bug]

# Sau khi push, vào Bitbucket UI xem lại danh sách file trong nhánh
# để chắc chắn TẤT CẢ file đã thay đổi đều có mặt
```

**Bước 4 — Tạo Pull Request trên Bitbucket UI**

1. Mở `https://git-sds.softdreams.vn:7990`
2. Chọn repo tương ứng (vd `200133ebwebbe`)
3. Menu bên trái → **Pull requests** → **Create pull request**
4. Điền form:
   - **From**: nhánh của bạn (`fixbug/hieund/EBW-6655`)
   - **To**: nhánh đích. Chỉ các nhánh sau được scan tự động:

     | Project | Nhánh đích hợp lệ |
     |---------|-------------------|
     | 200133ebwebbe | `dev-99`, `dev-99-mix-200`, `prod` |
     | 200133ebwebfe | `dev-99`, `prod` |
     | 200133EbQueue | `dev-99-4`, `prod` |
     | 88EBWEB | `dev-2`, `prod` |
     | 88ebqueue | `dev`, `prod` |

   - **Title**: tóm tắt ngắn (vd `[Fix bug] EBW-6655 Fix loi tao voucher`)
   - **Description**: mô tả chi tiết lỗi/tính năng + screenshot nếu có
   - **Reviewers**: team lead hoặc người liên quan
5. Click **Create**

> PR vào nhánh KHÔNG thuộc `target_branches` sẽ **không được scan**. Service bỏ qua hoàn toàn.

**Bước 5 — Service tự động scan (không cần làm gì)**

- Service poll Bitbucket mỗi 5 phút → phát hiện PR mới → queue scan
- Hoặc bạn có thể trigger ngay nếu không muốn chờ:
  ```bash
  curl -X POST https://binhturbo.me/trigger \
    -H "Content-Type: application/json" \
    -d '{"repo_slug":"200133ebwebbe","pr_id":"2535"}'
  ```
- Muốn xem danh sách repo_flug thì truy cập vào đây để xem:
  ```bash
  https://binhturbo.me/health
  ```
- Nếu push thêm commit mới vào PR → service tự scan lại, chỉ commit mới nhất (không scan commit cũ)

**Bước 6 — Chờ kết quả và xử lý**

Thời gian scan trung bình:

| Project | Cache hit | Lần đầu / cache miss |
|---------|-----------|----------------------|
| 200133ebwebbe (Java + cores) | 7–9 phút | 25–30 phút |
| 200133EbQueue (Java + cores) | 7–9 phút | 25–30 phút |
| 88EBWEB (Java + Angular) | 8–12 phút | 15–20 phút |
| 200133ebwebfe (TS/Angular) | 3–5 phút | 8–10 phút |
| 88ebqueue (Java thuần) | 5–7 phút | 10–15 phút |

Kết quả đến qua 2 kênh cùng lúc:
- **Discord** (tag tên bạn theo `discord_user_map`) — chi tiết xem mục 10.10
- **Comment trên PR** — kết quả Quality Gate + 30 dòng log cuối

Xử lý:
- PASS + auto-merge không bị exclude → PR tự merge vào nhánh dev
- PASS + nhánh đích trong exclude list (mặc định `prod`) → chờ admin duyệt thủ công
- FAIL → đọc Discord log (mục 10.10), fix code trên local, push lại → service tự scan lại

**Checklist trước khi tạo PR:**

- [ ] Checkout từ `prod` (không phải `dev`, không phải nhánh người khác)
- [ ] Đặt tên nhánh đúng format (`feature/<ten>/<noi_dung>`, `fixbug/<ten>/<ma_bug>`, `hotfix/<ten>/<mo_ta>`)
- [ ] Commit message theo format `[Fix bug] EBW-xxxx` hoặc `[Feature] ...`
- [ ] `mvn compile -DskipTests` (hoặc `npm run build`) trên local OK
- [ ] Đã push **TẤT CẢ** file đã sửa (kể cả `pom.xml`, config, resources)
- [ ] Nếu sửa code core (200133) → tạo nhánh **cùng tên** trong repo core và push code core trước
- [ ] Nhánh đích có trong `target_branches` của project
- [ ] Bitbucket username của bạn đã có trong `discord_user_map` (nếu muốn nhận tag Discord)

---

### 10.9. Luồng scan 7 bước chi tiết — service làm gì sau khi nhận PR

Khi service nhận được 1 PR cần scan, nó thực hiện đúng 7 bước theo thứ tự. Bước nào không áp dụng cho project sẽ skip tự động.

**Bước 1/7 — Prepare source (chuẩn bị source code)**

- Service dùng git mirror cache (`/data/git-cache/<repo>.git`, bare mirror shared giữa các lần scan)
- Mirror chưa có → `git clone --mirror` (chậm lần đầu, ~1–5 phút tuỳ repo)
- Mirror đã có → `git fetch` (nhanh). Nếu có fetch trong 30 giây gần nhất → skip luôn (`GIT_FETCH_FRESHNESS_SECONDS`)
- Tạo worktree tách biệt tại `/data/workspace/<scan_key>` checkout đúng commit SHA của PR
- Log dòng nhận biết: `📦 Preparing source...` → `✅ Source ready at /data/workspace/...`
- FAIL phổ biến: `Khong tim thay ref '<branch>' trong git cache` → nhánh bị xóa hoặc commit không tồn tại

**Bước 2/7 — Resolve core branches** (chỉ áp dụng cho `200133ebwebbe`, `200133EbQueue`)

- Service lấy manifest `core_repos` từ `projects.json`: 4 core là `core-model`, `core-config`, `core-voucher`, `core-report`
- Với mỗi core repo, service thử tìm nhánh **cùng tên PR của bạn** trong core repo đó
  - Tìm thấy → dùng nhánh đó
  - Không tìm thấy → fallback về `prod` của core
- Kết quả là 1 `core_branch_map` quyết định bundle cache hash
- Log dòng: `🔍 Resolving core branches...` + bảng mapping (core-model → prod, core-config → fixbug/hieund/..., ...)
- Project không có core → **SKIP** bước 2–4

**Bước 3/7 — Build core dependencies** (chỉ project có core)

- Tính hash bundle từ commit SHA của 4 core repo
- Check cache tại `/data/core-bundles-cache/<hash>/`
  - Cache hit → skip build, reuse bundle (rất nhanh, ~1s)
  - Cache miss → clone 4 core repo, build theo thứ tự `core-model → core-config → core-voucher → core-report` bằng `mvn package -DskipTests`
- Bundle đóng gói các JAR + được cache lại cho lần sau
- Log: `🔧 Building core bundle...` hoặc `♻️ Core bundle cache hit`
- FAIL phổ biến:
  - `Build core failed at core-xxx` → dev đã push code core nhưng code core không compile được. Về repo core checkout nhánh đó, chạy `mvn package -DskipTests` trên local, fix, push lại.
  - Nếu service fallback về `prod` mà `prod` core cũng fail → báo admin ngay, **nhánh prod core phải luôn compile được**

**Bước 4/7 — Install core JARs** (chỉ project có core)

- Copy JAR từ bundle cache vào per-scan Maven repo: `/data/maven-cache/<scan_key>/.m2/`
- Chạy `mvn install:install-file` cho từng core JAR
- Rất nhanh (~5–10s)
- Log: `📚 Installing core JARs...`
- Ít khi fail ở đây — nếu fail thường do disk đầy hoặc permission

**Bước 5/7 — Maven compile** (chỉ `language = java`)

- Chạy `compile_command` từ `projects.json`, mặc định:
  ```
  mvn -Dmaven.repo.local=$MAVEN_REPO -T 1C --no-transfer-progress \
      -Dmaven.artifact.threads=10 compile -DskipTests \
      -Dmaven.javadoc.skip=true -Dmaven.source.skip=true
  ```
- Dùng Maven cache shared (`/data/maven-cache/shared/.m2/`) kết hợp per-scan local repo
- Log: `🔨 Compiling project...` + toàn bộ output Maven
- FAIL phổ biến:
  - `[ERROR] cannot find symbol / package does not exist` → dev quên push file import hoặc `pom.xml` mới
  - `[ERROR] Could not resolve dependencies` → thiếu dependency, thường do `pom.xml` chưa push
  - `Failed to execute goal ... COMPILATION ERROR` → đọc message tìm đúng file/dòng
- `language = ts` → **SKIP**

**Bước 6/7 — Frontend gate** (nếu cấu hình `frontend_*`)

- Áp dụng cho `200133ebwebfe` và `88EBWEB`
- Chạy theo thứ tự (bước nào có config thì chạy): `npm ci` → `npm run lint` → `npm run build` → `npm run jest`
- PR mode chạy fast (config mặc định, có thể chỉ lint)
- Branch mode chạy đầy đủ (xem `scan_mode_overrides.branch` trong `projects.json`)
- npm cache shared ở `/data/npm-cache` (persist qua các scan)
- Log: `🎨 Running frontend gate...` + output từng bước
- FAIL phổ biến:
  - `Module not found` → thiếu file import chưa push
  - `ESLint: error` → fix lint
  - `FAIL src/.../*.spec.ts` → test assertion sai, fix logic hoặc fix test
- Project không cấu hình frontend_* → **SKIP**

**Bước 7/7 — SonarQube scan + Quality Gate + notify + auto-merge**

- Chạy `sonar-scanner` với heap tuỳ ngôn ngữ (Java 4GB, TS 2GB)
- Scanner đẩy phân tích lên SonarQube server ở `http://10.100.120.190:9090`
- Service poll Compute Engine task (~vài giây / lần) đến khi SUCCESS hoặc timeout ~5 phút
- Fetch Quality Gate: `OK` / `ERROR`
- Post comment lên PR: status + link SonarQube + 30 dòng log cuối
- Gửi Discord embed (chi tiết mục 10.10)
- Nếu `QG=OK` và `to_branch` **không** thuộc `auto_merge_exclude_branches` → gọi Bitbucket API merge PR tự động
- Log: `📊 Running sonar-scanner...` → `⏳ Waiting for CE task...` → `✅ Quality Gate: OK|ERROR`

**Tóm tắt skip logic từng project:**

| Project | B2 Resolve core | B3 Build core | B4 Install core | B5 Compile Maven | B6 Frontend gate |
|---------|:---:|:---:|:---:|:---:|:---:|
| 200133ebwebbe | ✓ | ✓ | ✓ | ✓ | — |
| 200133ebwebfe | — | — | — | — | ✓ |
| 200133EbQueue | ✓ | ✓ | ✓ | ✓ | — |
| 88EBWEB | — | — | — | ✓ | ✓ |
| 88ebqueue | — | — | — | ✓ | — |

**Quan trọng:** Nếu bất kỳ bước nào fail, service **dừng ngay**, post lỗi lên PR + Discord, **không chạy tiếp các bước sau**. Workspace được giữ lại tại `/data/workspace/<scan_key>` để admin debug (vì `KEEP_WORKSPACE_ON_FAILURE=true`).

---

### 10.10. Đọc thông báo Discord — từng field nghĩa là gì và cách xử lý

Khi scan xong (thành công hay thất bại), bot gửi 1 embed message vào channel Discord. Hiểu được từng field sẽ giúp dev biết **scan fail ở đâu** và **fix như thế nào**.

**Ví dụ message PASS:**

```
@hieund SonarQube scan ✅ PASSED for 200133ebwebbe PR #2535

┌──────────────────────────────────────────────┐
│ SonarQube PR Scan                            │
│ ──────────────────────────────               │
│ SonarQube scan completed successfully.       │
│ [View in SonarQube](http://10.100.120.190:9090/dashboard?id=EA_...&pullRequest=2535)
│ [View Pull Request](https://git-sds.../pull-requests/2535)
│                                              │
│ Status: ✅ PASSED     Project: Ebweb Backend │
│ PR: #2535             Flow: fixbug/... → dev-99 │
│ Author: Hieu ND (hieund)  Quality Gate: OK   │
│ Commit: b628cccd      Repo: 200133ebwebbe    │
│ Duration: 8m 23s                             │
│                              2026-04-17 10:35 │
└──────────────────────────────────────────────┘
```

Viền message: **xanh lá** (color `3066993`).

**Ví dụ message FAIL (có file log đính kèm):**

```
@hieund SonarQube scan ❌ FAILED for 200133ebwebbe PR #2535

┌──────────────────────────────────────────────┐
│ SonarQube PR Scan                            │
│ ──────────────────────────────               │
│ Scan loi tai buoc [COMPILE] (line 582, exit 1).
│ [View Pull Request](https://git-sds.../pull-requests/2535)
│                                              │
│ Status: ❌ FAILED     Project: Ebweb Backend │
│ PR: #2535             Flow: fixbug/... → dev-99 │
│ Author: Hieu ND (hieund)  Quality Gate: N/A  │
│ Commit: b628cccd      Repo: 200133ebwebbe    │
│ Duration: 3m 12s                             │
│                                              │
│ 📎 200133ebwebbe-pr-2535.log (500 dòng cuối) │
│                              2026-04-17 10:35 │
└──────────────────────────────────────────────┘
```

Viền message: **đỏ** (color `15158332`). File log chỉ đính kèm khi FAIL (`DISCORD_ATTACH_LOG_ON_FAILURE=true`).

**Giải thích từng field:**

| Field | Ý nghĩa | Cách dev dùng |
|-------|---------|---------------|
| `@<user>` (mention) | Tag theo `discord_user_map` trong `projects.json`, dựa trên Bitbucket username của commit author cuối cùng | Dev nhận notification cá nhân. Không thấy tag → báo admin bổ sung mapping |
| **Status** | `✅ PASSED` hoặc `❌ FAILED` | PASS = OK, FAIL = phải đọc description + log |
| **Project** | Trường `name` trong `projects.json` | Biết đang nói về repo nào |
| **PR / Branch** | `#<pr_id>` (PR scan) hoặc tên branch (branch scan định kỳ) | Click **View Pull Request** để mở trực tiếp |
| **Flow** | `from_branch → to_branch` | Xác nhận đúng target branch, tránh scan nhầm PR |
| **Author** | `Display Name (bitbucket_username)` | Biết ai cần fix |
| **Quality Gate** | `OK` / `ERROR` / `N/A` | `OK` = qua gate; `ERROR` = fail gate (mở SonarQube xem chi tiết); `N/A` = scan fail **trước khi** chạy sonar (b1–b6), xem description |
| **Commit** | SHA ngắn 8 ký tự | So sánh với commit mới nhất trên PR. Nếu khác → service chắc đang scan lại commit mới hơn |
| **Repo** | Repo slug Bitbucket | Xác định repository chính xác |
| **Duration** | Tổng thời gian scan | Bất thường nếu quá lâu (scan 200133ebwebbe > 30 phút thì có thể treo) |
| **Description** (text trong embed) | Mô tả ngắn: scan PASS hoặc bước nào FAIL + lý do | Cực kỳ quan trọng khi FAIL — cho biết **bước nào** để biết đọc log ở đâu |
| **View in SonarQube** link | Link dashboard PR trên SonarQube (chỉ có khi đến được b7) | Xem chi tiết bugs / code smells / vulnerabilities / duplications / coverage |
| **View Pull Request** link | Link PR trên Bitbucket | Mở PR trực tiếp |
| 📎 File `.log` đính kèm | 500 dòng cuối của scan log (chỉ đính kèm khi FAIL) | Download đọc chi tiết lỗi |

**Description cho biết bước nào FAIL — bảng tra cứu:**

| Description trong embed | Bước fail | Cách fix |
|-------------------------|-----------|----------|
| `Khong tim thay ref '<branch>' trong git cache...` | B1 Prepare source | Branch/commit bị xóa. Push lại commit, hoặc trigger poll |
| `Scan loi tai buoc [CORE_BUILD] ...` | B3 Build core | Core repo không compile. Về repo core checkout nhánh đó, chạy `mvn package -DskipTests` local, fix, push |
| `Scan loi tai buoc [COMPILE] (line ..., exit 1)` | B5 Maven compile | Mở file log, tìm `[ERROR]` → đọc file/dòng lỗi → fix |
| `Maven compile thanh cong nhung khong tao duoc file .class` | B5 | Source rỗng hoặc `pom.xml` sai cấu trúc module. Báo admin |
| `frontend_workdir '<dir>' khong ton tai trong source repo` | B6 | Config `projects.json` sai. Báo admin |
| `Frontend enabled nhung khong tim thay package.json` | B6 | Thiếu `package.json` ở `frontend_workdir`. Báo admin |
| `Scan loi tai buoc [FRONTEND_LINT / FRONTEND_BUILD / FRONTEND_TEST]` | B6 | Mở log, tìm `ERROR in` hoặc `FAIL src/...`, fix code |
| `Khong co source directory nao ton tai trong repo` | B7 Sonar | `sonar_sources` config trỏ sai đường dẫn. Báo admin |
| `sonar-scanner chay xong nhung khong tao duoc report-task.txt` | B7 | Scanner crash (OOM / network). Báo admin retry |
| `SonarQube Compute Engine task ended with status: FAILED` | B7 server-side | SonarQube server lỗi. Báo admin |
| `Khong lay duoc Quality Gate tu SonarQube...` | B7 API | Tạm thời, trigger lại scan |
| `SonarQube Quality Gate failed` | B7 business logic | QG thực sự fail. Mở **View in SonarQube** → xem bugs/smells → fix code |

**Cách đọc file `.log` đính kèm — các mẫu lỗi phổ biến:**

File log chứa 500 dòng cuối của scan. Mở bằng editor bất kỳ, tìm một trong các pattern sau:

**Pattern 1 — Java compile error:**
```
[ERROR] /workspace/.../src/main/java/com/example/Foo.java:[42,15] cannot find symbol
[ERROR]   symbol:   class Bar
[ERROR]   location: class Foo
```
→ `Foo.java` dòng 42 thiếu import `Bar`. Kiểm tra bạn có quên push class `Bar` hoặc import statement.

**Pattern 2 — Missing Maven dependency:**
```
[ERROR] Failed to execute goal ... Could not resolve dependencies for project ...
[ERROR] The following artifacts could not be resolved: com.core:model:jar:1.0
```
→ Core JAR chưa được install. Project 200133 → b3/b4 có fail không? Project khác → `pom.xml` chưa push đủ.

**Pattern 3 — Frontend import thiếu file:**
```
ERROR in ./src/app/foo.component.ts
Module not found: Error: Can't resolve './bar'
```
→ `foo.component.ts` import `./bar` nhưng `bar.ts` chưa push.

**Pattern 4 — Frontend test fail:**
```
FAIL  src/app/foo.spec.ts
  ● FooComponent › should render
    Expected: "Hello"
    Received: "World"
```
→ Test assertion sai. Fix logic hoặc fix test.

**Pattern 5 — Sonar scanner OOM:**
```
ERROR: Java heap space
ERROR: Error during SonarScanner execution
```
→ Không phải lỗi dev, báo admin tăng heap hoặc giảm `sonar.sources`.

**Pattern 6 — Core build fail:**
```
[ERROR] Failed to execute goal on project core-model: ...
[ERROR] COMPILATION ERROR : /workspace/core-model/src/main/java/...
```
→ Code core không compile. Nếu bạn sửa core → fix trên nhánh core. Nếu bạn KHÔNG sửa core (service fallback về `prod`) → báo admin, `prod` core phải luôn compile được.

**Thứ tự ưu tiên khi nhận FAIL message (quy trình fix):**

1. Đọc **Description** trong embed → xác định bước nào fail (B1–B7)
2. Tra bảng ở trên → biết cách fix chung
3. Mở file log đính kèm → tìm `[ERROR]` / `FAIL` / `ERROR:` / `COMPILATION ERROR`
4. Xác định file + dòng + lý do → fix code trên local
5. Chạy `mvn compile -DskipTests` (hoặc `npm run build` / `npm run lint`) trên local → confirm OK
6. `git add . && git commit && git push` → service tự động scan lại commit mới nhất
7. Chờ Discord message tiếp theo. Lặp lại tới khi PASS.

**Khi nào KHÔNG phải lỗi của dev (báo admin):**

- Description chứa `day la loi API/tam thoi` → lỗi mạng/API tạm thời
- `SonarQube Compute Engine task ended with status: FAILED` → server SonarQube lỗi
- Duration > 30 phút + FAIL → scan có thể treo, cần admin kill
- `Khong co source directory nao ton tai` / `frontend_workdir ... khong ton tai` → config `projects.json` sai
- `prod` core repo không compile được → hạ tầng fail, không phải do PR của bạn

---

### 10.11. Tóm tắt quy trình làm việc hằng ngày

```text
1. Checkout từ prod
   git checkout prod && git pull origin prod
   git checkout -b fixbug/[ten_ban]/[ma_bug]

2. Sửa code, test local, đảm bảo compile được
   mvn compile -DskipTests

3. Push TẤT CẢ code đã sửa (không giữ lại thay đổi local)
   git add .
   git commit -m "[Fix bug] EBW-xxxx"
   git push -u origin fixbug/[ten_ban]/[ma_bug]

4. Tạo Pull Request trên Bitbucket vào nhánh dev (dev-99, dev-2...)

5. Đợi scan tự động (5 phút) hoặc xem kết quả trên PR comment + Discord
   - PASS: OK, đợi merge hoặc auto-merge
   - FAIL: đọc lỗi, fix code, push lại, scan tự động chạy lại

6. Khi đã test xong, tạo Pull Request vào prod
   - Đợi admin review và merge thủ công
```

**Không nên:**
- Sửa quá nhiều bug vào cùng 1 nhánh (nhánh để lâu không merge sẽ nhiều conflict)
- Tạo nhánh từ `dev` thay vì từ `prod`
- Pull code từ nhánh người khác
- Push code không compile được (scan sẽ fail và thông báo cho cả team trên Discord)
