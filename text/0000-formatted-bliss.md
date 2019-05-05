- Feature Name: `formatted_bliss`
- Start Date: 2019-05-05
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

1. Introduce a formatting convention (hence: *"the style*")
   to `rust-lang/rust` (hence: *"the repository*") in a `rustfmt.toml` file.

2. Enforce the style with `src/tools/tidy` in the repository's continuous integration (CI).

3. Add a command `fmt` to rustbuild (`./x.py`).
   The command formats the repository according to the style.

4. Add a command `@rustbot fmt` on GitHub which will add a commit to a
   pull request (PR) such that the applied to PR will pass CI in terms of the style.

5. Add a command `install-fmt-hook` which will install a pre-commit git-hook for
   formatting.

# Motivation
[motivation]: #motivation

[Rust Style Guide]: https://github.com/rust-lang/rfcs/tree/master/style-guide#rust-style-guide

The [Rust Style Guide] notes:

> Formatting code is a mostly mechanical task which takes both time and mental effort. By using an automatic formatting tool, a programmer is relieved of this task and can concentrate on more important things.
>
> Furthermore, by sticking to an established style guide (such as this one), programmers don't need to formulate ad hoc style rules, nor do they need to debate with other programmers what style rules should be used, saving time, communication overhead, and mental energy.
>
> Humans comprehend information through pattern matching. By ensuring that all Rust code has similar formatting, less mental effort is required to comprehend a new project, lowering the barrier to entry for new developers.
>
> Thus, there are productivity benefits to using a formatting tool (such as rustfmt), and even larger benefits by using a community-consistent formatting, typically by using a formatting tool's default settings.

Simply put, we wish to unlock these benefits for the repository.
In particular, as the repository is large and is read and written
to daily by many developers, we expect the benefits in terms of a
consistent style to be large.

Moreover, to lower the barrier to entry for working on compiler development,
we wish to adopt the standard Rust Style Guide as much as is possible.
This will ensure that most Rust developers feel at home with style inside the repository.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

See the [summary] for a brief explanation or read below for a detailed technical
description of what this proposal entails.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

To disrupt normal daily development as little as possible in the repository,
the proposal has been divided into steps which in part can and should be
executed independently of each other.

## 1. Introducing *the style*

As noted in the [motivation], we wish to make the repository where any Rust
developer feels at home. To that end, we introduce and adopt the standard
Rust Style Guide into the repository.

*The style* is defined in terms of a `rustfmt.toml` file (hence: *the style file*):

```toml
version = "Two"
use_small_heuristics = "Max"
```

The style file is located at the top of the repository.

This step has already been completed.

## 2. Enforce the style

The `tidy` tool, defined in `src/tools/tidy`, is run each time a new PR is created
or when a commit is added to said PR. The tool is also run when a PR,
approved for merging with e.g. `@bors r=reviewer`, is tested for merging.

To `tidy`, new logic is added that checks whether the code is conformant with
the style. This will result in a PR being rejected when changes in said PR does
not conform to the style.

To facilitate incremental enforcement of the style and to disrupt development
minimally inside the repository, the `tidy` tool will only enforce the style
in folders with an `.enforcefmt` file. In such folders, the style will be
checked for all files in the immediate folder as well as all subfolders.

It is expected that applying the style throughout the repository will generate
large diffs. Since it is unfeasible to effectively review so many changes,
to avoid any security risks, we will only accept PRs applying the style by
regulars who themselves have reviewer rights in the repository.

Eventually, when the entire repository has been formatted,
the `.enforcefmt` trigger may be removed in favor of checking everything.

## 3. Add the `./x.py fmt` command

A command `fmt` is added to rustbuild (`./x.py`).
The command will format the repository according to the style
but only in folders with `.enforcefmt` files as described above.

## 4. Add a `@rustbot fmt` command

Sometimes compiler developers do not have the time, or may simply forget,
to apply `./x.py fmt`. In other situations, the developer may not have a
computer with a compiler and git handy. It may also happen that the PR
is perfectly good aside from formatting. For these situations, a
`@rustbot fmt` command is provided with which the author, a reviewer,
or a T-release member can simply ask `@rustbot` to run `./x.py fmt`
on their PR and add a commit for that.

