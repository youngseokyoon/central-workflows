# Central Workflows Integration Testing

이 문서는 `central-workflows` repository의 변경사항이 consumer repository들에 미치는 영향을 테스트하는 방법을 설명합니다.

## 🎯 목적

`central-workflows`의 PR이 생성되면:

1. 자동으로 모든 consumer repository의 **실제 CI workflow**를 trigger
2. 각 repository의 CI 결과를 모니터링
3. PR merge 전에 호환성 검증

## 🔧 작동 방식

### Integration Test Workflow

**파일**: `.github/workflows/integration-test.yml`

**트리거**:

- PR이 `main` branch로 생성될 때
- `.github/workflows/` 내 파일이 변경될 때
- 수동 실행 (`workflow_dispatch`)

**프로세스**:

```
PR Created
    ↓
Matrix Strategy (병렬 실행)
    ├─ sample-cpp-repo: gh workflow run ci.yml
    ├─ sample-java-repo: gh workflow run ci.yml
    ├─ sample-binary-repo: gh workflow run ci.yml
    └─ sample-multi-lang-repo: gh workflow run ci.yml
         ↓
    각 workflow 완료 대기
         ↓
    결과 확인 (success/failure)
         ↓
    ✅ All Pass → Safe to Merge
```

## 📋 사용 방법

### 1. Feature Branch에서 작업

```bash
cd central-workflows
git checkout -b feature/update-workflow

# workflow 수정
vim .github/workflows/compile-cpp.yml

# commit
git add .
git commit -m "feat: Update compile-cpp workflow"
git push origin feature/update-workflow
```

### 2. PR 생성

```bash
gh pr create \
  --title "feat: Update compile-cpp workflow" \
  --body "Update compiler settings for better compatibility"
```

또는 GitHub UI에서 PR 생성

### 3. Integration Test 자동 실행

PR이 생성되면 자동으로:

1. 4개 consumer repository의 CI를 trigger
2. 병렬로 모든 workflow 실행
3. 결과를 PR checks에 표시

### 4. PR Checks 확인

PR 페이지 → "Checks" 탭:

```
✅ trigger-consumer-tests (sample-cpp-repo)
✅ trigger-consumer-tests (sample-java-repo)
✅ trigger-consumer-tests (sample-binary-repo)
✅ trigger-consumer-tests (sample-multi-lang-repo)
✅ integration-summary
```

모든 체크가 통과해야 safe to merge!

## 💡 장점

### ✅ 실제 CI를 테스트

- Consumer repo의 진짜 CI workflow 실행
- ci.yml을 수정할 필요 없음
- 정확한 호환성 검증

### ✅ 간단한 구조

- 빌드 로직 중복 없음
- `gh workflow run`으로 trigger
- `gh run watch`로 결과 확인

### ✅ 병렬 실행

- Matrix strategy로 모든 repo 동시 테스트
- 빠른 피드백

## 🚨 주의사항

### GITHUB_TOKEN 권한

기본 `GITHUB_TOKEN`으로 다른 repository의 workflow를 trigger할 수 있습니다.

만약 권한 오류가 발생하면:

1. Settings → Actions → General
2. "Workflow permissions" 확인
3. 필요시 Personal Access Token (PAT) 사용:

   ```yaml
   env:
     GH_TOKEN: ${{ secrets.PAT_TOKEN }}
   ```

### Timing Issue

Workflow trigger 후 바로 run list를 조회하면 찾지 못할 수 있습니다.
현재는 10초 대기 (`sleep 10`)로 해결.

필요시 retry 로직 추가 가능:

```bash
for i in {1..30}; do
  RUN_ID=$(gh run list ... --limit 1 ...)
  if [ -n "$RUN_ID" ]; then break; fi
  sleep 2
done
```

## 🔍 실패 시 대응

### CI 실패 확인

```
❌ trigger-consumer-tests (sample-cpp-repo)
```

1. **실패한 job 클릭**
2. **"Check workflow result" step 확인**
3. **Consumer repo의 실제 CI run으로 이동**:

   ```
   https://github.com/youngseokyoon/sample-cpp-repo/actions
   ```

4. **실패 원인 파악 및 수정**

### Workflow가 trigger되지 않는 경우

- Consumer repo에 `ci.yml`이 있는지 확인
- `workflow_dispatch` trigger가 설정되어 있는지 확인
- GITHUB_TOKEN 권한 확인

## 🎓 예제: Breaking Change 방지

### Scenario

`compile-cpp.yml`에서 필수 parameter 추가:

```yaml
inputs:
  new-required-param:
    required: true  # ⚠️ Breaking change!
```

### Integration Test 결과

```
❌ sample-cpp-repo CI failed
   Error: Missing required input 'new-required-param'
```

### 올바른 수정

```yaml
inputs:
  new-required-param:
    required: false  # ✅ Backward compatible
    default: 'default-value'
```

Integration test가 다시 통과하면 safe to merge!

## 🚀 고급 활용

### 특정 Repository만 테스트

```yaml
strategy:
  matrix:
    repo:
      - sample-cpp-repo  # 이것만 테스트
```

### 수동 실행

```bash
gh workflow run integration-test.yml \
  --repo youngseokyoon/central-workflows \
  --ref feature/your-branch
```

## 📊 Best Practices

1. **PR 생성 전 local test**: `act pull_request` (선택)
2. **작은 단위로 변경**: 한 번에 하나의 workflow만 수정
3. **Changelog 업데이트**: CHANGELOG.md에 변경사항 기록
4. **모든 체크 통과 확인**: Integration test 모두 ✅ 후 merge

## 🎉 결론

이제 `central-workflows`를 안전하게 수정할 수 있습니다!

- PR 생성 시 자동 검증
- Consumer repo 영향도 자동 확인
- Breaking change 사전 방지
