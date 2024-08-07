# Introduction

This repository contains minimal Python, Rust, and Haskell projects
that provide hello-world-ish projects where we can demonstrate and
test the capabilities of CVE scan tools for these languages.

We also describe what github's *Dependabot* can bring to the table.
(See below, *Dependabot* is generally not highly recommended).

# Scanning for CVEs, General Considerations
## Terminology

Let's define some general terminology here to allow for describing build
tools and CVE scan tools *generically* across ecosystems/languages:
  1. *package constraint description* (or *PCD*): e.g., p.cabal,
     Cargo.toml, pyproject.toml
  2. *resolver*: takes the *PCD*, the *database of package
     specifications* and generates the package list. The resolver is
     not necessarily deterministic, it is stateful. This is typically
     part of our "build" ecosystem/tool.
  3. *resolver output file* (or, package list). (E.g., `Cargo.lock`,
     `requirements.txt`, `cabal.project.freeze`) This is the output of
     the resolver, it is a *complete* listing of all required packages
     that are consistent with (1) *package constraint description*;
     the resolver might override constraints in (1), we expect that
     the resolver outputs such information.
  4. *"build tool proper"* : the part of our build tool that is invoked
     after the resolver.
  5. *scan tool* : (or *audit tool*): this tool is *not* about
     scanning project code, rather this tool checks **package**
     dependencies to determine if any used package version is the
     subject of a CVE.

## Knowing Your Scan Tool

Some questions we should know the answers to regarding any *scan tool*
we use:

 - Dependency on the resolver: Does it input the (1) *PCD* or (3) the
   *resolver output file*?

 - If the scan tool relies on the *package constraint description* (PCD),
   - Does it call the resolver?  Does it give us the ability to
     configure the resolver (in a way that *exactly* matches the real build).
   - Or does it try to duplicate the resolver logic?  Is it accurate? Does
     it correctly determine transitive package dependencies?

 - Does the scan tool make any attempt to ensure the *PCD* is accurate
   with respect to the packages referred to by the source code?  E.g.,
   Does the scan tool check that the PCD has no extraneous packages?
   Does the scan tool check that the PCD has no missing packages?

   Both of these would be hard to determine without doing a full build
   of the project.  Extraneous packages may be an inconvenience (and
   may lead to false positives), but missing packages can result in
   false negatives, possibly leading to using code with vulnerabilities!

 - Does the scan tool try to be more fine-grained than package level,
   e.g., by attempting to look at actual function calls to package
   dependencies?

 - If we have a non-trivial build, can the scan tool be configured to
   work exactly like the non-trivial build?

 - Can we setup the scan tool to **only** check package dependencies
   of the target library and/or executable?  Can we configure the scan
   tool to separately check the dependencies of the build-time only
   packages or test-time only dependencies?

## Accurate Scans

Clearly, we want accurate scans, but easier said than done.  Note the
following:

Causes of false positives (i.e., CVEs reported when not applicable)
 - Coarse grain dependencies (only package level but not checking
   whether the actual vulnerability can actually be invoked by looking
   at the code).
 - Not taking architecture into account for CVEs that are architecture
   dependent (similar to the previous).
 - If our package constraint description file (and thus the resolver
   output file) has packages that are not actually used in the source code.

Causes of false negatives (i.e., CVE vulnerabilities that are not reported)
 - Not determining and checking the indirect package dependencies
 - Not re-running scan tool when the CVE database is updated
 - An incomplete *PCD* file (which may not be determinable without
   full build).

Causes of either or both
 - *scan tool* looking at the *PCD* file and making assumptions about
   the actual packages that get resolved by the *resolver*.

## State and Other Gotchas

Besides some of the issues already brought up, note that the
*resolver* is stateful, and it may not be ``deterministic'':
 - It depends on the state of the "package database" at the
   time of the tool execution.  (A related question, do we have
   guarantees that package information for a given version is
   immutable?)
 - Is there a deterministic algorithm that all *resolvers* follow?

The scan tool often calls the resolver, and note
 - the *scan tool* depends on the state of the CVE database, at the
   time of the tool execution.

## Best Practices

