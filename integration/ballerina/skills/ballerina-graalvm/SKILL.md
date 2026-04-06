---
name: ballerina-graalvm
description: Build and run Ballerina GraalVM native executables locally or in a Docker container. Use when the user wants native image compilation, faster startup, or smaller container images for Ballerina projects.
allowed-tools: Read Write Edit Bash Glob Grep
---

Help the user build a GraalVM native executable from their Ballerina project. Ask which approach they prefer if not specified: **local** or **container (Docker)**.

## Prerequisites

| Requirement | Local | Container |
|-------------|-------|-----------|
| Ballerina Swan Lake (latest) | Yes | Yes |
| GraalVM (Java 21 for Update 11+) | Yes | No (built inside container) |
| `GRAALVM_HOME` or `JAVA_HOME` set | Yes | No |
| Docker (8GB+ memory recommended) | No | Yes |

Install GraalVM locally via SDKMAN!: `sdk install java 21.0.2-graalce`

## Workflow A: Build Locally

1. Verify GraalVM is installed: `java -version` (should show GraalVM).
2. Build: `bal build --graalvm`
3. Run: `./target/bin/<project-name>`
4. If build warns about incompatible packages, run tests against native image: `bal test --graalvm`

## Workflow B: Build in Docker Container

1. Verify Docker is running and has 8GB+ memory allocated.
2. Build: `bal build --graalvm --cloud=docker`
3. This auto-generates a multi-stage Dockerfile (GraalVM build stage -> distroless runtime).
4. Run: `docker run -d -p <port>:<port> <project-name>:latest`

## GraalVM Build Options

Pass options via CLI or `Ballerina.toml`:

**CLI:**
```
bal build --graalvm --graalvm-build-options="-H:+StaticExecutableWithDynamicLibC"
```

**Ballerina.toml:**
```toml
[build-options]
graalvmBuildOptions = "--verbose -H:+StaticExecutableWithDynamicLibC"
```

## Marking Library Packages as GraalVM-Compatible

Add to `Ballerina.toml`:
```toml
[platform.java21]
graalvmCompatible = true
```

## Limitations

- Native image builds are resource-intensive (high memory + CPU, long build times).
- Code coverage and runtime debug are not supported with GraalVM native image testing.
- Apple M1 (darwin-aarch64) support is experimental.
- Windows requires Visual Studio with MSVC.

## Reference

Full docs: https://ballerina.io/learn/graalvm-executable-overview/
