FROM debian:trixie
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates build-essential clang cmake ninja-build pkg-config \
      git python3 python3-venv curl unzip zip zlib1g-dev libstdc++-14-dev \
    && rm -rf /var/lib/apt/lists/*

# Use a known WABT version: the tests require at least 1.0.37.
RUN git clone --depth 1 --recurse-submodules --shallow-submodules --branch 1.0.37 https://github.com/WebAssembly/wabt.git /opt/wabt-src \
    && cmake -S /opt/wabt-src -B /opt/wabt-build -G Ninja \
         -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/opt/wabt \
         -DBUILD_TESTS=OFF -DBUILD_LIBWASM=OFF \
    && cmake --build /opt/wabt-build --parallel 4 \
    && cmake --install /opt/wabt-build
ENV WABT_DIR=/opt/wabt/bin

# Scanner contract: the public Graal checkout is the build context, at /src.
COPY . /src
WORKDIR /src
# Match mx and the JVMCI JDK to this checkout, rather than using an installed
# GraalVM release to build/test a different version of the runtime.
RUN python3 -c 'import json, subprocess; v=json.load(open("common.json"))["mx_version"]; subprocess.run(["git", "clone", "--depth", "1", "--branch", v, "https://github.com/graalvm/mx.git", "/opt/mx"], check=True)'
ENV PATH=/opt/mx:/opt/wabt/bin:${PATH}
RUN mx --primary-suite-path /src/compiler -y fetch-jdk \
      --jdk-id labsjdk-ce-latest --to /opt/jdks --alias /opt/java
ENV JAVA_HOME=/opt/java
ENV PATH=/opt/java/bin:${PATH}

# Build sequentially: suites share build outputs. All dependencies and the test
# runner are fetched while network access is still available.
RUN mx -p /src/substratevm build --build-logs=silent
RUN mx -p /src/wasm --dynamicimports /compiler build --build-logs=silent
# Retain the source-matched Wasm isolate library for offline sandbox testing.
RUN mx -p /src/wasm --dynamicimports /substratevm,/compiler \
      --polyglot-isolates=wasm build --build-logs=silent --dependencies WASM_ISOLATE

# Compile and execute a native binary produced by this source-built builder.
# Keep the source, classes, and executable for offline experiments.
RUN mkdir -p /opt/graalvm-smoke \
    && printf '%s\n' 'public class HelloWorld { public static void main(String[] args) { System.out.println("Hello from Native Image"); } }' > /opt/graalvm-smoke/HelloWorld.java \
    && javac -d /opt/graalvm-smoke /opt/graalvm-smoke/HelloWorld.java \
    && mx -p /src/substratevm native-image --no-fallback -g \
         -cp /opt/graalvm-smoke -o /opt/graalvm-smoke/hello HelloWorld \
    && /opt/graalvm-smoke/hello

# Warm the unit-test dependencies/cache before the scanner loses network access.
# As in the scanner template, test failures are visible but do not abort setup.
RUN mx -p /src/wasm --dynamicimports /compiler unittest \
      -Dwasmtest.watToWasmExecutable=/opt/wabt/bin/wat2wasm WasmTestSuite \
    || echo 'WARNING: GraalWasm WasmTestSuite failed; inspect the build log'
WORKDIR /src
