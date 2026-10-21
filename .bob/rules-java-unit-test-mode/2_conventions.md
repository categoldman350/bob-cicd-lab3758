# Java Unit Test Writer — Repo Conventions

These conventions are extracted directly from the existing tests in this repository.
Follow them exactly for every test you write.

---

## Test naming

Pattern: `methodName_scenario_expectedBehaviour`

```java
void getAllOrders_returnsAllOrders()
void getOrderById_existingId_returnsOrder()
void getOrderById_nonExistingId_returnsEmpty()
void updateOrderStatus_invalidTransition_throwsException()
```

- All lowercase camelCase words separated by underscores between the three segments.
- No `test` prefix on method names.
- The scenario and expected-behaviour segments should be concise but unambiguous.

---

## Service-layer tests (`*ServiceTest.java`)

### Class skeleton

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;   // one @Mock per dependency

    @InjectMocks
    private OrderService orderService;         // class under test

    private Order testOrder;                   // shared fixture field

    @BeforeEach
    void setUp() {
        testOrder = new Order();
        testOrder.setId(1L);
        testOrder.setCustomerName("John Doe");
        testOrder.setProduct("Widget");
        testOrder.setAmount(new BigDecimal("29.99"));
        testOrder.setStatus("PENDING");
    }

    @Test
    void methodName_scenario_expectedBehaviour() {
        // arrange
        when(orderRepository.someMethod(...)).thenReturn(...);

        // act
        SomeType result = orderService.methodUnderTest(...);

        // assert  — AssertJ only
        assertThat(result)...;
    }
}
```

### Key rules
- `@ExtendWith(MockitoExtension.class)` — never `@SpringBootTest` for pure service tests.
- Declare all mocks as fields with `@Mock`; never create them with `Mockito.mock()` inline.
- One `@InjectMocks` field for the class under test.
- Shared fixture built in `setUp()` with `@BeforeEach`.
- Use `when(...).thenReturn(...)` for stubbing, never `doReturn(...).when(...)` unless void methods require it.
- Use `verify(mock).method(...)` to assert side-effects (e.g. `deleteById`).

### Imports (static)

```java
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.when;
import static org.mockito.Mockito.verify;
```

---

## Controller-layer tests (`*ControllerTest.java`)

### Class skeleton

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private OrderService orderService;         // @MockBean, NOT @Mock

    @Autowired
    private ObjectMapper objectMapper;

    @Test
    void methodName_scenario_expectedBehaviour() throws Exception {
        Order order = new Order();
        order.set...(...);
        ...

        when(orderService.someMethod(...)).thenReturn(...);

        mockMvc.perform(get("/api/orders/..."))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.fieldName").value("expectedValue"));
    }
}
```

### Key rules
- `@WebMvcTest(OrderController.class)` — slice test, not `@SpringBootTest`.
- Use `@MockBean` (Spring-managed) for service dependencies — never `@Mock` in controller tests.
- Inject `ObjectMapper` with `@Autowired` for serialising request bodies.
- Assert HTTP status with `MockMvcResultMatchers.status()`.
- Assert response body fields with `jsonPath`.
- No AssertJ `assertThat` on raw strings from `andReturn()` — use `jsonPath` chains instead.
- Serialise request bodies with `objectMapper.writeValueAsString(obj)`.

### Imports

```java
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;
```

---

## Assertion library rules

| Situation | Use |
|---|---|
| Simple value equality | `assertThat(x).isEqualTo(y)` |
| Optional present | `assertThat(opt).isPresent()` |
| Optional empty | `assertThat(opt).isEmpty()` |
| Collection size | `assertThat(list).hasSize(n)` |
| Exception type + message | `assertThatThrownBy(() -> ...).isInstanceOf(X.class).hasMessageContaining("...")` |

**Never use**: `assertEquals`, `assertTrue`, `assertNull` (JUnit assert methods), or Hamcrest `assertThat`/`is`.

---

## What NOT to do
- Do not add `@DisplayName` — not used in this codebase.
- Do not use `@ParameterizedTest` unless the user specifically requests it.
- Do not mix `@SpringBootTest` into the service-layer tests.
- Do not introduce constructor injection in tests; use field injection via annotations.
- Do not add Lombok annotations to test classes.
