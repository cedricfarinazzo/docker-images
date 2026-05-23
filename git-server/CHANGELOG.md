# [git-server-v2.0.0](https://github.com/cedricfarinazzo/docker-images/compare/git-server-v1.0.1...git-server-v2.0.0) (2026-05-23)


* fix(security)!: pin tarball sha256s, multistage to drop build toolchain, defensive bootstrap quoting ([fe712b0](https://github.com/cedricfarinazzo/docker-images/commit/fe712b0d288a96e9ee77122f4821d58a636d6c5d))


### BREAKING CHANGES

* py-ta-lib no longer ships a build toolchain. Existing
`Dockerfile`s that `pip install ta-lib` on top of this image will fail
with missing gcc/g++. Add `RUN apt-get install -y build-essential`
(debian variant) or `RUN apk add build-base` (alpine variant) before
the pip install. See py-ta-lib/README.md downstream-usage section.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>

# [git-server-v1.0.1](https://github.com/cedricfarinazzo/docker-images/compare/git-server-v1.0.0...git-server-v1.0.1) (2026-05-23)


### Bug Fixes

* **git-server:** unbreak first-build (idempotent git user + ARG after FROM) ([f0a79b2](https://github.com/cedricfarinazzo/docker-images/commit/f0a79b2b3039d2eb71094e7f21be4fb6bef5525f))

# git-server-v1.0.0 (2026-05-23)


### Features

* **git-server:** scaffold gitolite SSH + nginx smart-HTTP image ([1bb93e3](https://github.com/cedricfarinazzo/docker-images/commit/1bb93e3a174a3d15e9d56a89ef2b7cd5182e61e6))
