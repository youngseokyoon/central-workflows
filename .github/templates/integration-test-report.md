## {{EMOJI}} Integration Test Results

**Status**: {{STATUS}}

### Consumer Repository Tests

| Repository | Status |
|------------|--------|
| `sample-cpp-repo` | {{CPP_STATUS}} |
| `sample-java-repo` | {{JAVA_STATUS}} |
| `sample-binary-repo` | {{BINARY_STATUS}} |
| `sample-multi-lang-repo` | {{MULTI_STATUS}} |

---

<details>
<summary>ℹ️ What was tested?</summary>

This PR triggered actual CI workflows in all consumer repositories to verify compatibility.

- Each repository's **real CI workflow** was executed
- Tests ran against the `main` branch of each consumer repo
- All workflows used the **current state** of central-workflows

</details>

{{CONCLUSION}}

---
*🤖 Automated by Integration Test workflow*
