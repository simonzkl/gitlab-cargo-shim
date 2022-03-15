# gitlab-cargo-shim

To use `github-cargo-shim` at the moment you will have to apply [this patch][patch] ([MR][MR])
to your GitLab instance.

Say goodbye to your Git dependencies, `gitlab-cargo-shim` is an SSH server
that serves crates just like a standard Cargo registry but from a
[GitLab package registry][gitlab-package-registry], allowing you to use
your private dependencies like any other dependency. No more `git push --force`s
breaking your builds & proper versioning in one simple little binary.

Access controls work just like they do in GitLab, builds are scoped to
users - if they don't have permission to the dependency they can't build
it, it's that simple.

Users are identified by their SSH keys when connecting to the server and
will be authenticated to the GitLab API via an [impersonation token][imp-token],
builds will insert their token as a username to the SSH server and the
shim will use that to call the GitLab API.

To publish simply run `cargo package` and push the resulting `.crate` file
to the GitLab package repository with a semver-compatible version string, to
consume the package simply configure your `.cargo/config.toml` and `Cargo.toml`
accordingly.

```toml
# .cargo/config.toml
[registries]
my-gitlab-group = { index = "ssh://gitlab-cargo-shim.local/my-gitlab-group" }

# Cargo.toml
[dependencies]
my-crate = { version = "0.1", registry = "my-gitlab-group" }
```

In your CI build, setup a `before_script` step to replace the connection string
with one containing the CI token:

```yaml
# .gitlab-ci.yml
before_script:
  - sed -i "s/(gitlab-cargo-shim.local)/gitlab-ci-token:$GITLAB-CI-TOKEN@\1/" .cargo/config.toml
```

(or add the corresponding [environment variable][envvar])

To release your package from CI, add a new pipeline step:

```yaml
stage:
  - release-crate

release-crate:
  stage: release-crate
  before_script:
    - cargo install cargo-get
    - export CRATE_NAME=$(cargo get name) CRATE_VERSION=$(cargo get version)
    - export CRATE_FILE=$(CRATE_NAME)-$(CRATE_FILE).crate
  script:
    - cargo package
    - cargo metadata > metadata.json
    - curl --header "JOB-TOKEN: $CI_JOB_TOKEN" --upload-file target/package/${CRATE_FILE} "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/generic/${NAME}/${VERSION}/${CRATE_FILE}"
    - curl --header "JOB-TOKEN: $CI_JOB_TOKEN" --upload-file metadata.json "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/generic/${NAME}/${VERSION}/metadata.json"
```

It's that easy. Go forth and enjoy your newfound quality of life improvements,
Rustacean.

[gitlab-package-registry]: https://docs.gitlab.com/ee/user/packages/package_registry/index.html
[imp-token]: https://docs.gitlab.com/ee/api/index.html#impersonation-tokens
[envvar]: https://doc.rust-lang.org/cargo/reference/registries.html#using-an-alternate-registry
[patch]: https://gitlab.com/gitlab-org/gitlab/-/merge_requests/82663.patch
[MR]: https://gitlab.com/gitlab-org/gitlab/-/merge_requests/82663
