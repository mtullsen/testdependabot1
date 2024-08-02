# Rust CVE Scanning with `cargo audit`

First, install the `cargo-audit` tool
```
   cargo install cargo-audit
```

To run it on our project, do
```
   cd rust-project/
   cargo update    # create/update the Cargo.lock.
   cargo audit
```
and you should then see
```
    Fetching advisory database from `https://github.com/RustSec/advisory-db.git`
      Loaded 645 security advisories (from /Users/tullsen/.cargo/advisory-db)
    Updating crates.io index
    Scanning Cargo.lock for vulnerabilities (7 crate dependencies)
Crate:     time
Version:   0.1.45
Title:     Potential segfault in the time crate
Date:      2020-11-18
ID:        RUSTSEC-2020-0071
URL:       https://rustsec.org/advisories/RUSTSEC-2020-0071
Severity:  6.2 (medium)
Solution:  Upgrade to >=0.2.23
Dependency tree:
time 0.1.45
└── rust-project 0.1.0

error: 1 vulnerability found!
```
The command returns an error code to indicate a CVE vulnerability has been found.
```
rust-project $ echo $?
1
```

See the (commented) `Cargo.toml` file: you can comment out the `time`
package and re-run `cargo audit` to see that it now determines there
are no known CVE vulnerabilities.

A few things to note here
 - `cargo audit` will call `cargo update` so you can dispense
    with the latter if you wish.
 - `cargo update` follows all crate dependencies and creates/updates
    the Cargo.toml, thus we find the indirect dependencies which may
    have CVE vulnerabilities.
 - when `cargo update` updates the `Cargo.lock` file, it can override
   crate versions specified in the `Cargo.toml` file as part of
   "dependency resolution".  This might be surprising, this is why
   our audit tool should use the `Cargo.lock` file.