# Changelog

All notable changes to this project will be documented in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## v0.1.4 — 2026-09-13

- musl→glibc loader fix (Lustre client requires glibc)
- nsenter probe for `lfs --version` (replaces `/host/*` env+mount hack)
- DaemonSet no longer requires `/host/*` mounts or `HOST_*` envs
