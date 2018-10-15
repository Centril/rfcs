- Feature Name: `gradual_struct_init`
- Start Date: 2018-10-11
- RFC PR: _
- Rust Issue: _

# Summary
[summary]: #summary

Permit the gradual initialization of structs like so:

```rust
struct Point<T> { x: T, y: T }

let pt: Point<_>;
pt.x = 42;
pt.y = 24;

drop(pt);
```

# Motivation
[motivation]: #motivation

TODO

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

TODO

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

TODO

# Drawbacks
[drawbacks]: #drawbacks

TODO

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

TODO

# Prior art
[prior-art]: #prior-art

TODO

# Unresolved questions
[unresolved-questions]: #unresolved-questions

TODO

# Future work
[future-work]: #future-work

TODO
