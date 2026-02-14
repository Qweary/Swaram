# tools/ — Utility Scripts

## Purpose

This directory contains utility scripts for development workflows, testing automation, and operational tooling. Scripts here support the development process but are not part of the compiled Rust project.

## Current State

This directory may be empty or contain project-specific helper scripts. Check the directory contents for what's currently available.

## Guidelines for Adding Scripts

Name scripts descriptively: `run-music-test.sh`, `generate-test-images.py`, `benchmark-modem.sh`. Include a usage comment at the top of every script. Scripts that generate test data should write to a temporary directory, not assets/. Scripts that modify source files should run `cargo fmt` afterward.

Appropriate uses: test automation (running specific test scenarios with parameters), benchmarking (timing modem encode/decode, measuring pipeline throughput), image generation (creating test images with specific feature profiles), deployment helpers (installation setup, FluidSynth configuration).

Not appropriate: anything that should be a Rust integration test (put those in tests/), build system modifications (use Cargo.toml), or one-off experiments (use a scratch directory).

## Running Scripts

Most scripts assume they're run from the repository root. Use `./tools/script-name.sh` or `python3 tools/script-name.py`. Scripts that need Rust binaries should run `cargo build --release` first or check for the binary's existence.
