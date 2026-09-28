---
title: SDK Development
sidebar_position: 2
description: Plan a Tenon language SDK contribution and find the implementation contracts in the code repository.
---

<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

    https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

# Develop a plugin SDK

A language SDK lets plugin authors focus on their external system. It handles communication with Tenon, startup and shutdown, and the results of sending or receiving records.

## Keep responsibilities clear

| Component | Responsibility |
| --- | --- |
| Process sampling module | Own-process CPU and RSS with the same sampling meaning across languages |
| SDK | Tenon communication, lifecycle, bounded submission, completion results, and publishing process observations through its private OpenTelemetry SDK |
| Plugin | External clients, business configuration, payload meaning, and external acknowledgement or replay |
| Scaffold and bundler | A usable starter project and an installable plugin bundle |

Use APIs that fit the language while preserving Tenon's behavior. Different callback or async styles must still agree on ordering, backpressure, completion, and failure outcomes.

## Approach the work

1. Read the shared contract and compare the existing Rust and Java implementations.
2. Establish the [IPC integration](/tenon/docs/development/contribute/ipc/) and an independently reusable `sdk/<language>/process-metrics` package. Keep OS sampling independent of OpenTelemetry and transport; the language SDK uses the sampler and owns metric publication.
3. Support Source, Sink, and combined plugins with a clear lifecycle for each. Implement request-driven CPU and RSS reporting through the SDK metrics protocol, independently of business lifecycle and Queues.
4. Verify behavior through generated plugins and a real Runner, including first CPU samples, missing observations, metrics reconnection, process restarts, failure and shutdown.

These requirements apply to every SDK. The currently available Java and Rust SDKs and scaffolds implement them; a Plugin author does not add instrumentation or install another collector.

## Implementation details

Use the code repository for protocol fields, callback rules, ownership, and compatibility requirements:

- [SDK implementation contract](https://github.com/apache/bifromq-tenon/blob/d1848739c16373f5a150258fccd4475c0ef67992/sdk/plugin-sdk-contract.md) — shared behavior across languages.
- [Rust SDK](https://github.com/apache/bifromq-tenon/blob/d1848739c16373f5a150258fccd4475c0ef67992/sdk/rust/plugin-sdk/README.md) — the Rust API and implementation entry point.
- [Java SDK](https://github.com/apache/bifromq-tenon/blob/d1848739c16373f5a150258fccd4475c0ef67992/sdk/java/README.md) — the Java API and implementation entry point.
- [Verification instructions](https://github.com/apache/bifromq-tenon/blob/d1848739c16373f5a150258fccd4475c0ef67992/CONTRIBUTING.md) — toolchains and checks for a contribution.
