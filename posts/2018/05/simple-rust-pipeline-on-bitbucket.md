<!--
.. title: Simple Rust Pipeline on Bitbucket
.. slug: simple-rust-pipeline-on-bitbucket
.. date: 2018-05-04 22:54:44 UTC
.. tags: rust, CI
.. category: programming
.. type: text
-->

[Bitbucket](https://bitbucket.org/) offers a builtin execution pipeline (Docker based), here is a minimal configuration to have your Rust test running:

```yaml
# This is a sample build configuration for all languages.
# Check our guides at https://confluence.atlassian.com/x/VYk8Lw for more examples.
# Only use spaces to indent your .yml configuration.
# -----
# You can specify a custom docker image from Docker Hub as your build environment.
image: python:2.7

pipelines:
  default:
    - step:
        script:
          - curl https://sh.rustup.rs -sSf | sh -s -- --default-toolchain nightly -y
          - source $HOME/.cargo/env
          - cargo test
```
