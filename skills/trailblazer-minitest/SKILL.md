---
name: trailblazer-minitest
description: Use when writing or reviewing Minitest tests for Trailblazer Operations or Activities in Rails 8, using trailblazer-test helpers assert_pass/assert_fail, or testing validation failures, authorization, persistence and enqueued jobs.
---

# Unit Testing Trailblazer with Minitest

Source guide: `docs/rails/minitest-trailblazer.md`. Operation tests cover outcome and observable effects; request/integration tests still cover controller/HTTP.

## Setup

- Path: `test/concepts/<concept>/<plural_type>/<name>_test.rb`, e.g. `test/concepts/blog_post/operations/create_test.rb`.
- Optional gem for `assert_pass`/`assert_fail`; add to the test group and check the locked version (docs describe API from 1.0.0):

```ruby
group :test do
  gem "trailblazer-test", ">= 1.0.0", "< 2.0.0"
end
```

- Run: `bin/rails test test/concepts/blog_post/operations/create_test.rb` (confirm paths/suite command against the repo).

## Success + persistence

```ruby
require "test_helper"

class BlogPost::Operations::CreateTest < ActiveSupport::TestCase
  Trailblazer::Test.module!(self)

  test "creates a blog post from valid input" do
    input = { params: { blog_post: { title: "Trailblazer on Rails", body: "Example" } } }

    assert_difference("BlogPost.count", 1) do
      result = assert_pass(
        BlogPost::Operations::Create, input,
        title: "Trailblazer on Rails", body: "Example"
      )
      assert_predicate result[:model], :persisted?
    end
  end
end
```

`assert_pass` calls the Operation, checks the `success` terminus, returns the result; 3rd arg compares attributes with `result[:model]`. Examples assume nested `params[:blog_post]`, model at `result[:model]`, contract at `result[:"contract.default"]` — adapt to the project.

## Validation failure

Assert outcome **and** the failing field; a bare "it failed" can pass for the wrong reason.

```ruby
test "rejects a blog post without a title" do
  input = { params: { blog_post: { title: "", body: "Body" } } }

  assert_no_difference("BlogPost.count") do
    result = assert_fail(BlogPost::Operations::Create, input, [:title])
    assert_not result[:model].persisted?
  end
end
```

Assert exact error text only if the wording is a committed UI contract.

## Edge cases and side effects

One test per meaningful case: not found, not authorized, conflict, repeat call, job enqueued. For denial, assert failure plus no data change; if callers distinguish reasons, assert on public error/result keys, never private steps.

```ruby
assert_enqueued_with(job: PublishBlogPostJob) do
  assert_pass(BlogPost::Operations::Publish, input)
end
```

## Activity in isolation

The helper returns the Activity's context (not an Operation result). Use when the Activity has its own interface/branches, and still cover one real caller via an Operation test.

```ruby
ctx = assert_pass(User::Activities::Provision, email: " Sam@Example.com ")
assert_equal "sam@example.com", ctx[:normalized_email]
assert_predicate ctx[:user], :persisted?
```

## Without trailblazer-test

```ruby
result = BlogPost::Operations::Create.call(input)
assert_predicate result, :success?
assert_equal "Rails", result[:model].title
```

## Rules

- One behavior per test; use the project's fixtures/factories.
- Assert outcome plus observable data/side effects; never internal step call order.
- On failure, confirm no persistence or unwanted side effects.
- `assert_pass?`/`assert_fail?` print a trace for debugging only; never assert on trace content.
- HTTP status, redirect/render and filtered params belong in request/integration tests.
