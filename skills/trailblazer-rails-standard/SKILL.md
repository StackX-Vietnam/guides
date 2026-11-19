---
name: trailblazer-rails-standard
description: Use when creating or reviewing Trailblazer code in a Rails 8 app (app/concepts, Operation, Contract, Activity, Subprocess), deciding where a file or namespace belongs, or wiring a controller to an Operation.
---

# Trailblazer Standard on Rails 8

Company conventions for structure, component roles and controller integration. Source guide: `docs/rails/trailblazer-standard.md` in the guides repo.

**Version check first:** examples target Trailblazer 2.1. Verify DSL against the version locked in the app's `Gemfile.lock` before applying syntax or adding gems.

## Layout

```text
app/concepts/user/
├── activities/    # only when a flow is shared or needs independent naming/tests
│   └── provision.rb
├── contracts/
│   └── register.rb
└── operations/
    ├── register.rb
    └── sso_callback.rb
```

- Concept name singular (`user`); folder and namespace for component type are **plural**: `User::Operations::Register`, `User::Contracts::Register`, `User::Activities::Provision`.
- Plural applies only to our folders/namespaces — never rename gem macros or keys.
- Create folders only when a component exists. No empty scaffolding.
- Controllers, routes, views follow normal Rails conventions.

## Roles

| Component | Responsibility |
| --- | --- |
| Controller | Receive request, filter params, call Operation, choose HTTP response |
| Operation | Orchestrate one use case, return outcome |
| Contract | Declare/validate input. Validation is not authorization |
| Activity/step | Steps or branches; separate Activity only for reuse or independent naming/test |
| Model | Persistence, relations, domain invariants |
| View/serializer | Present prepared data |

- Never orchestrate a workflow through model callbacks.
- Pass actor and dependencies explicitly; Operations must not reach into request/session.

## Example: two Operations sharing an Activity

```ruby
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

module User::Operations
  class Register < Trailblazer::Operation
    step Subprocess(User::Activities::Provision)
  end
end
```

`SsoCallback` extracts the email from the SSO payload, then calls the same Activity. A short linear one-off flow: put steps directly in the Operation, no Activity.

## Conventions

- One Operation = one use case, named as an action: `Register`, `Update`, `Publish`.
- One Contract per use-case input; split Create/Update when payload or validation differs.
- One job per step, named as an action, with clear inputs/outputs.
- Model outcomes callers must handle (`success`, `failure`, `not_found`); wire custom Activity outcomes explicitly.
- Side effects go after persistence; decide retry/idempotency per use case.
- No Activity per tiny operation.

## Controller

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

Result keys vary by config/version; `trailblazer-rails` has its own helpers — verify against the locked gem. Pass only the params the Operation needs, not all of `params`.

## Review checklist

- [ ] File path and namespace match; `Operations`/`Contracts`/`Activities` plural
- [ ] Operation has one goal; controller handles only HTTP
- [ ] Validation distinguished from authorization
- [ ] Important failures/outcomes and side effects handled and tested
- [ ] API matches `Gemfile.lock` version

Related: `trailblazer-use-case-workflow` (process), `trailblazer-minitest` (tests).
