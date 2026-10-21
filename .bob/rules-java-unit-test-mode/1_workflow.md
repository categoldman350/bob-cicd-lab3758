# Java Unit Test Writer — Workflow

## Overview

Write JUnit 5 unit tests for the order-service Spring Boot application.
Always read existing tests before writing new ones, so every new test file is indistinguishable in style from the ones already in the repo.

---

## Step-by-step workflow

### 1. Read the user's request
Identify:
- Which class or method needs new tests
- Whether this is service-layer, controller-layer, or other
- Any specific scenario or edge case the user wants covered

### 2. Read the production source
Before writing anything, read the class under test:
```
order-service/src/main/java/com/example/orders/...
```
Understand the method signatures, return types, checked exceptions, and status-transition logic.

### 3. Read the existing tests for that layer
Always open the relevant existing test file first:

| Layer | Existing test file |
|---|---|
| Service | `order-service/src/test/java/com/example/orders/service/OrderServiceTest.java` |
| Controller | `order-service/src/test/java/com/example/orders/controller/OrderControllerTest.java` |

Study the file for:
- Import style (static imports, assertion library)
- Mocking annotation style (`@Mock`/`@InjectMocks` vs `@MockBean`)
- Fixture setup pattern (`@BeforeEach setUp()`)
- Test method naming pattern

### 4. Write the test(s)
Apply the conventions observed in step 3. See `2_conventions.md` for the canonical rules.

### 5. Append to the correct test file
- If the test class already exists, add the new `@Test` method(s) inside it.
- If a new test class is needed, mirror the package structure and follow the existing class skeleton exactly.

### 6. Verify consistency
Re-read the test file after editing and confirm:
- No new imports that contradict the existing import style
- Naming pattern is consistent
- No assertion library mixing (AssertJ only, no Hamcrest, no JUnit `assertEquals`)