In light of the preceding observations, we propose the following "best
practices" (whenever feasible and supported by the tools):

  - Run the *resolver* used by the real build tool.
    - Ensure that no warnings which indicate constraints were over-ridden
      (this is not required here, but it helps more heuristic tools
      such as Dependabot).
    - Capture the result in a *resolver output file*.
    - Maintain this *resolver output file* in version control.

  - Run the *scan tool* with the *resolver output file* as input.

  - Note that in the general case, different compilers and different
    target architectures may have different package constraints and
    "resolutions", so we need ensure that we do the resolve and
    the scan for every *build variation*.

  - Make efforts to avoid "dead packages" in the *PCD* file.  (A way
    to automate this?  Do some build tools help us here?)  The dead
    package may not be explicitly in the *PCD* file, for instance,
    we might these dependencies,
     ```
     a -> b
     a -> c
     ```
    if the *PCD* indicates `a` but the source code only uses `b`, then we
    would consider `c` to be dead.

  - Ensure no "missing packages" in the *PCD* file: a full build
    is the best way to ensure this.  So, in the final analysis a full
    build (for each *build variation*) is the only way to determine
    correctness here.

  - After the above are done, update the *resolver output file* in
    version control; we would like to maintain the invariant that
    *resolver output file* is CVE warning free.

  - Capture the output of the CVE *scan tool* in a *scan tool results*
    file (one for each language and architecture); these files should
    be under version control.

      Ideally, we'd like a timestamp or hash indicating the version of
      the CVE database (used by the *scan tool*) as well as timestamp
      or hash for the version of the package database used (by the
      *resolver*).

  - What to do with *scan tool results*?

      Despite the desire for the *scan tool results* to be effectively
      null, we are likely to have CVEs that we want to ignore for
      various reasons (minor, inapplicable, waiting on upstream fixes,
      etc.).

      Some some *scan tools* allow for filtering or ignoring CVEs,
      unfortunately this solution is specific to the language
      ecosystem.

      One solution here would be
       1. catenate all the language specific *scan tool results* into
          a summary results file.
       2. have a separate *CVE status file* to capture a list of
          tuples of type`(CVE-ID, Date, Status)`
          where the `Status` could be a type along these lines:
          ```
          data Status = NoFix
                          -- no fixed package yet.
                      | NotApplicable String
                          -- why the CVE vuln. does not affect our code.
                      | Ignored String
                          -- why we can ignore this.
                      | Unresolved
                          -- a CVE remaining to be addressed.
          ```
       3. from the last two, we generate a report to roll up the CVE
          data.

# Scanning for CVEs in Rust, Python, and Haskell

For each of our three languages, we attempted to pick the best option
for a CVE scan tool.  See the associated directories and documentation
for further details:

- Rust CVE scanning with `cargo-audit`:
  See [Rust CVE Scans](rust-project/README.md).

- Python CVE Scanning with `pip-audit`.
  See [Python CVE Scans](py-project/README.md).

- Haskell CVE Scanning with MangoIV's `cabal-audit`.
  See [Haskell CVE Scans](hs-project/README.md).

# Dependabot
## In a Nutshell

Dependabot is a tool integrated into GitHub for keeping dependencies
up-to-date automatically, it also notifies when dependencies have
published security vulnerabilities (CVEs).

Dependabot optionally allows for configuration via a
`.git/dependabot.yml` file.  This file primarily indicates directories where
there is package and which 'package-ecosystem' ("cargo", "pip", etc.)
is associated with the directory.

Dependabot provides a high degree of automation:
 - It understands and supports a dozen or more languages.
 - It generates PRs whenever there are updated packages or vulnerable
   packages.
 - It **tries** to be intelligent: in each directory specified in the
   `dependabot.yml` file it searches for and interprets the package
   meta-data files (such as `Cargo.toml`) and attempts to do the
   "right thing."
 - When there is not a `dependabot.yml` file, it searches through the
   whole filesystem to find directories that look like top level
   directories of packages.  Dependabot has been observed to process
   any file with the right name (e.g., `requirements.txt`) even if
   that file doesn't appear to be part of any reasonably well-formed
   package structure!
 - Naive usage of Dependabot is very easy: One button on the project
   github page enables it.  Adding a `dependabot.yml` file would also
   do the same.

It is important to note that Dependabot isn't doing a build of your
project, it's just calling some *scan tool* (or code) for the given
ecosystem.  It is clearly more heuristics-based and it should not be
solely relied on to generate accurate CVE warnings.

## Limitations

Some limitations of Dependabot
 - It doesn't necessarily invoke the associated "resolve tool": it can
   wrongly extract files from the *PCD* file.  If it finds a *resolver
   output file*, even if incorrect, it naively trusts it: as a result,
   it can miss intermediate package dependencies.

 - There is very little control of how Dependabot interprets package
   description files. You tell it the directory and the ecosystem and
   hope for the best.

     (So, Dependabot *is* useful for single language packages that
     follow standard/default package practices closely. In this case,
     it does provide a lot of bang for the buck.)

 - It can use up significant github resources; for instance, the
   Dependabot in the saw-script repository takes 10 minutes to run.

 - Dependabot automatically starts a Dependabot "run" when it detects
   changes in `dependabot.yml` or in any files it deems to be *PCD* or
   *resolver output files*: however the output can be quite confusing
   as
    - Alerts and PRs must be explicitly discarded as they don't go
      away when these project "meta" files are updated.
    - Some "historical" alerts seem impossible to be removed.  It it
      not clear whether this is intended behavior, or just a bug.

  - Dependabot does not support Haskell.  (Not surprising as Haskell tools for
    CVE scans are very beta quality.)

## Reverse Engineering Dependabot: Creating a Model

**TODO: relation to Dependency scan/checker?**

Although there appears to be a good amount of documentation ...
*TODO* on how to set up Dependabot, it is not at all clear how it ...

Also, we have provided a `.github/dependabot.yml` along with *...*

**TODO: update ^**

See file **TODO** for a play by play of how we have developed this model.

## Assessment of Dependabot

In light of above comments, we would recommend against Dependabot as a
reliable source for CVE warnings.  However, as noted above, Dependabot
could be useful as
 - a very low overhead CVE scan tool
 - for supported language ecosystems
 - for simple, single package repository.
