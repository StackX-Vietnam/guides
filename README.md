# Rails / Trailblazer — Tài liệu chuẩn công ty

> Bộ tài liệu này chuẩn hóa cách tổ chức và phát triển ứng dụng Rails dùng Trailblazer, dựa trên trải nghiệm nội bộ với Rails 8.

## Tài liệu

- [Chuẩn Trailblazer trên Rails 8](docs/rails/trailblazer-standard.md) — cấu trúc thư mục, vai trò thành phần và ví dụ Operation gọi Activity.
- [Quy trình triển khai một use case](docs/rails/use-case-workflow.md) — từ xác định input/outcome đến kiểm thử và review.
- [Unit test Trailblazer bằng Minitest](docs/rails/minitest-trailblazer.md) — ví dụ kiểm tra Operation, Activity, validation và side effect.

## Skills cho Claude Code

Thư mục [`skills/`](skills/) chứa các skill rút gọn từ tài liệu trên (viết bằng tiếng Anh):

- [`trailblazer-rails-standard`](skills/trailblazer-rails-standard/SKILL.md) — cấu trúc, vai trò thành phần, controller gọi Operation.
- [`trailblazer-use-case-workflow`](skills/trailblazer-use-case-workflow/SKILL.md) — quy trình triển khai và review một use case.
- [`trailblazer-minitest`](skills/trailblazer-minitest/SKILL.md) — unit test Operation/Activity bằng Minitest.

Để dùng, liên kết từng thư mục vào `~/.claude/skills/`:

```sh
ln -s "$PWD/skills/trailblazer-minitest" ~/.claude/skills/trailblazer-minitest
```

## Lưu ý phiên bản

Ví dụ Trailblazer trong tài liệu tham khảo nhánh 2.1. Trước khi áp dụng cú pháp hoặc cài gem, đối chiếu phiên bản đang khóa trong `Gemfile.lock` của ứng dụng.
