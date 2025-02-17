# cargo-vulnerabilities

## Overview
This document contains a compiled list of notes on some potential threats to the security of rust's package management system cargo. These threats could be considered to fall under two distinct categories:
1. How can we verify the authenticity of the packages we download through cargo?
2. How can we determine whether the author(s) of a package can be trusted, i.e. how can we be sure we haven't downloaded malicious code?

We will briefly cover security concerns in both of these categories, but first it is necessary to give a refresher on the cargo ecosystem and how crates travel from the author to the user.

## Cargo Ecosystem
The flowchart below illustrates the process that connects the author of a Rust crate to the eventual end user:
<img src="/Images/drawing.svg" width="100%" height="144">

When cargo goes to fetch a crate, the default source that it will seach is the official cargo package registry, known as [crates.io](https://crates.io/). All crates available on crates.io must be uploaded by an author over an https connection, which also requires a special API token generated by linking a Github account to the author's crates.io account. The crate's metadata for all of its versions is maintained in crates.io's index, but the code itself is stored on a separate database. Thus, when a user downloads a crate through crates.io, first an https request is made to crates.io, then crates.io searches its index (hosted on Github) for the appropriate package. Once the package has been identified, it is fetched from the database and sent back to the user over https. Although crates.io is the default source of all packages that cargo downloads, cargo supports downloading code from alternate registries or from git repos, which must be specified explicitly. When fetching crates from an alternate source, a variety of additional safe and unsafe protocols like https may be used.

The diagram below, which is originally from [this insightful blog post on creating a custom mirror](https://www.integer32.com/2016/10/08/bare-minimum-crates-io-mirror-plus-one.html), illustrates the download process clearly:
<img src="https://www.integer32.com/images/posts/bare-minimum-mirror-plus-one/cratesio-diagram.png" width="100%" alt="Diagram explaining cargo download process">

For more information on how cargo registries work, see the [cargo manual](https://doc.rust-lang.org/cargo/reference/registries.html).

## Verifying Authenticity of Packages
A number of questions arise when considering the authenticity of packages downloaded by cargo:

### Could someone inject malicious code en route to the user?
Since packages downloaded through crates.io occur over a secure https connection, it is unlikely that a man-in-the-middle injection could occur. However, if downloading a crate from an alternative source over plain http (or downloading a crate that depends upon a crate that must be fetched over plain http) this is certainly a concern.

### Could someone create fake mirror of crates.io?
Once again, connections between the end user and crates.io occur over https, and therefore require SSL certificates signed by the real crates.io. Furthermore, at the moment the only mirrors of crates.io that can be created feature read-only copies of the crates index and cannot directly access the crate database, making it impossible for a mirror to create a false index. This could change if writeable indexes are ever introduced for mirrors. 

### Could the database be compromised?
As cargo does not use any algorithm to verify that a crate it sends to the end user is identical to the code that was originally uploaded by the crate author, it would be impossible to detect whether malicious code with injected into the database entry for a crate.

### Could someone hijack a project?
The `cargo owner` subcommand allows the creators of a package to add and remove crate owners, who are allowed to upload new versions of a package to crates.io. Since owners can be added or removed by any other owner, it is conceivable that a malicious agent could either become added as an owner or simply steal the API token from another owner, remove all other owners, and begin to upload malicious code into a new version of the crate. The agent would then control the crate forever.

### Potential solutions
An [ongoing issue on the crates.io Github repo](https://github.com/rust-lang/crates.io/issues/75) seeks to implement *TUF*, a security protocol that allows for crates to contain signatures from the author that can be used to verify their authenticity. However, no major progress has been made on the issue since 2014.

## Trusting a Package
While there are ways a package could become corrupted between the author and the user, another issue in ensuring safe code is verifying whether all the crates the code depends upon were not written by a malicious author. Although this process has a theoretical solution of "reading through all the source code," the sheer amount of code one would have to review for even a simple project makes this solution unfeasible. Thankfully, one solution exists in the form of [cargo-crev](https://github.com/crev-dev/cargo-crev), a tool that allows for decentralized authentication of crates among a network of trusted reviewers. Under this system, `crev` users can review a crate manually and publish that review. Then, that review becomes accessible to any other users in the trusted network. `crev` then provides a cargo subcommand that allows a user to audit all of the dependencies of the project they are currently working on, seeing which have been trusted, which have been flagged, and which have not been reviewed, along with helpful statistics such as the number of unsafe code blocks a depended on crate uses.
