# 통합 테스트 워크플로우 가이드 (Integration Test Workflow Guide)

이 문서는 `actions/github-script`를 사용하여 리팩토링된 통합 테스트 워크플로우의 구조, 설정 방법, 그리고 향후 확장 가능한 시나리오를 설명합니다.

## 1. 아키텍처 개요

이 통합 테스트 시스템은 **중앙 워크플로우(Central Workflows)**와 **컨슈머 리포지토리(Consumer Repositories)** 간의 연동을 검증합니다.

* **Central Workflow (`integration-test.yml`)**: 테스트의 오케스트레이터(Orchestrator) 역할을 수행합니다.
* **Consumer Repos**: 실제 빌드 및 테스트가 수행되는 프로젝트들입니다 (예: `sample-cpp-repo`, `sample-java-repo`).

### 동작 방식 (Workflow Flow)

1. **Trigger (트리거)**: GitHub API를 사용하여 컨슈머 리포지토리의 `ci.yml` 워크플로우를 실행합니다.
2. **Monitor (모니터링)**: 주기적으로(15초 간격) API를 호출하여 해당 워크플로우의 상태(`status`)를 확인합니다.
3. **Verify (검증)**: 워크플로우가 `completed` 상태가 되면, 실행 결과(`conclusion`)가 `success`인지 확인합니다.
4. **Report (보고)**: 모든 매트릭스 실행이 완료되면, PR에 댓글로 결과를 요약하여 보고합니다.

## 2. 구현 상세 (`gh` CLI 제거)

기존의 `gh` CLI 의존성을 제거하고, GitHub Actions의 기본 환경인 Node.js와 `actions/github-script`를 사용하여 구현되었습니다. 이를 통해 별도의 도구 설치 없이 프로덕션 환경에서 안정적으로 동작합니다.

### 주요 코드 로직

```javascript
// 1. 워크플로우 트리거
await github.rest.actions.createWorkflowDispatch({ owner, repo, workflow_id, ref });

// 2. 실행 ID 조회 (Polling)
// 트리거 직후 생성된 최신 런을 찾을 때까지 잠시 대기

// 3. 상태 감시 (Loop)
while (true) {
    const run = await github.rest.actions.getWorkflowRun({ ... });
    if (run.data.status === 'completed') {
        // 결과 확인 및 종료
    }
}
```

## 3. 설정 방법 (Setup)

이 워크플로우를 실행하기 위해서는 적절한 권한을 가진 PAT(Personal Access Token)가 필요합니다.

1. **토큰 생성**: `repo` 스코프가 있는 PAT를 생성합니다.
2. **Secrets 등록**: `central-workflows` 리포지토리의 Secrets에 `CI_TOKEN`이라는 이름으로 저장합니다.
    * 경로: Settings > Secrets and variables > Actions > Repository secrets

## 4. 추가 시나리오 제안 (Recommended Scenarios)

현재 구현은 "실행 및 성공 여부"만 확인하고 있습니다. 더 견고한 통합 테스트를 위해 다음 시나리오들을 추가하는 것을 권장합니다.

### A. 아티팩트 검증 (Artifact Verification)

컨슈머 리포지토리의 CI가 성공했다면, 실제로 예상한 산출물(Binary, JAR 등)이 생성되었는지 검증해야 합니다.

* **방법**: `github.rest.actions.listWorkflowRunArtifacts` API를 사용하여 업로드된 아티팩트 목록을 확인하고, 필요하다면 다운로드하여 내용을 검사합니다.

### B. 네거티브 테스트 (Negative Testing)

통합 테스트가 "실패"를 정확하게 감지하는지 확인하는 테스트입니다.

* **방법**: 의도적으로 빌드가 실패하는 브랜치나 입력을 주입하여, `integration-test.yml`이 이를 정확히 `failed`로 보고하는지 검증합니다.

### C. 배포 트리거 연동 (Continuous Deployment)

통합 테스트가 모든 컨슈머 리포지토리에 대해 성공할 경우, 스테이징(Staging) 환경으로의 배포를 자동으로 트리거합니다.

* **방법**: `integration-summary` job의 `needs` 결과가 성공일 경우, `repository_dispatch` 이벤트를 발생시켜 배포 워크플로우를 시작합니다.

### D. 보안 스캔 결과 집계

컨슈머 리포지토리에서 실행된 SAST/DAST 도구의 결과를 수집합니다.

* **방법**: Code Scanning API를 호출하여 해당 Run에서 발생한 보안 경고(Alerts) 개수를 확인하고, 일정 기준(예: High Severity > 0)을 초과하면 통합 테스트를 실패 처리합니다.
