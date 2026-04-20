# MCS đồ án LaTeX

Kho chứa bản báo cáo LaTeX. Bản đang dùng nằm trong thư mục **`latex_new/`**; **`latex_old/`** là bản soạn thảo / tham chiếu trước đó.

## Biên dịch (`latex_new`)

`main.tex` dùng `inputenc` + `[T5]{fontenc}` + `babel` (tiếng Việt) — **biên dịch bằng pdfLaTeX**, không dùng XeLaTeX/LuaLaTeX (dễ lệch phông hoặc lỗi hiển thị).

Trong thư mục `latex_new`:

```powershell
cd latex_new
pdflatex -interaction=nonstopmode -synctex=1 main.tex
pdflatex -interaction=nonstopmode -synctex=1 main.tex
```

- **Mục lục và số trang** đọc từ `main.toc` / `main.aux` của **lần biên dịch trước**, nên sau khi thêm hoặc sửa `\chapter` / `\include` cần **ít nhất hai lần** `pdflatex` (hoặc dùng `latexmk -pdf main.tex` để công cụ tự chạy đủ vòng).
- Nếu mục lục vẫn lệch: xóa các file sinh ra (`main.aux`, `main.toc`, `main.out`, `chuong_*.aux`, …) rồi chạy lại hai lần như trên.
- Trong `main.tex`, trang bìa được bọc bằng `\hypersetup{pageanchor=false}` rồi bật lại `pageanchor=true`, để tránh cảnh báo hyperref trùng đích `page.1` (do `titlepage` reset bộ đếm trang).

## Tuỳ chọn

```powershell
latexmk -pdf -interaction=nonstopmode main.tex
```

(MiKTeX/TeX Live cần có gói `latexmk`.)
