# Central Workflows Repository

이 repository는 여러 프로젝트에서 재사용 가능한 GitHub Actions workflows를 제공합니다.

## 📋 제공하는 Workflows

### 1. checkout-code.yml

소스 코드를 checkout하는 workflow

**입력 파라미터:**

- `repository`: checkout할 repository (기본값: 현재 repo)
- `ref`: branch/tag/commit (기본값: github.ref)
- `submodules`: git submodule 지원 (기본값: false)
- `fetch-depth`: 가져올 commit 수 (기본값: 1)

### 2. compile-cpp.yml

C++ 프로젝트를 CMake로 빌드하는 workflow

**입력 파라미터:**

- `compiler`: 사용할 컴파일러 (기본값: gcc)
- `build-type`: 빌드 타입 (기본값: Release)
- `cmake-args`: 추가 CMake 인자 (선택)
- `working-directory`: 작업 디렉토리 (기본값: .)
- `output-path`: 빌드 결과 경로 (기본값: build)
- `env-vars`: 환경 변수 JSON (선택)

### 3. compile-java.yml

Java 프로젝트를 Maven/Gradle로 빌드하는 workflow

**입력 파라미터:**

- `java-version`: Java 버전 (기본값: 17)
- `build-tool`: maven 또는 gradle (기본값: maven)
- `build-command`: 사용자 정의 빌드 명령어 (선택)
- `working-directory`: 작업 디렉토리 (기본값: .)
- `env-vars`: 환경 변수 JSON (선택)

### 4. upload-artifact.yml

빌드 결과를 GitHub Artifacts 또는 Release에 업로드하는 workflow

**입력 파라미터:**

- `artifact-name`: 아티팩트 이름 (기본값: build-artifact)
- `artifact-path`: 업로드할 경로 (필수)
- `retention-days`: 보관 기간 (기본값: 90일)
- `upload-to-release`: Release 업로드 여부 (기본값: false)

### 5. reusable-test.yml

테스트를 실행하는 workflow (C++, Java, Python, Node 지원)

**입력 파라미터:**

- `test-command`: 테스트 실행 명령어 (필수)
- `language`: 프로그래밍 언어 (기본값: cpp)
- `coverage-enabled`: 커버리지 활성화 (기본값: false)
- `working-directory`: 작업 디렉토리 (기본값: .)

## 🚀 사용 방법

각 프로젝트의 `.github/workflows/ci.yml`에서 필요한 workflow를 호출하면 됩니다.

### C++ 프로젝트 예제

```yaml
name: CI

on: [push, pull_request]

jobs:
  build:
    uses: your-org/central-workflows/.github/workflows/compile-cpp.yml@main
    with:
      compiler: gcc
      build-type: Release
      cmake-args: "-DENABLE_TESTS=ON"
      env-vars: '{"CC":"gcc-11","CXX":"g++-11"}'
  
  upload:
    needs: build
    uses: your-org/central-workflows/.github/workflows/upload-artifact.yml@main
    with:
      artifact-name: cpp-binary
      artifact-path: build/bin/
```

### Java 프로젝트 예제

```yaml
name: CI

on: [push, pull_request]

jobs:
  build:
    uses: your-org/central-workflows/.github/workflows/compile-java.yml@main
    with:
      java-version: '17'
      build-tool: maven
      build-command: "mvn clean package"
  
  upload:
    needs: build
    uses: your-org/central-workflows/.github/workflows/upload-artifact.yml@main
    with:
      artifact-name: java-jar
      artifact-path: target/*.jar
```

## 📚 자세한 사용 가이드

더 많은 예제와 사용 방법은 [docs/usage.md](docs/usage.md)를 참조하세요.

## 🤝 기여

Workflow 개선 제안이나 버그 리포트는 이슈로 제출해 주세요.
