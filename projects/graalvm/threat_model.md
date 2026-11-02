# GraalVM threat model

## Scope and priorities

Scan the public `oracle/graal` repository, prioritizing:

1. Native Image: `substratevm/` and its use of `compiler/`, `sdk/`, and shared runtime infrastructure. Cover the builder and, especially, the security of the resulting native executables and shared libraries.
2. GraalWasm: `wasm/` as a sandboxable Truffle language, including the `truffle/`, `sdk/`, compiler, and Native Image paths that enforce its execution boundary.

## Trust assumptions and attacker control

The operator, embedding application, Polyglot API configuration, command-line options, environment, native libraries, build toolchain, and operating system are trusted. Intentional host exports and custom service providers define the allowed interface. A deliberately overprivileged export, unsafe custom filesystem, or custom host type mapping is not a runtime escape.

For Native Image AOT compilation, application bytecode, build configuration, Features, substitutions, reachability metadata, and build-time initialization code are generally trusted. Building an attacker-supplied application is not a sandboxed operation: build-time code may execute with builder privileges. Prioritize miscompilation of trusted applications that becomes exploitable when the resulting executable processes attacker-controlled runtime input. Compiler failures on hostile build inputs alone do not establish a security issue.

For GraalWasm, the attacker can supply module bytes, imports/exports, function bodies, memory/table contents, input data, and execution patterns. They can shape JIT profiles, trigger compilation/deoptimization, and exercise supported concurrency and context lifecycles. They do not start with arbitrary native-memory access, privileged host access, or the ability to change the embedder's policy.

Use a production configuration that explicitly supports the claimed boundary. The default `SandboxPolicy.TRUSTED` is not a sandbox. `CONSTRAINED` regulates resource access for trusted programs; do not infer full hostile-code containment or availability protection from it. For stronger policies, establish that the build supports them and enable their required limits and mitigations. Experimental features are excluded.

## Native Image security objectives and attack surfaces

The compiler must preserve Java and Truffle semantic safety and memory-model guarantees through graph construction, optimization, lowering, register allocation, and code emission. Prioritize bounds/null/type checks, integer width/sign handling, aliasing, speculative assumptions, deoptimization and frame reconstruction, atomics, monitors, volatile accesses, fences, and safe publication. A compiler defect must have a demonstrated security consequence, beyond a wrong answer or performance regression.

The runtime and generated artifact must preserve memory safety, code/metadata integrity, and the guarantees of supported runtime interfaces.
Code generation must preserve the isolation and hardening actually configured, including runtime-compilation mitigations where applicable. Do not assume all isolates provide hostile-code isolation or that architecture-specific mitigations are universal.
Runtime diagnostics or metadata handling that exposes secrets or permits attacker-driven corruption in production. Debugger privileges and intentionally enabled privileged diagnostic endpoints are not attacker capabilities by default.

Ordinary safe Java code must not gain unchecked memory access through compiler/runtime errors. An existing arbitrary native write, intentional Unsafe use, or a buggy external native library cannot be assumed as the starting primitive.

## GraalWasm and Truffle security objectives

For a supported sandbox configuration, guest code must not expand the explicitly configured host interface, bypass mediated resource access, or compromise the runtime, embedding application, or another trust domain.

## How to exercise it

The Dockerfile installs a checkout-matched JVMCI JDK at `/opt/java`, `mx` at `/opt/mx`, and WABT at `/opt/wabt`. The public source and all build outputs remain under `/src`; downloaded build/test dependencies are retained for offline use. It builds Native Image and compiler-enabled GraalWasm, leaves a source-built native smoke executable at `/opt/graalvm-smoke/hello`, and runs `WasmTestSuite`. It also builds the `WASM_ISOLATE` distribution for offline sandbox testing. The optional C/Emscripten corpus is not built.

Useful commands inside the scanner image:

```sh
mx -p /src/substratevm build --build-logs=silent
mx -p /src/substratevm native-image --no-fallback -g -cp /opt/graalvm-smoke -o /opt/graalvm-smoke/hello HelloWorld
/opt/graalvm-smoke/hello
mx -p /src/wasm --dynamicimports /compiler build --build-logs=silent
mx -p /src/wasm --dynamicimports /compiler unittest -Dwasmtest.watToWasmExecutable=/opt/wabt/bin/wat2wasm WasmTestSuite
mx -p /src/wasm --dynamicimports /compiler wasm /path/to/reproducer.wasm
```

Use `wat2wasm` for small guest modules. Existing tests in `wasm/src/org.graalvm.wasm.test/`, `substratevm/src/com.oracle.svm.test/`, compiler tests, and Truffle Polyglot tests provide harnesses. For sandbox reports, use a Java Polyglot API embedding harness with explicit policy, allowed language `wasm`, redirected streams, minimal host exports, and all required limits. A default command-line launcher run alone does not reproduce a sandbox escape. The matching isolate resources are built with `mx -p /src/wasm --dynamicimports /substratevm,/compiler --polyglot-isolates=wasm build --dependencies WASM_ISOLATE`. Use the corresponding mx classpath/distribution outputs in the harness and verify it works offline; do not silently substitute another build.

## Reporting and severity

Include the affected revision, build, architecture, policy/options, attacker-controlled entry point, violated guarantee, and a minimal reproducer. Distinguish guest traps/exceptions and context cancellation from host corruption or host-wide denial of service. Identify whether impact occurs while building a trusted application, running a native executable, or executing guest Wasm.

Rate by demonstrated impact and reachability. Arbitrary host code execution, usable host memory corruption, or a sandbox escape with broad host compromise generally warrants high or critical severity. Unauthorized cross-domain disclosure or mutation is security-relevant even without code execution. Host-wide denial of service from bounded guest input or bypass of an explicitly promised resource limit is generally medium, adjusted for deployment impact. A guest trap, correctly cancelled context, unsupported configuration, pure miscompilation without security impact, or hypothetical exploitation requiring a pre-existing arbitrary native write is insufficient.

Exclude pure performance issues, intentionally insecure debug modes, privileged operator actions, and attacks requiring an unrelated CPU/kernel defect (including speculation beyond a fault/Meltdown). Do not report hardening features unavailable in this build as vulnerabilities. Do not include private assessment findings or internal documents in reports or patches.

## Public reference contracts

Consult the scanned checkout's documentation and implementation first:

- `docs/reference-manual/native-image/contribute/DevelopingNativeImage.md`
- `docs/security/polyglot-sandbox.md`
- `wasm/docs/contributor/Building.md` and `TestsAndBenchmarks.md`
