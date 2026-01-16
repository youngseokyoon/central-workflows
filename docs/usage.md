# GitHub Reusable Workflows 사용 가이드

이 문서는 중앙 workflow repository의 각 workflow를 사용하는 방법을 상세히 설명합니다.

## 📖 목차

1. [Reusable Workflows 개요](#reusable-workflows-개요)
2. [각 Workflow 상세 가이드](#각-workflow-상세-가이드)
3. [실전 예제](#실전-예제)
4. [GitHub에 배포하기](#github에-배포하기)
5. [트러블슈팅](#트러블슈팅)

## Reusable Workflows 개요

### 기본 사용 패턴

```yaml
jobs:
  job-name:
    uses: owner/repo/.github/workflows/workflow-name.yml@ref
    with:
      input-name: value
    secrets: inherit  # organization 내부에서만
```

### 버전 관리

- `@main`: 최신 버전 (개발 중)
- `@v1`: 안정 버전 (권장)
- `@abc123`: 특정 commit

## 각 Workflow 상세 가이드

### 1. checkout-code.yml

소스 코드를 checkout합니다. 대부분의 경우 GitHub Actions의 기본 `actions/checkout@v4`를 직접 사용하는 것이 더 간단하지만, 조직 전체의 checkout 표준을 강제하고 싶을 때 유용합니다.

**사용 예시:**

```yaml
jobs:
  checkout:
    uses: org/central-workflows/.github/workflows/checkout-code.yml@main
    with:
      submodules: 'recursive'  # Git submodule 포함
      fetch-depth: 0           # 전체 히스토리
```

**입력 파라미터:**

| 파라미터 | 타입 | 기본값 | 설명 |
|----------|------|--------|------|
| `repository` | string | (현재 repo) | Checkout할 repository |
| `ref` | string | (현재 ref) | Branch/tag/commit |
| `submodules` | string | false | Submodule checkout 여부 |
| `fetch-depth` | number | 1 | 가져올 commit 수 |

### 2. compile-cpp.yml

C++ 프로젝트를 CMake로 빌드합니다.

**기본 사용:**

```yaml
jobs:
  build:
    uses: org/central-workflows/.github/workflows/compile-cpp.yml@main
    with:
      build-type: Release
```

**고급 사용 (환경 변수 커스터마이징):**

```yaml
jobs:
  build:
    uses: org/central-workflows/.github/workflows/compile-cpp.yml@main
    with:
      compiler: gcc
      build-type: Release
      cmake-args: "-DENABLE_TESTS=ON -DBUILD_SHARED_LIBS=OFF"
      working-directory: cpp-module
      output-path: build
      env-vars: |
        {
          "CC": "gcc-12",
          "CXX": "g++-12",
          "CXXFLAGS": "-O3 -march=native"
        }
```

**입력 파라미터:**

| 파라미터 | 타입 | 기본값 | 설명 |
|----------|------|--------|------|
| `compiler` | string | gcc | C++ 컴파일러 (gcc, g++, clang) |
| `build-type` | string | Release | 빌드 타입 (Debug, Release, etc.) |
| `cmake-args` | string | "" | 추가 CMake 인자 |
| `working-directory` | string | . | 작업 디렉토리 |
| `output-path` | string | build | 빌드 결과 경로 |
| `env-vars` | string | {} | 환경 변수 (JSON 형식) |

### 3. compile-java.yml

Java 프로젝트를 Maven 또는 Gradle로 빌드합니다.

**Maven 사용:**

```yaml
jobs:
  build:
    uses: org/central-workflows/.github/workflows/compile-java.yml@main
    with:
      java-version: '17'
      build-tool: maven
```

**Gradle 사용:**

```yaml
jobs:
  build:
    uses: org/central-workflows/.github/workflows/compile-java.yml@main
    with:
      java-version: '21'
      build-tool: gradle
      build-command: "./gradlew build -x test"
```

**Maven 메모리 옵션:**

```yaml
jobs:
  build:
    uses: org/central-workflows/.github/workflows/compile-java.yml@main
    with:
      java-version: '17'
      build-tool: maven
      env-vars: '{"MAVEN_OPTS":"-Xmx4096m -XX:+UseG1GC"}'
```

**입력 파라미터:**

| 파라미터 | 타입 | 기본값 | 설명 |
|----------|------|--------|------|
| `java-version` | string | 17 | Java 버전 (8, 11, 17, 21) |
| `build-tool` | string | maven | 빌드 도구 (maven, gradle) |
| `build-command` | string | "" | 사용자 정의 명령어 (선택) |
| `working-directory` | string | . | 작업 디렉토리 |
| `env-vars` | string | {} | 환경 변수 (JSON 형식) |

### 4. upload-artifact.yml

빌드 결과를 GitHub Artifacts 또는 Release에 업로드합니다.

**Artifacts에 업로드:**

```yaml
jobs:
  upload:
    uses: org/central-workflows/.github/workflows/upload-artifact.yml@main
    with:
      artifact-name: my-app-binary
      artifact-path: build/bin/
      retention-days: 30
```

**Release에 업로드 (태그 push 시):**

```yaml
jobs:
  release:
    uses: org/central-workflows/.github/workflows/upload-artifact.yml@main
    with:
      artifact-name: release-binaries
      artifact-path: |
        build/bin/*
        dist/*.tar.gz
      upload-to-release: true
```

**입력 파라미터:**

| 파라미터 | 타입 | 기본값 | 설명 |
|----------|------|--------|------|
| `artifact-name` | string | build-artifact | 아티팩트 이름 |
| `artifact-path` | string | (필수) | 업로드할 파일/디렉토리 경로 |
| `retention-days` | number | 90 | 보관 기간 (일) |
| `upload-to-release` | boolean | false | GitHub Release에 업로드 여부 |
| `if-no-files-found` | string | warn | 파일 없을 때 동작 |

### 5. reusable-test.yml

테스트를 실행합니다. 여러 언어를 지원합니다.

**C++ 테스트:**

```yaml
jobs:
  test:
    uses: org/central-workflows/.github/workflows/reusable-test.yml@main
    with:
      test-command: "ctest --test-dir build --output-on-failure"
      language: cpp
      coverage-enabled: true
```

**Java 테스트:**

```yaml
jobs:
  test:
    uses: org/central-workflows/.github/workflows/reusable-test.yml@main
    with:
      test-command: "mvn test"
      language: java
      coverage-enabled: true
```

**입력 파라미터:**

| 파라미터 | 타입 | 기본값 | 설명 |
|----------|------|--------|------|
| `test-command` | string | (필수) | 테스트 실행 명령어 |
| `language` | string | cpp | 언어 (cpp, java, python, node) |
| `coverage-enabled` | boolean | false | 커버리지 활성화 여부 |
| `working-directory` | string | . | 작업 디렉토리 |

## 실전 예제

### 예제 1: C++ 단일 프로젝트

```yaml
name: CI

on: [push, pull_request]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build
        uses: org/central-workflows/.github/workflows/compile-cpp.yml@main
        with:
          compiler: gcc
          build-type: Release
          cmake-args: "-DENABLE_TESTS=ON"
      
      - name: Test
        uses: org/central-workflows/.github/workflows/reusable-test.yml@main
        with:
          test-command: "ctest --test-dir build"
          language: cpp
      
      - name: Upload
        uses: org/central-workflows/.github/workflows/upload-artifact.yml@main
        with:
          artifact-name: release-binary
          artifact-path: build/bin/
```

### 예제 2: Java with Matrix Strategy

```yaml
name: CI

on: [push, pull_request]

jobs:
  test-matrix:
    strategy:
      matrix:
        java: ['11', '17', '21']
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: org/central-workflows/.github/workflows/compile-java.yml@main
        with:
          java-version: ${{ matrix.java }}
          build-tool: maven
```

### 예제 3: 복합 프로젝트 (병렬 빌드)

```yaml
name: CI

on: [push, pull_request]

jobs:
  build-cpp:
    uses: org/central-workflows/.github/workflows/compile-cpp.yml@main
    with:
      working-directory: cpp-module
      build-type: Release
  
  build-java:
    uses: org/central-workflows/.github/workflows/compile-java.yml@main
    with:
      working-directory: java-module
      java-version: '17'
  
  upload-all:
    needs: [build-cpp, build-java]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
      - uses: org/central-workflows/.github/workflows/upload-artifact.yml@main
        with:
          artifact-name: combined-build
          artifact-path: ./
```

## GitHub에 배포하기

### 1. Central Workflows Repository 생성

```bash
# 1. GitHub에서 새 repository 생성 (예: your-org/central-workflows)

# 2. 로컬 central-workflows 디렉토리를 push
cd central-workflows
git init
git add .
git commit -m "Initial commit: Reusable workflows"
git remote add origin https://github.com/your-org/central-workflows.git
git push -u origin main

# 3. 버전 태그 생성 (optional but recommended)
git tag v1.0.0
git push origin v1.0.0
```

### 2. Consumer Repository 설정

각 프로젝트에서:

```yaml
# .github/workflows/ci.yml
uses: your-org/central-workflows/.github/workflows/compile-cpp.yml@v1.0.0
```

### 3. 권한 설정

- **Public repository**: 아무 설정 불필요
- **Private repository**:
  - Organization 내부: `Settings > Actions > General > Workflow permissions`
  - External: Personal Access Token 필요

## Composite Action vs Reusable Workflow

| 용도 | 권장 |
|------|------|
| 빌드 환경 설정 | Composite Action |
| 실제 빌드/테스트 | Reusable Workflow |
| 여러 step 묶기 | Composite Action |
| 여러 job 구성 | Reusable Workflow |
| 로컬 테스트 (act) | Composite Action |

## 트러블슈팅

### 문제: Workflow를 찾을 수 없음

```
Error: Unable to resolve action `org/repo/.github/workflows/xxx.yml@main`
```

**해결방법:**

1. Repository 이름과 경로 확인
2. Branch/tag 확인
3. Private repo의 경우 권한 확인

### 문제: Secrets를 전달할 수 없음

```yaml
# ❌ 작동 안 함 (cross-organization)
secrets: inherit

# ✅ 명시적으로 전달
secrets:
  TOKEN: ${{ secrets.MY_TOKEN }}
```

### 문제: 환경 변수 JSON 파싱 오류

```yaml
# ❌ 작동 안 함 (잘못된 JSON)
env-vars: {CC: gcc}

# ✅ 올바른 JSON
env-vars: '{"CC": "gcc", "CXX": "g++"}'
```

### 문제: act에서 reusable workflow 테스트 실패

Reusable workflow는 act에서 제한적으로 지원됩니다. 대신:

1. Workflow 로직을 consumer repository의 inline workflow로 복사
2. 또는 composite action 사용
3. 또는 GitHub에 push하여 테스트

## 추가 리소스

- [GitHub Actions 공식 문서](https://docs.github.com/en/actions)
- [Reusing workflows 가이드](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [act 툴 문서](https://nektosact.com)
