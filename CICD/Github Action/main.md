Mỗi workflow của GitHub Actions được cấp một GitHub token tự động (GITHUB_TOKEN) để:
- Truy cập repo (đọc hoặc ghi),
- Gọi GitHub API,
- Trigger workflows khác…

Tuy nhiên, từ giữa 2022, GitHub thay đổi cơ chế bảo mật: Mặc định GITHUB_TOKEN không có toàn quyền, mà bạn phải tự khai báo rõ permission nào cần dùng trong phần permissions
```
permissions:
  id-token: write
  contents: read
```

Giải thích hai quyền trong permission
- `id-token: write`: Cho phép workflow yêu cầu (request) một OIDC (OpenID Connect) token từ GitHub. Quyền này bắt buộc nếu bạn dùng “OpenID Connect (OIDC)” authentication với AWS.
- `contents: read` : Cho phép workflow đọc nội dung repository, ví dụ: Checkout code (actions/checkout@v4 cần quyền này), đọc file, commit, tag... Ở đây được đặt là read (không phải write), nghĩa là workflow chỉ cần đọc, không chỉnh sửa nội dung repo. Nếu bạn không khai rõ quyền này, GitHub có thể cấp quyền write theo mặc định (trước kia), tiềm ẩn rủi ro bảo mật vì workflow có thể push hoặc thay đổi nội dung repo.
