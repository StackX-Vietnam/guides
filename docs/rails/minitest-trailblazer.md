# Unit test Trailblazer bằng Minitest

> Phạm vi: kiểm thử Operation/Activity trực tiếp trong ứng dụng Rails 8.

Rails dùng Minitest mặc định. Test Operation tập trung vào outcome và tác động quan sát được: dữ liệu được lưu, không được lưu khi thất bại, và side effect cần thiết. Request/integration test vẫn kiểm tra controller và HTTP response; chúng bổ sung cho unit test Operation.

## Cấu hình

Tạo test trong `test/concepts` theo concept và namespace số nhiều của công ty:

```text
test/concepts/blog_post/operations/create_test.rb
```

Để dùng helper `assert_pass` và `assert_fail` trong tài liệu Trailblazer, thêm gem `trailblazer-test` vào nhóm test trong `Gemfile`, cài dependency theo quy trình dự án và kiểm tra phiên bản đã khóa. Tài liệu Trailblazer 2.1 mô tả API từ phiên bản 1.0.0.

```ruby
# Gemfile
group :test do
  gem "trailblazer-test", ">= 1.0.0", "< 2.0.0"
end
```

Nếu dự án không muốn thêm gem, phần cuối có ví dụ chỉ dùng assertion chuẩn của Minitest.

## Kiểm thử thành công và persistence

Ví dụ giả định `BlogPost::Operations::Create` nhận payload nested dưới `params[:blog_post]`, trả model ở `result[:model]`, và có contract mặc định tại `result[:"contract.default"]`.

```ruby
# test/concepts/blog_post/operations/create_test.rb
require "test_helper"

class BlogPost::Operations::CreateTest < ActiveSupport::TestCase
  Trailblazer::Test.module!(self)

  test "creates a blog post from valid input" do
    input = {
      params: {
        blog_post: {
          title: "Trailblazer on Rails",
          body: "Operation test example"
        }
      }
    }

    assert_difference("BlogPost.count", 1) do
      result = assert_pass(
        BlogPost::Operations::Create,
        input,
        title: "Trailblazer on Rails",
        body: "Operation test example"
      )

      assert_predicate result[:model], :persisted?
    end
  end
end
```

`assert_pass` gọi Operation, kiểm tra terminus `success`, và trả result để test có thể kiểm tra thêm. Các thuộc tính truyền ở tham số thứ ba được so sánh với `result[:model]`. `assert_difference` xác nhận tác động lên cơ sở dữ liệu.

## Kiểm thử validation failure

Kiểm tra cả outcome lẫn trường có lỗi. Không chỉ assert thất bại, vì Operation cũng có thể thất bại do nguyên nhân khác.

```ruby
test "rejects a blog post without a title" do
  input = {
    params: {
      blog_post: {
        title: "",
        body: "Body text"
      }
    }
  }

  assert_no_difference("BlogPost.count") do
    result = assert_fail(
      BlogPost::Operations::Create,
      input,
      [:title]
    )

    assert_not result[:model].persisted?
  end
end
```

Danh sách `[:title]` yêu cầu contract mặc định báo lỗi ở trường `title`. Nếu contract/result key của dự án khác, thay theo cấu hình thực tế. Chỉ kiểm tra chính xác thông điệp lỗi khi câu chữ đó là một phần giao diện cần cam kết; nếu không, kiểm tra trường lỗi sẽ ít giòn hơn.

## Kiểm thử input biên hoặc side effect

Tạo một test riêng cho từng trường hợp nghiệp vụ quan trọng: không tìm thấy bản ghi, không có quyền, xung đột dữ liệu, gọi lặp, job được enqueue. Dùng helper trả result để kiểm tra phần cần thiết:

```ruby
test "does not create a post when the actor is not authorized" do
  input = {
    params: { blog_post: { title: "Private", body: "Text" } },
    current_user: users(:reader)
  }

  assert_no_difference("BlogPost.count") do
    assert_fail(BlogPost::Operations::Create, input)
  end
end
```

Ví dụ xác nhận Operation thất bại và không tạo dữ liệu. Nếu caller cần phân biệt lý do bị từ chối, hãy assert trên error/result key công khai mà Operation của dự án cung cấp; không dựa vào step riêng tư.

Với job, kiểm tra enqueue thay vì gọi method riêng tư của step:

```ruby
assert_enqueued_with(job: PublishBlogPostJob) do
  assert_pass(BlogPost::Operations::Publish, input)
end
```

## Kiểm thử Activity độc lập

Có thể gọi helper với Activity; theo tài liệu Trailblazer, helper trả context của Activity thay vì Operation result. Dùng test này khi Activity có interface/nhánh đáng kiểm tra độc lập; vẫn nên có test Operation bao phủ ít nhất một caller thực tế.

```ruby
test "normalizes email before provisioning a user" do
  ctx = assert_pass(
    User::Activities::Provision,
    email: " Sam@Example.com "
  )

  assert_equal "sam@example.com", ctx[:normalized_email]
  assert_predicate ctx[:user], :persisted?
end
```

## Chỉ dùng Minitest, không dùng `trailblazer-test`

Nếu không cài helper gem, gọi Operation trực tiếp và assert outcome bằng API Minitest:

```ruby
test "creates a blog post from valid input" do
  input = {
    params: {
      blog_post: { title: "Rails", body: "Example" }
    }
  }

  assert_difference("BlogPost.count", 1) do
    result = BlogPost::Operations::Create.call(input)

    assert_predicate result, :success?
    assert_equal "Rails", result[:model].title
  end
end
```

## Quy ước test

- Đặt test cùng concept và loại thành phần trong `test/concepts`.
- Mỗi test nêu rõ một hành vi; dùng fixture/factory theo quy ước sẵn có của dự án.
- Assert outcome cùng dữ liệu/side effect quan sát được; tránh kiểm tra thứ tự gọi step nội bộ.
- Với failure, xác nhận không có persistence hoặc side effect không mong muốn.
- Dùng `assert_pass?`/`assert_fail?` để in trace khi debug; không assert dựa trên nội dung trace.
- Giữ request/integration test cho HTTP status, redirect/render và params được lọc.

## Chạy test

```sh
bin/rails test test/concepts/blog_post/operations/create_test.rb
```

Đây là lệnh Rails tiêu chuẩn; xác nhận đường dẫn test và cách chạy suite theo cấu hình repository.

## Nguồn và giới hạn

- [Trailblazer 2.1 — Testing](https://trailblazer.to/2.1/docs/test/) — `trailblazer-test`, Minitest helpers `assert_pass`/`assert_fail`, result/context và Suite.
- [Rails 8.1 Guides — Testing Rails Applications](https://guides.rubyonrails.org/v8.1/testing.html) — `ActiveSupport::TestCase`, Minitest assertions và `assert_difference`.

Trailblazer test docs nêu gem `trailblazer-test` từ 1.0.0; README gem giới hạn nhánh dưới 2.0.0. Do chưa có `Gemfile.lock` của ứng dụng Rails 8 cụ thể, cần xác nhận gem tương thích và API với dependency đã khóa. Context7 MCP không khả dụng trong phiên soạn; nội dung được đối chiếu với tài liệu Trailblazer và Rails liên kết ở trên.
