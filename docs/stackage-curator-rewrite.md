`stackage-curator` is the tool used by the Stackage Curator team to:

* Calculate Stackage Nightly and LTS Haskell build plans
* Check that these plans build and pass tests
* Generate Haddock files
* Upload the plans and Haddocks to the appropriate places:
    * S3 for the docs
    * Github for the YAML files

The current version of the tool was started a while ago by Michael
Snoyman, and has evolved significantly since then. It's currently
something of a mess, and is holding us back. This document is intended
to list out goals for a rewrite, and spell out the responsibilities
this tool needs to meet.

Note that there was an earlier document beginning to lay out some of
this, but it was not as comprehensive:
https://github.com/fpco/stackage-curator/blob/use-stack/PLAN2.md.

## Goals for rewrite

* __Use Stack for building__. Currently, `stackage-curator` builds by
  calling out to `Setup.hs` files directly. If we move over to Stack,
  we'll automatically inherit a bunch of nice functionality
  (especially caching), and have less maintenance burden.
* __Share library code with Stack__. There's an
  [open issue for Stack](https://github.com/commercialhaskell/stack/issues/3620)
  to separate off some general-purpose code around the Hackage index
  and cabal file parsing. The primary codebase to share the code with
  is `stackage-curator`. Doing this would reduce the
  `stackage-curator` maintenance burden.
* __Reduce size of the YAML files__. The YAML files currently include
  a _lot_ of information that most people don't need, such as the list
  of modules per package and reverse dependencies. This information is
  needed for `stackage-server`, but `stackage-server` itself could
  derive that information by parsing cabal files instead. We should
  reduce the unneeded information in these files to save user
  bandwidth.
* __Hash based downloads__. It would be nice to implement the
  necessary changes to make
  [hash based downloads](https://www.fpcomplete.com/blog/2018/01/hash-based-package-downloads-part-2-of-2)
  a reality, further increasing security and decreasing bandwidth
  usage.
* __Allow modifications to packages__. Currently, all packages we
  include are identical to their Hackage variants. We theoretically
  want to start allowing modifications to packages, and maybe even
  packages that are not on Hackage. (The latter is significantly more
  contentious; see
  [my SLURP blog post](https://www.snoyman.com/blog/2018/01/slurp).)
  The hash-based downloads helps with this somewhat, but I haven't
  fully fleshed out the ideas in the document below.

## Responsibilities for stackage-curator

### Calculate the build constraints

The `fpco/stackage` repo has a `build-constraints.yaml` file which
places a bunch of _constraints_ on the build. This includes which
packages are present, which upper bounds to respect, tests/benchmarks
to skip, etc. For a Stackage Nightly run, calculating these
constraints is simple: parse that YAML file.

For an LTS major release (e.g., `lts-11.0`), we do exactly the same
thing: take a current `build-constraints.yaml` and calculate a set of
constraints.

For an LTS minor release (e.g., `lts-11.5`), the process is
different. We take the previous LTS plan (e.g., `lts-11.4`) and
calculate the constraints from it. This means inheriting information
on the tests/benchmarks to skip, for example. It also means creating
some upper bounds. The rule here is:

* If the build plan includes an explicit version bound, like `mtl <
  2.2.3`, we respect that bound. As the curator team knows, we'll
  often add such explicit bounds due to some new breaking change.
* If no explicit version bound exists, then we follow PVP rules and
  keep the same major version. So, for instance, if we have
  `stm-2.3.4` in the previous LTS snapshot, and no explicit upper
  bounds, we'll generate a bound `stm < 2.4`.

__NOTE__ This implies that the version bound information needs to be
in the LTS build plans, even though it is only needed for
`stackage-curator` itself. An altnerative would be to store a separate
file with additional metadata that only `stackage-curator` cares
about, including tests/benchmarks to skip. We do not do this today.

Sometimes, when creating this constraints from a previous LTS, it
becomes necessary to add new, additional constraints. We do that today
via the `CONSTRAINTS` environment variable. I'm not convinced this is
the best way to handle things: it's tedious, and allows us no ability
to add comments as to why constraints were added. Here's a
theoretically better design, meant mostly as a strawman:

* `stackage-curator` spits out a new `build-constraints-lts-11.5.yaml`
  file, with all of the constraints it calculated.
* The curator team can edit that file directly to modify constraints,
  and add any extra metadata (such as via comments) desired.
* This file gets uploaded together with the actual `lts-11.5.yaml`
  file.

This step definitely deserves some discussion.

### Calculate the build plan

Now that we have all of the constraints, we can figure out an actual
plan. This will combine the constraints with the current state of the
Hackage package index to decide on specific package versions to build.

__NOTE__ We currently hard-code in `build-constraints.yaml` which
version of the `Win32` package we have. It would be nice if we had
some kind of a lookup table that correlated GHC versions to `Win32`
package versions, so we never accidentally forget to update this
version when changing GHC versions.

### Check the build plan

Confirm that the version bounds in the plan are compatible with each
other, and generate the error messages we all know and love/hate
:). We could theoretically stop doing this, see
[ignore version bounds](https://www.stackage.org/blog/2018/01/ignore-version-bounds)
post. We could add explicit "ignore version bounds" metadata (I think
Gershom brought up this option).

One idea for improvement: currently, `stackage-curator` only checks
plans for compatibility with Linux 64-bit. While we can't easily test
the builds themselves on Windows and OS X, we _can_ improve this step
to check that the plans work for Windows and OS X as well.

### Perform the build

Fairly straightforward to describe, harder to implement:

* Build all packages
* Build all non-skipped tests and benchmarks
* Build all Haddocks
* Fail if a test suite, benchmark or Haddock fails, unless it was an expected failure
* Create the "bundle", a tar.xz file with all of the generated Haddocks and some other metadata
    * __NOTE__ This bundle isn't strictly necessary anymore, since
      stackage-server gets all of the contents from S3 directly. See
      discussion at https://github.com/fpco/stackage/issues/2788.

### Upload Haddocks

Upload all of the Haddock files to S3. This is relatively
straightforward, assuming we figure out a good way to build the
Haddocks (see Gotchas). There's a good argument for S3 being a
terrible mechanism for this, however, and instead using some kind of
hash-based download to save a lot of space and make the uploads
faster. That kind of change is probably out of scope for now. Any
changes made here would need to be mirrored in stackage-server.

### Upload plan files

The plan files need to be added to the relevant Github repos. We use
signed commits to prove that it was the stackage-curator tool that
created the commits.

## Build scripts

It's tempting to put all of the logic above into a single execution of
the `stackage-curator` tool. However, that's not how it's currently
implemented. Instead, each of the actions above (more or less) is its
own command. This allows a set of bash scripts to call the commands
individually. This is nice for two reasons:

1. It lets us skip some parts of the run easily, such as the
   `NOPLAN=1` builds that do not recreate a build plan.
2. We're able to play with read/write permissions and access to secret
   files depending on the stage we're in. For example, we need a GPG
   key when uploading the plan files, but do _not_ want to have access
   to that key when performing the build, since a nefarious `Setup.hs`
   file or Template Haskell code could leak our secret information. We
   leverage Docker for this.

## Gotchas

* Stack doesn't know anything right now about:
    * expected test/benchmark failures
    * skipped tests/benchmarks
* Haddock generation from Stack probably generates links in a way that
  won't be amenable for the S3 upload. There's also likely an inherent
  mismatch with the way this will play with caching. I'm not sure what
  the right solution is for this. I've spent _many_ brain cycles
  thinking through this one, and it's prevented me from making
  progress. I recommend we start getting other bits of this working
  and then circle back to this problem.
* We need to think about backwards compatibility with older tools
  (especially older Stacks) pointing to fpco/lts-haskell and
  fpco/stackage-nightly. See the
  [PLAN2.md doc](https://github.com/fpco/stackage-curator/blob/use-stack/PLAN2.md)
  for some ideas around this.
