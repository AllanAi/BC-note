## sccache

建议安装 https://github.com/mozilla/sccache/ 加快rust编译速度，最好从源代码编译

```
cargo install --force --git https://github.com/mozilla/sccache
sccache --show-stats
```

## Why do you have to use `as_ref` rather than just `&`?

https://www.reddit.com/r/rust/comments/e03piv/why_do_you_have_to_use_as_ref_rather_than_just/

For `Option<_>`, `&` gives you `&Option<_>`, and `as_ref` gives you `Option<&_>`. Which one you want depends on what you're going to use it for.

For types which `impl Deref`, you should be able to in most cases use `&` and let deref coersion handle the rest (e.g. `&String` to `&str`).

