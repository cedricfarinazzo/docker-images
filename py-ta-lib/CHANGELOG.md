# [py-ta-lib-v3.0.0](https://github.com/cedricfarinazzo/docker-images/compare/py-ta-lib-v2.0.0...py-ta-lib-v3.0.0) (2026-05-23)


* fix(security)!: pin tarball sha256s, multistage to drop build toolchain, defensive bootstrap quoting ([fe712b0](https://github.com/cedricfarinazzo/docker-images/commit/fe712b0d288a96e9ee77122f4821d58a636d6c5d))


### BREAKING CHANGES

* py-ta-lib no longer ships a build toolchain. Existing
`Dockerfile`s that `pip install ta-lib` on top of this image will fail
with missing gcc/g++. Add `RUN apt-get install -y build-essential`
(debian variant) or `RUN apk add build-base` (alpine variant) before
the pip install. See py-ta-lib/README.md downstream-usage section.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>

# [py-ta-lib-v2.0.0](https://github.com/cedricfarinazzo/docker-images/compare/py-ta-lib-v1.1.0...py-ta-lib-v2.0.0) (2026-05-23)


* feat(py-ta-lib)!: ship only the TA-Lib C library, no Python binding ([dc892b6](https://github.com/cedricfarinazzo/docker-images/commit/dc892b6bbe6fc1a7d08246b385abde6df04a2714))


### BREAKING CHANGES

* `ta-lib` is no longer importable out of the box.
Existing `Dockerfile`s and `docker run` invocations that assumed
`import talib` works on the bare image will fail. Migration:
add `RUN pip install --no-cache-dir ta-lib==<version>` to your
downstream Dockerfile (the build toolchain needed to compile the
C extension is still in the image).

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>

# [py-ta-lib-v1.1.0](https://github.com/cedricfarinazzo/docker-images/compare/py-ta-lib-v1.0.0...py-ta-lib-v1.1.0) (2026-05-23)


### Features

* **py-ta-lib:** set Python container hygiene env vars by default ([6f85557](https://github.com/cedricfarinazzo/docker-images/commit/6f855572310ebee4f5cb1146c326503bcd452c64))

# py-ta-lib-v1.0.0 (2026-05-23)


### Bug Fixes

* **py-ta-lib:** drop semantic-release-monorepo extends ([c8ba774](https://github.com/cedricfarinazzo/docker-images/commit/c8ba7748333ceee593f2666fa3bff72cbe59ced6))
* **py-ta-lib:** include commit-analyzer + release-notes-generator in plugins ([d2fba31](https://github.com/cedricfarinazzo/docker-images/commit/d2fba314579de49fd1653a1767563cad17bb2391))
* **py-ta-lib:** inline release plugins + explicit releaseRules ([7991f10](https://github.com/cedricfarinazzo/docker-images/commit/7991f10a284693302d787f082d7cddc8f9b40dc2))
* **py-ta-lib:** reorder extends so semantic-release-monorepo wins ([87e4c5d](https://github.com/cedricfarinazzo/docker-images/commit/87e4c5d8fed72fcfd3e1dd39f66093bec4ca065d))
* **py-ta-lib:** use semantic-release-monorepo + switch CI to bun ([324471d](https://github.com/cedricfarinazzo/docker-images/commit/324471d85dff9924f3fe6de4e85fea6707c43559))


### Features

* **py-ta-lib:** scaffold monorepo with py-ta-lib image ([934471c](https://github.com/cedricfarinazzo/docker-images/commit/934471c5ed89ef55aedf71f1dd9da26163073d0a))

# py-ta-lib 1.0.0 (2026-05-23)


### Bug Fixes

* **py-ta-lib:** drop semantic-release-monorepo extends ([c8ba774](https://github.com/cedricfarinazzo/docker-images/commit/c8ba7748333ceee593f2666fa3bff72cbe59ced6))
* **py-ta-lib:** inline release plugins + explicit releaseRules ([7991f10](https://github.com/cedricfarinazzo/docker-images/commit/7991f10a284693302d787f082d7cddc8f9b40dc2))
* **py-ta-lib:** reorder extends so semantic-release-monorepo wins ([87e4c5d](https://github.com/cedricfarinazzo/docker-images/commit/87e4c5d8fed72fcfd3e1dd39f66093bec4ca065d))


### Features

* **py-ta-lib:** scaffold monorepo with py-ta-lib image ([934471c](https://github.com/cedricfarinazzo/docker-images/commit/934471c5ed89ef55aedf71f1dd9da26163073d0a))