## 5. Add the `./x.py install-fmt-hook`

We add a command `install-fmt-hook` to rustbuild.
This command will install a pre-commit git-hook.
This git-hook will check that the formatting in the commit is good.

# Drawbacks
[drawbacks]: #drawbacks

There are two notable drawbacks to this proposal:

1. Formatting commits will need to be added to the repository.
   This will be in the way of checking who is to `git blame`
   for a certain change to the compiler. We should try to
   limit the cost of this to as one-time as possible.

   Other than that, it is unavoidable to have one such formatting
   commit in the entire repository if we wish to enforce a style.
   Moreover, people do drive-by commits for style changes sometimes
   so this cost already exists to some extent. By enforcing style
   mechanically, we will have fewer such PRs.

2. More impediments to merging a PR.

   Since we now mechanically check and enforce a style,
   some developers may feel an additional burden when hacking on the compiler.

   Moreover, automatic enforcement of style may make rollups harder to land
   as a PR which would otherwise be guaranteed to merge may now fail due to
   incorrect formatting. To mitigate that risk and to not overburden
   the `bors` queue more than it is today, it is important that CI will check
   for style problems early on.

   In both cases, the `@rustbot fmt` command should help reduce the burden
   of automatic style checking.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## The proposed style

[small]: https://github.com/rust-lang/rustfmt/blob/master/Configurations.md#use_small_heuristics
[churn]: https://internals.rust-lang.org/t/running-rustfmt-on-rust-lang-rust-and-other-rust-lang-repositories/8732/81

As aforementioned, the standard Rust Style Guide is adopted.
This style is adopted to make the repository feel idiomatic in terms of style.

However, we make a small modification to the style guide.
In particular, we enable [`use_small_heuristics = "Max"`][small].
This is a compromise with a default style since:

> The style team already had that discussion and agreed that that kind of
> `if` should be on one line; I think that’s interacting badly with the rules
> regarding what kind of expression can comprise a one-line match arm.

We also want to enable `use_small_heuristics` to [reduce the amount of churn][churn]
in the repository.

## Automatic enforcement

In this proposal, automatic enforcement with `tidy` is proposed for CI.
An alternative to this would be to apply periodical style PRs using `./x fmt`.
The benefit of that would be to reduce the burden upon new contributors.
However, this has notable drawbacks:

+ Someone has to remember to apply these periodical reformattings.
  It is likely that this would be forgotten or would result in notable churn
  including having more people rebase their PRs.

+ Many reformatting commits negatively impacts `git blame`.

A more realistic to automatic enforcement would be for `homu` to
automatically apply `./x.py fmt` in its own commit and add to the repository.
This would result in the repository always being properly formatted
and so rebases due to reformatting commits would be less frequent.
However, `git blame` would still be negatively impacted by having automatic commits. Meanwhile, `@rustbot fmt` would still result in those formatting commits.
However, those can be reintegrated when rebasing a PR.

## Rollout plan

This RFC proposes an incremental rollout plan rather than doing all
the reformatting at once. This has notable advantages:

+ We can test out the style on specific places before committing to it everywhere.
+ We can test out CI with respect to enforcement more.
+ The normal flow of development in the repository is disrupted less.
  If we were to apply reformatting in a single commit,
  we would need to close the homu tree.

We can still be flexible with the incremental enforcement plan in terms of
how granular we are with the placement of `.enforcefmt` files.
For example, we can start with the crates in the repository that are least
often changed and then we can move on from there.

# Prior art
[prior-art]: #prior-art

The main prior art from the repository itself is `tidy`.
We already enforce certain style aspects with that tool.
For example, we do not allow lines with trailing whitespace.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

# Future possibilities
[future-possibilities]: #future-possibilities

1. We could add a spell-checker to `tidy`.
2. We could have `./x.py fmt` fix more things that `tidy` complains about today.
