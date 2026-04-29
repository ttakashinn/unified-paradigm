# Bản đồ tư duy thống nhất

> React Hooks, Elixir/OTP, Inversion of Control và Landscape Rộng hơn — tài liệu kiến trúc dành cho senior developer.

Tài liệu được build bằng [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) và deploy lên Cloudflare Pages.

---

## Cấu trúc dự án

```
unified-paradigm-docs/
├── mkdocs.yml                    # Cấu hình MkDocs Material
├── requirements.txt              # Python dependencies
├── README.md                     # File này
├── .gitignore
├── .github/
│   └── workflows/
│       └── deploy.yml            # GitHub Actions deploy lên Cloudflare Pages
└── docs/
    ├── index.md                  # Phần 0 — Nền tảng lý thuyết (landing page)
    ├── react.md                  # Phần 1 — React Hiện đại
    ├── elixir-otp.md             # Phần 2 — Elixir, Actor Model, GenServer
    ├── ioc.md                    # Phần 3 — Inversion of Control
    ├── liveview.md               # Phần 4 — Phoenix LiveView
    ├── unified-map.md            # Phần 5 — Bản đồ tư duy thống nhất
    ├── tradeoffs.md              # Phần 6 — Trade-offs & Anti-patterns
    ├── landscape.md              # Phần 7 — Landscape Toàn cục
    ├── bibliography.md           # Phần 8 — Phụ lục & Bibliography
    └── stylesheets/
        └── extra.css             # CSS tùy chỉnh
```

---

## Chạy local để preview

### Yêu cầu

- Python 3.10+
- pip

### Cài đặt và chạy

```bash
# Tạo virtual environment (khuyến nghị)
python -m venv venv
source venv/bin/activate     # Linux/macOS
# hoặc: venv\Scripts\activate  # Windows

# Cài dependencies
pip install -r requirements.txt

# Chạy dev server (http://localhost:8000)
mkdocs serve

# Hoặc build static site
mkdocs build
# Output sẽ ở thư mục site/
```

`mkdocs serve` có hot reload — sửa file markdown thì browser tự refresh.

---

## Deploy lên Cloudflare Pages

Có 2 cách: **GitHub Actions** (khuyến nghị) hoặc **Cloudflare Pages tự build**.

### Cách 1: GitHub Actions (khuyến nghị — full control)

#### Bước 1: Tạo Cloudflare API Token

1. Đăng nhập [Cloudflare Dashboard](https://dash.cloudflare.com/)
2. Vào **My Profile** → **API Tokens** → **Create Token**
3. Chọn template **"Edit Cloudflare Workers"** hoặc tạo Custom Token với permissions:
   - `Account` → `Cloudflare Pages` → `Edit`
   - `User` → `User Details` → `Read`
4. Copy token (chỉ hiện 1 lần)

#### Bước 2: Lấy Account ID

- Vào Cloudflare Dashboard → bất kỳ domain nào
- Account ID hiện ở sidebar bên phải

#### Bước 3: Tạo Cloudflare Pages project (rỗng)

1. Cloudflare Dashboard → **Workers & Pages** → **Create application** → **Pages** tab
2. Chọn **Direct Upload** (không connect Git, vì Actions sẽ push)
3. Đặt tên project: `unified-paradigm` (phải khớp với `projectName` trong `deploy.yml`)
4. Tạo project, có thể bỏ qua bước upload đầu tiên

#### Bước 4: Thêm Secrets vào GitHub repo

Vào GitHub repo → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**:

- `CLOUDFLARE_API_TOKEN` = token ở bước 1
- `CLOUDFLARE_ACCOUNT_ID` = account ID ở bước 2

#### Bước 5: Push lên GitHub

```bash
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin git@github.com:USERNAME/unified-paradigm.git
git push -u origin main
```

GitHub Actions sẽ tự build và deploy. Theo dõi ở tab **Actions** của repo.

URL sau khi deploy: `https://unified-paradigm.pages.dev` (hoặc custom domain nếu config thêm).

---

### Cách 2: Cloudflare Pages tự build (đơn giản hơn nhưng ít control)

1. Push code lên GitHub repo trước
2. Cloudflare Dashboard → **Workers & Pages** → **Create application** → **Pages** → **Connect to Git**
3. Chọn repo `unified-paradigm`
4. Build settings:
   - **Framework preset**: None
   - **Build command**: `pip install -r requirements.txt && mkdocs build`
   - **Build output directory**: `site`
   - **Root directory**: `/` (mặc định)
   - **Environment variables**:
     - `PYTHON_VERSION` = `3.12`
5. Save and Deploy

Mỗi lần push lên `main` thì Cloudflare tự rebuild. Pull request tự tạo preview deployment.

**Lưu ý:** Cloudflare Pages build environment có Python sẵn nhưng version có thể cũ — set `PYTHON_VERSION=3.12` trong env vars để chắc chắn.

---

## Cấu hình custom domain (tùy chọn)

1. Cloudflare Pages project → **Custom domains** → **Set up a custom domain**
2. Nhập domain (vd: `paradigm.example.com`)
3. Cloudflare tự thêm CNAME record nếu domain đã quản lý ở Cloudflare
4. SSL tự động (Cloudflare Universal SSL)

---

## Cập nhật nội dung

Mỗi page là một file markdown trong `docs/`. Sửa file → commit → push → deploy tự động.

### Quy ước

- File đầu là `# Heading` (H1) — sẽ thành page title.
- `## Heading` (H2) hiện trong sidebar TOC bên phải.
- Footer điều hướng (Trước/Tiếp theo) ở cuối mỗi file.

### Thêm page mới

1. Tạo file mới trong `docs/`, ví dụ `new-page.md`
2. Thêm vào `nav:` trong `mkdocs.yml`:
   ```yaml
   nav:
     - "Phần mới": new-page.md
   ```
3. Commit và push.

---

## Troubleshooting

### Build fail với "strict mode" warning

`mkdocs build --strict` (trong workflow) sẽ fail nếu có warning như:
- Link gãy
- Page không có trong `nav`
- Heading anchor không tồn tại

Chạy local `mkdocs build --strict` trước khi push để catch sớm.

### Bị thiếu dependency khi deploy

Check `requirements.txt` đã đủ chưa. Nếu thêm extension mới (vd `mkdocs-glightbox`), thêm vào `requirements.txt` và `markdown_extensions:` trong `mkdocs.yml`.

### Cloudflare Pages cache không clear

Vào project → **Deployments** → chọn deployment cũ → **Retry deployment** hoặc trigger lại bằng push commit mới (commit rỗng được: `git commit --allow-empty -m "trigger redeploy"`).

---

## License

Tài liệu thuộc bản quyền của tác giả. Code/template của site này có thể dùng tự do.
