# Chuẩn Trailblazer trên Rails 8

> Phạm vi: cấu trúc thư mục, vai trò thành phần và cách gọi Operation trong ứng dụng Rails.

Công ty đã thử nghiệm cấu trúc này trên Rails 8. Ví dụ dùng API Trailblazer 2.1 để minh họa; hãy đối chiếu cú pháp với phiên bản gem trong `Gemfile.lock` trước khi áp dụng.

## Cấu trúc thư mục

Đặt code Trailblazer trong `app/concepts`, chia theo concept nghiệp vụ rồi theo loại thành phần:

```text
app/concepts/user/
├── activities/           # chỉ tạo khi có luồng dùng chung/cần tách
│   └── provision.rb
├── contracts/
│   └── register.rb
└── operations/
    ├── register.rb
    └── sso_callback.rb
```

Quy ước công ty dùng tên số nhiều cho thư mục và namespace:

- `operations/` → `User::Operations::Register`
- `contracts/` → `User::Contracts::Register`
- `activities/` → `User::Activities::Provision`

Tên concept thường ở số ít (`user`). Controller, route và view vẫn theo quy ước Rails. Chỉ tạo thư mục khi có thành phần tương ứng; không tạo sẵn cấu trúc rỗng.

## Vai trò thành phần

| Thành phần | Trách nhiệm |
| --- | --- |
| Controller | Nhận request, lọc input, gọi Operation và chọn HTTP response. |
| Operation | Điều phối một use case và trả kết quả cho caller. |
| Contract | Khai báo và xác thực input của use case; validation không thay thế authorization. |
| Activity/step | Mô tả các bước hoặc nhánh; tách Activity riêng khi có luồng cần dùng lại hoặc cần đặt tên/test độc lập. |
| Model | Persistence, quan hệ và invariant thuộc domain. |
| View/serializer | Trình bày dữ liệu đã chuẩn bị. |

Không điều phối cả workflow qua callback model. Truyền actor và dependency cần thiết vào Operation một cách tường minh; không để Operation phụ thuộc ngầm vào request hoặc session.

## Ví dụ: hai Operation dùng chung Activity

Đăng ký email và callback SSO đều cần chuẩn hóa email rồi tạo user. Mỗi Operation xử lý riêng input của điểm vào; Activity chứa phần quy trình thật sự dùng chung.

```ruby
# app/concepts/user/activities/provision.rb
module User::Activities
  class Provision < Trailblazer::Activity::Railway
    step :normalize_email
    step :create_user

    def normalize_email(ctx, email:, **)
      ctx[:normalized_email] = email.strip.downcase
    end

    def create_user(ctx, normalized_email:, **)
      user = User.new(email: normalized_email)
      ctx[:user] = user
      user.save
    end
  end
end
```

```ruby
# app/concepts/user/operations/register.rb
module User::Operations
  class Register < Trailblazer::Operation
    step Subprocess(User::Activities::Provision)
  end
end
```

Operation gọi Activity bằng `Subprocess`. Context đầu vào như `email` được truyền vào; các giá trị Activity ghi vào context như `normalized_email` và `user` có thể được caller đọc từ result:

```ruby
result = User::Operations::Register.call(email: " Sam@Example.com ")

result.success?                 # true nếu user được lưu
result[:normalized_email]       # "sam@example.com"
result[:user]                   # user vừa tạo
```

`User::Operations::SsoCallback` có thể gọi cùng Activity sau khi trích email từ payload SSO. Nếu chỉ có một luồng ngắn, tuyến tính và không tái sử dụng, đặt các bước trực tiếp trong Operation.

## Quy ước khi triển khai

- Một Operation đại diện cho một use case, đặt tên theo hành động: `Register`, `Update`, `Publish`.
- Một Contract tương ứng với input của use case; tách Create/Update khi payload hoặc validation khác nhau.
- Mỗi step làm một việc; đặt tên theo hành động và xác định rõ dữ liệu đầu vào/đầu ra.
- Thiết kế các kết quả cần caller xử lý như `success`, `failure` hoặc `not_found`; nối rõ outcome tùy chỉnh của Activity con.
- Đặt side effect ở bước phù hợp sau persistence; xác định retry/idempotency theo yêu cầu use case.
- Chỉ thêm Activity khi có luồng cần tái sử dụng hoặc cần tách để đọc, test độc lập. Không tạo Activity cho từng thao tác nhỏ.
- Giữ API của gem đúng nguyên bản. Quy ước số nhiều áp dụng cho thư mục và namespace nội bộ, không đổi tên macro hoặc key do gem định nghĩa.

## Controller gọi Operation

Controller chỉ chuyển input đã lọc và ánh xạ kết quả sang HTTP:

```ruby
def create
  result = BlogPost::Operations::Create.call(
    params: { blog_post: blog_post_params.to_h },
    current_user: current_user
  )

  if result.success?
    redirect_to blog_post_path(result[:model])
  else
    @form = result["contract.default"]
    render :new, status: :unprocessable_entity
  end
end
```

Key result và cách gọi có thể khác theo cấu hình/phiên bản gem. Adapter `trailblazer-rails` cũng có helper riêng; xác minh với dependency đang khóa trước khi dùng. Không truyền toàn bộ `params` nếu Operation chỉ cần một phần input.

## Kiểm tra nhanh khi review

- [ ] File và namespace khớp nhau; `Operations`, `Contracts`, `Activities` dùng số nhiều.
- [ ] Operation có một mục tiêu; controller chỉ xử lý HTTP.
- [ ] Validation được phân biệt với authorization.
- [ ] Failure/outcome quan trọng và side effect được xử lý, có test phù hợp.
- [ ] API khớp phiên bản gem trong `Gemfile.lock`.

Quy trình từ phân tích use case đến hoàn tất review nằm trong [Quy trình triển khai một use case](use-case-workflow.md).

Ví dụ unit test Operation bằng Minitest nằm trong [Unit test Trailblazer bằng Minitest](minitest-trailblazer.md).

## Nguồn

- [Trailblazer 2.1 — Activity tutorial](https://ornate-heliotrope-1c15e2.netlify.app/2.1/docs/tutorials/activity)
- [Trailblazer 2.1 — Operation](https://trailblazer.to/2.1/docs/operation/)
- *Building your own authentication library with Trailblazer*, Nick Sutterer, 2021 — tài liệu tham khảo; ví dụ auth/migration trong sách không phải cấu hình chuẩn Rails 8 của công ty.

Tài liệu Trailblazer tham khảo thuộc nhánh 2.1; chưa xác nhận phiên bản gem của từng dự án. Context7 không khả dụng trong phiên soạn.
