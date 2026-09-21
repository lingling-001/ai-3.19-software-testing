# Lesson 3.19: Software Testing — Unit and Integration Testing

## Lesson Overview

This lesson introduces automated testing in a Spring Boot application. You will write unit tests with JUnit, isolate the service layer using Mockito, and test REST endpoints end to end using MockMvc. All work is done inside the `simple-crm` project carried forward from earlier lessons.

## Lesson Objectives

By the end of this lesson, learners will be able to:

1. **Explain** the purpose of automated testing and where unit and integration tests sit
2. **Write** unit tests using JUnit following the Arrange-Act-Assert pattern
3. **Mock** dependencies using Mockito to test the service layer in isolation
4. **Perform** integration testing of REST endpoints using Spring Boot and MockMvc

## Session Plan

| Part | Topic | Time |
|---|---|---|
| 1 | Introduction to Software Testing | 10 min |
| 2 | Unit Testing with JUnit (incl. activity) | 45 min |
| — | Break | 10 min |
| 3 | Service Layer Testing with Mockito | 40 min |
| 4 | Integration Testing with MockMvc | 50 min |
| — | Wrap-up | 5 min |

---

## Part 1: Introduction to Software Testing

Software testing is the process of verifying that an application behaves correctly and continues to behave correctly as the code changes.

At a fundamental level, testing answers two questions:

1. Does the software do what it is supposed to do?
2. Does it continue to work correctly when the code changes?

Up to now we have tested `simple-crm` by running it and calling endpoints manually in Postman. That works, but it does not scale. Every time you change a line of code you would have to re-run every request by hand to be confident nothing broke. Automated tests do that for you in seconds.

### Why Automated Tests Matter

- Catch regressions the moment they are introduced, not in production
- Give you confidence to refactor
- Run automatically in a CI/CD pipeline on every push, so a broken build is caught before it reaches anyone else
- Document what the code is supposed to do

### The Two Levels We Cover

<img src="./assets/images/software-testing.jpg" width=500 style="background-color: #fff; padding: 20px;border-radius: 5px;border: 1px solid #eee;">

**Unit Testing** — tests a single method or class in complete isolation. Dependencies are mocked. Runs in milliseconds.

**Integration Testing** — tests how components work together. Real Spring context, real database, real wiring. Slower, but catches problems unit tests cannot.

Other levels exist above these — end-to-end, system, acceptance, performance, security testing — and in most organisations they are owned by QA or platform teams. This lesson focuses on the two levels that developers write and maintain themselves.

### The Test Pyramid

The standard industry shape is many fast unit tests at the base, fewer integration tests in the middle, and very few slow end-to-end tests at the top. If that pyramid gets inverted — a handful of unit tests and hundreds of slow end-to-end tests — your build time becomes the bottleneck and people stop running tests locally.

> 📖 **Self Reading — Other testing types:** Functional/End-to-End testing validates a complete user workflow. System testing validates the application as a whole. Acceptance testing (UAT) is run by business stakeholders to confirm requirements are met. Regression testing confirms existing functionality still works after a change. Performance testing evaluates speed and scalability under load. Security testing checks for vulnerabilities such as SQL injection and XSS. Read more: https://www.guru99.com/software-testing-introduction-importance.html

---

## Part 2: Unit Testing with JUnit

### What is Unit Testing?

A unit test exercises one unit of code — usually a single method — in complete isolation from everything else. No database, no Spring context, no network. That isolation is what makes unit tests fast enough to run on every save.

### Frameworks

- [JUnit 5](https://junit.org/junit5/) — the test framework: creates and runs tests
- [Mockito](https://site.mockito.org/) — the mocking framework: fakes dependencies

Both ship inside `spring-boot-starter-test`, which Spring Initializr adds by default. Nothing to install.

### Setting Up the Demo Class

We will start with a class that has **no dependencies at all**, so we can focus on JUnit itself before introducing mocking.

Create `DemoService.java` in `src/main/java/sg/edu/ntu/simple_crm/service/`:

```java
package sg.edu.ntu.simple_crm.service;

public class DemoService {

    public int calculateAge(int yearOfBirth, int currentYear) {
        return currentYear - yearOfBirth;
    }

    public String formatFullName(String firstName, String lastName) {
        return firstName + " " + lastName;
    }
}
```

Notice that `calculateAge` takes the current year as a parameter instead of calling `Year.now()` inside the method. That is deliberate.

> 📖 **Self Reading — Why pass the year in:** If the method read the system clock internally, the test would have to calculate the expected answer using the same clock, which means the test proves nothing. Worse, a test written this year could start failing on the 1st of January. Passing time-dependent values in as parameters — or injecting a `Clock` in a larger system — is standard practice precisely because it makes the logic deterministic and testable.

### Creating the Test Class

Test files must mirror the source folder structure exactly:

- Source: `src/main/java/sg/edu/ntu/simple_crm/service/DemoService.java`
- Test: `src/test/java/sg/edu/ntu/simple_crm/service/DemoServiceTest.java`

Spring Initializr only creates `SimpleCrmApplicationTests.java`. Every other test folder and file is created by you.

The quickest way is to let VS Code do it:

1. Open `DemoService.java` and **click inside the editor** (not the file explorer)
2. Right-click → **Source Action...** → **Generate Tests...**
3. Select the methods to generate stubs for, and confirm

VS Code creates the mirrored folder path, the test class, the correct `package` line and a stub for each method. This avoids the most common setup error, which is a package declaration that does not match the folder path.

> ⚠️ **If the option does not appear:** the cursor must be inside the Java editor, not the file explorer. Searching `Java: Generate Tests` in the Command Palette is unreliable across extension versions — use the right-click Source Action route. As a fallback, create the `service` folder and `DemoServiceTest.java` manually under `src/test/java/sg/edu/ntu/simple_crm/` and type the package line yourself.

The generator produces method names like `testCalculateAge` and leaves methods package-private. We will rename them to the convention below. Package-private is valid in JUnit 5 — unlike JUnit 4, `public` is no longer required — but we use `public` here for consistency.

### Writing the Test

Every unit test follows three steps, known as the **Arrange-Act-Assert** pattern (also called Given-When-Then):

- **Arrange** — set up inputs and expected values
- **Act** — call the one method under test
- **Assert** — verify the result

```java
package sg.edu.ntu.simple_crm.service;

import static org.junit.jupiter.api.Assertions.assertEquals;

import org.junit.jupiter.api.Test;

public class DemoServiceTest {

  @Test
  public void calculateAge_validYear_returnsCorrectAge() {
    // 1. ARRANGE
    DemoService demoService = new DemoService();
    int expectedAge = 35;

    // 2. ACT
    int actualAge = demoService.calculateAge(1990, 2025);

    // 3. ASSERT
    assertEquals(expectedAge, actualAge, "Age should be current year minus year of birth");
  }
}
```

Run the test by clicking the green arrow in the gutter next to the method. A passing test shows a green tick and **no console output** — silence means success. Output appears only on failure, showing expected versus actual.

To run every test in the project at once, use `./mvnw test` from the terminal. This is the same command your CI pipeline runs.

> ⚠️ **No green arrow in the gutter?** This usually means the Java language server has lost track of the file, and it happens most often right after you add a new method to a class in `main` while the test file is open. Run **Java: Clean Java Language Server Workspace** from the Command Palette and let VS Code reload. The arrows come back. Check the Problems panel first, though — a compile error anywhere in the test file hides the arrows for every test in it, not just the broken one.

Note that we created the object with `new`. Unit tests do not start the Spring context, so there are no beans to inject. This is exactly why they run in milliseconds.

> 📖 **Self Reading — Why `new` instead of DI:** The fact that a class can be tested with a plain `new` is a sign it is well designed. If a class cannot be instantiated without the Spring container, that is a coupling problem, not a testing problem.

> 📖 **Self Reading — Constructor injection:** Constructor injection (rather than `@Autowired` on a field) is the current industry standard and it matters directly for testing. It makes dependencies explicit, allows fields to be `final`, and lets you construct the class directly in a test without starting Spring. A class that can only be built by the Spring container is a class that is hard to unit test.

### Seeing the Test Fail

Introduce a deliberate bug:

```java
public int calculateAge(int yearOfBirth, int currentYear) {
    return yearOfBirth - currentYear; // wrong order
}
```

Run the test again. It fails, and the output tells you it expected 35 and got -35. This is the whole point — the test caught a regression the moment it was introduced. Fix it before moving on.

Seeing a test go red matters more than it looks. A test that has never failed might be asserting nothing at all, and you would have no way of knowing. This is the reasoning behind **Test Driven Development**, where you write the failing test first and then write the code that makes it pass — red, green, refactor. Not every team works that way, and most use it selectively for critical logic rather than across the whole codebase, but the underlying idea holds regardless: prove the test can fail before you trust it to pass.

### Test Naming

Tutorials often name tests `testCalculateAge`. In production codebases the convention is:

```
methodName_scenario_expectedBehaviour
```

So `calculateAge_validYear_returnsCorrectAge`. When a build fails at 2am, the test name alone should tell you what broke and under what conditions, without opening the file.

> 📖 **Self Reading — Other naming styles:** A BDD (Behaviour Driven Development) style also exists: `givenValidYear_whenCalculateAge_thenReturnCorrectAge`. BDD writes tests in near-English so non-technical stakeholders can read them. Either convention is fine — consistency within a codebase matters more than which one you pick.

### Reducing Repetition with `@BeforeEach`

Once you have more than one test, every one of them starts by creating a `DemoService`. That repetition can move into a lifecycle method that runs before every test:

Add the import `org.junit.jupiter.api.BeforeEach`, then refactor the class:

```java
public class DemoServiceTest {

  DemoService demoService;

  @BeforeEach
  public void init() {
    demoService = new DemoService();
  }

  @Test
  public void calculateAge_validYear_returnsCorrectAge() {
    // 1. ARRANGE — demoService already created by init()
    int expectedAge = 35;

    // 2. ACT
    int actualAge = demoService.calculateAge(1990, 2025);

    // 3. ASSERT
    assertEquals(expectedAge, actualAge, "Age should be current year minus year of birth");
  }
}
```

The `new DemoService()` line is gone from the test itself. A fresh instance is created before every test method, which keeps tests independent — no test can leave state behind that affects the next one. You will use this in the activity.

> 📖 **Self Reading — Full lifecycle annotations:**
>
> | Annotation | Description |
> |---|---|
> | `@BeforeAll` | Runs once before all tests in the class (must be `static`) |
> | `@BeforeEach` | Runs before each test method |
> | `@AfterEach` | Runs after each test method |
> | `@AfterAll` | Runs once after all tests in the class (must be `static`) |
>
> `@BeforeAll` and `@AfterAll` are typically used for expensive one-time setup such as starting a test container or opening a connection pool.

> 📖 **Self Reading — Common assertions:**
>
> | Method | Description |
> |---|---|
> | `assertEquals()` | Two values are equal |
> | `assertNotEquals()` | Two values are not equal |
> | `assertTrue()` / `assertFalse()` | A condition holds |
> | `assertNull()` / `assertNotNull()` | An object is or is not null |
> | `assertArrayEquals()` | Two arrays are equal |
> | `assertThrows()` | An exception is thrown |
>
> Full reference: https://junit.org/junit5/docs/current/user-guide/#writing-tests-assertions

> 📖 **Self Reading — Generating an HTML report:** Running `mvn surefire-report:report` produces `target/site/surefire-report.html`, showing which tests passed, failed, and how long each took. In practice you rarely run this locally — CI pipelines (GitHub Actions, Jenkins) generate and publish it automatically on every push so the whole team can see results without checking out the code.

### Testing the Same Method with Many Values

The test above proves `calculateAge` works for one pair of years. What if we want to check several — a normal case, the boundary, someone born this year?

Writing a separate `@Test` for each would mean four near-identical methods. JUnit has a better way:

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
```

```java
@ParameterizedTest
@CsvSource({
    "1990, 2025, 35",
    "1965, 2025, 60",
    "2000, 2025, 25",
    "2025, 2025, 0"
})
public void calculateAge_variousYears_returnsCorrectAge(int yearOfBirth, int currentYear, int expectedAge) {
    // 1. ACT
    int actualAge = demoService.calculateAge(yearOfBirth, currentYear);

    // 2. ASSERT
    assertEquals(expectedAge, actualAge);
}
```

Each row becomes the method's parameters, in order. Row 1 supplies `yearOfBirth = 1990`, `currentYear = 2025`, `expectedAge = 35`.

Four things to notice:

- **`@ParameterizedTest` replaces `@Test`.** Do not use both.
- **Each row runs as a separate test.** Four rows means four entries in the test tree, each labelled with its values. If only the 1965 row fails, the other three still show green — you see exactly which input broke.
- **The expected values are still hardcoded**, one per row. That never changes. You are listing your known-good answers in a table instead of scattering them across methods. Adding a fifth case is one more line, not a new method.
- **`@CsvSource` has nothing to do with CSV files.** No file is read. It is just a convenient way to write rows as comma-separated strings inline.

Use a plain `@Test` when one example proves the behaviour. Use `@ParameterizedTest` when you are checking the same rule across a range of inputs — which in practice usually means boundaries and edge cases.

> 📖 **Self Reading — Parameterized test sources:** `@CsvSource` is the most common, but JUnit also offers `@ValueSource` for a single parameter, `@EnumSource` for every value of an enum, `@MethodSource` for programmatically generated arguments, and `@CsvFileSource` for reading an actual CSV file. `@CsvFileSource` exists but is uncommon — driving tests from an external data file makes them slower and dependent on something outside the repository. In production the data stays small, hardcoded and version-controlled next to the test.

### 👨‍💻 Activity (15 minutes)

There are two tasks. Both tests go in `DemoServiceTest`.

**Task 1 — test an existing method.**
`formatFullName` is already in `DemoService`. Write a unit test for it. You only write the test; the method is already there.

Use a plain `@Test` for Task 1 and a `@ParameterizedTest` for Task 2.

**Task 2 — add a new method, then test it with multiple values.**
Add `isSeniorCustomer` to `DemoService`:

```java
public boolean isSeniorCustomer(int yearOfBirth, int currentYear) {
    return (currentYear - yearOfBirth) >= 60;
}
```

Then write a `@ParameterizedTest` for it using these rows:

```java
@CsvSource({
    "1950, 2025, true",    // 75 years old
    "1965, 2025, true",    // exactly 60 — the boundary
    "1966, 2025, false",   // 59 — just under
    "1990, 2025, false"    // 35 years old
})
```

Your method signature takes three parameters: `int yearOfBirth`, `int currentYear`, `boolean expected`. Use `assertEquals(expected, actual)`.

**Requirements for both tasks:**

- Follow the Arrange-Act-Assert pattern
- Use the `methodName_scenario_expectedBehaviour` naming convention
- Rely on the `@BeforeEach` method — neither test should create its own `DemoService`

**Think about:** rows 2 and 3 are the ones that matter. The 75-year-old and the 35-year-old pass whether the code says `>= 60` or `> 60`. Only the pair either side of the boundary can tell those two apart. Boundary conditions are where most production bugs actually live.

---

## Part 3: Service Layer Unit Testing with Mockito

`DemoService` had no dependencies, which is why we could test it with `new`. Real services are not like that. `CustomerServiceImpl` depends on `CustomerRepository`, and we do not want a unit test hitting a real database — that would be slow, and a failure would not tell us whether our logic is wrong or the database is down.

The solution is **mocking**. Mockito creates a fake `CustomerRepository`. When the service calls it, Mockito intercepts the call, the real repository is never invoked, no database is touched, and Mockito returns whatever you told it to return.

The point is important: by mocking the repository you are testing **your own logic**, not Hibernate's. Each layer is tested in isolation and owns its own tests.

### Prepare the Customer Class

Add these Lombok annotations to `Customer` if not already present:

```java
@Builder            // fluent object creation in tests
@EqualsAndHashCode  // value-based equality, required by assertEquals()
@NoArgsConstructor  // required by JPA
@AllArgsConstructor // required by @Builder when other constructors exist
public class Customer {
  // ...
}
```

`@EqualsAndHashCode` is not optional here. Without it, Java compares object **references** — two separate `Customer` objects holding identical data sit at different memory addresses and `assertEquals()` fails even though they look the same. `@EqualsAndHashCode` generates `equals()` and `hashCode()` based on field values, which is what the test actually means by "equal".

> 📖 **Self Reading — Reference vs value equality:** This is the same reference-semantics issue that shows up whenever you pass objects around. `==` and the default `equals()` ask "is this the same object in memory?" A value-based `equals()` asks "do these two objects hold the same data?" In JPA entities and DTOs you almost always want the second, which is why this annotation appears on nearly every entity in a real codebase.

### Setting Up the Test Class

- Source: `src/main/java/sg/edu/ntu/simple_crm/service/CustomerServiceImpl.java`
- Test: `src/test/java/sg/edu/ntu/simple_crm/service/CustomerServiceImplTest.java`

```java
package sg.edu.ntu.simple_crm.service;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.mockito.Mockito.times;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

import java.util.Optional;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import sg.edu.ntu.simple_crm.exceptions.CustomerNotFoundException;
import sg.edu.ntu.simple_crm.model.Customer;
import sg.edu.ntu.simple_crm.repository.CustomerRepository;

@ExtendWith(MockitoExtension.class)
public class CustomerServiceImplTest {

  @Mock
  private CustomerRepository customerRepository;

  @InjectMocks
  private CustomerServiceImpl customerService;

}
```

- `@ExtendWith(MockitoExtension.class)` — enables Mockito for JUnit 5
- `@Mock` — creates a fake `CustomerRepository`
- `@InjectMocks` — creates a real `CustomerServiceImpl` and injects the mock into it

> ⚠️ **Common mistake:** `@InjectMocks` must target the **concrete class** (`CustomerServiceImpl`), not the interface (`CustomerService`). Mockito has to instantiate the class to inject into it, and it cannot instantiate an interface.

### The Two Mockito Calls You Need

Before writing the tests, understand the two methods that do all the work:

- `when(...).thenReturn(...)` — **programs** the mock. "When `save()` is called with this customer, hand back this customer." This is setup, not an assertion. It never fails a test.
- `verify(...)` — **asserts on the interaction**. "Confirm `save()` was actually called, exactly once." This catches a whole class of bug where the method returns a plausible-looking result but never actually called the repository at all.

`verify()` is the one developers routinely skip, and it is the one that catches the silent failures.

All test methods below go inside the `CustomerServiceImplTest` class body.

### Test Create Customer

```java
@Test
public void createCustomer_validCustomer_returnsSavedCustomer() {

  // 1. ARRANGE
  Customer customer = Customer.builder()
      .firstName("Clint").lastName("Barton")
      .email("clint@avengers.com").contactNo("12345678")
      .jobTitle("Special Agent").yearOfBirth(1975)
      .build();

  // Program the mock. The real repository is never called, no database is touched.
  when(customerRepository.save(customer)).thenReturn(customer);

  // 2. ACT
  Customer savedCustomer = customerService.createCustomer(customer);

  // 3. ASSERT
  assertEquals(customer, savedCustomer, "The saved customer should match the new customer");
  verify(customerRepository, times(1)).save(customer);
}
```

### Test Get Customer Not Found

```java
@Test
public void getCustomer_missingId_throwsCustomerNotFoundException() {
  // 1. ARRANGE
  Long customerId = 1L;

  // Optional.empty() simulates no record found
  when(customerRepository.findById(customerId)).thenReturn(Optional.empty());

  // 2. ACT + 3. ASSERT
  assertThrows(CustomerNotFoundException.class, () -> customerService.getCustomer(customerId));
}
```

`assertThrows(ExceptionClass, lambda)` runs the lambda and verifies the expected exception is thrown. If no exception is thrown, the test fails.

Spring Data JPA's `findById()` returns `Optional<Customer>` rather than `Customer`. An `Optional` either holds a value (`Optional.of(customer)`) or is empty (`Optional.empty()`). Here we program the mock to return empty, which is how the repository signals "no record found" — and that is what drives the service into throwing. Returning `Optional` instead of `null` forces the caller to handle the missing case explicitly.

This test is where the exception handling from Lesson 3.16 pays off. Testing the failure path is at least as important as testing the happy path — most production incidents happen on paths nobody tested.

---

## Part 4: Integration Testing with MockMvc

Unit tests validate components in isolation. Integration tests validate that they work together — the full request and response cycle from controller through service to repository to database.

### Mockito vs MockMvc

| Tool | Tests | Mocks | Database |
|---|---|---|---|
| **Mockito** | Service layer | Repository (fake) | Not touched |
| **MockMvc** | Controller layer (full stack) | HTTP transport only | Real |

**MockMvc** simulates the HTTP layer — it pretends to be a client sending a request to your API and checks the response. Everything behind the controller is real.

> 📖 **Self Reading — Common misconception:** The "Mock" in MockMvc refers only to the fake HTTP transport. No server starts, no port is opened. It does **not** mock your application layers. `@SpringBootTest` wires the full context, so integration tests do real database work — which is exactly why they are slower than unit tests.

> 📖 **Self Reading — `@SpringBootTest` vs `@WebMvcTest`:** `@SpringBootTest` loads the full application context — all beans, datasource, security. `@WebMvcTest` loads only the web layer (controllers, filters, `@ControllerAdvice`) with services mocked, so it is much faster. Production teams tend to prefer `@WebMvcTest` for controller logic and reserve `@SpringBootTest` for genuine end-to-end database tests. We use `@SpringBootTest` here because we want the full stack.

### Setting Up the Integration Test

- Source: `src/main/java/sg/edu/ntu/simple_crm/controller/CustomerController.java`
- Test: `src/test/java/sg/edu/ntu/simple_crm/controller/CustomerControllerTest.java`

The static imports on `MockMvcResultMatchers` and `MockMvcRequestBuilders` are essential — without them `status()`, `content()` and `jsonPath()` will not resolve.

```java
package sg.edu.ntu.simple_crm.controller;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.RequestBuilder;
import org.springframework.transaction.annotation.Transactional;

import com.fasterxml.jackson.databind.ObjectMapper;

import sg.edu.ntu.simple_crm.model.Customer;
```

> ⚠️ **Import gotcha:** use `org.springframework.transaction.annotation.Transactional`, **not** `jakarta.transaction.Transactional`. Both will compile and VS Code often suggests the wrong one first, but only the Spring version gives you automatic rollback in tests.

```java
@SpringBootTest
@AutoConfigureMockMvc
@Transactional
public class CustomerControllerTest {

  @Autowired
  private MockMvc mockMvc;

  @Autowired
  private ObjectMapper objectMapper;
}
```

- `@SpringBootTest` — loads the full application context
- `@AutoConfigureMockMvc` — auto-wires the `MockMvc` bean
- `@Transactional` — each test runs in a transaction that is **rolled back** afterwards, so writes never persist
- `ObjectMapper` — converts Java objects to JSON strings (provided by Jackson)

`@Transactional` solves a real problem. Without it, every test that writes data leaves that data behind, and tests start polluting each other — a record created in test 1 changes the count in test 2. Rollback gives each test a clean slate with no manual cleanup scripts.

### Understanding the Test Structure

Every MockMvc test follows the same three-part shape.

**Build the request** using `MockMvcRequestBuilders`:

```java
MockMvcRequestBuilders.get("/customers")      // GET
MockMvcRequestBuilders.post("/customers")     // POST
MockMvcRequestBuilders.put("/customers/1")    // PUT
MockMvcRequestBuilders.delete("/customers/1") // DELETE
```

**Send it** with `mockMvc.perform(request)` — the equivalent of hitting Send in Postman.

**Assert** with chained `.andExpect()` calls, one assertion each:

```java
.andExpect(status().isOk())                                    // HTTP 200
.andExpect(content().contentType(MediaType.APPLICATION_JSON))  // response is JSON
.andExpect(jsonPath("$.id").value(1))                          // id field equals 1
```

> 📖 **Self Reading — JsonPath:** JsonPath is a query language for JSON. `$` is the root of the response. `$.id` means "the `id` field at the root", `$.size()` means "the size of the root array". If the API returns `{"id": 1, "firstName": "John"}`, then `jsonPath("$.firstName").value("John")` asserts that field equals `"John"`.

All test methods below go inside the `CustomerControllerTest` class body. Each one declares `throws Exception`, because `mockMvc.perform()` is a checked-exception method — there is nothing to handle, the test simply fails if it throws.

### Test Get Customer by ID

```java
@Test
public void getCustomerById_existingId_returnsOk() throws Exception {
  // Step 1: Build a GET request to /customers/1
  RequestBuilder request = MockMvcRequestBuilders.get("/customers/1");

  // Step 2: Perform and assert
  mockMvc.perform(request)
      .andExpect(status().isOk())
      .andExpect(content().contentType(MediaType.APPLICATION_JSON))
      .andExpect(jsonPath("$.id").value(1));
}
```

This test relies on the `DataLoader` having created a customer with ID 1, which is safe here because `spring.jpa.hibernate.ddl-auto=create` drops and recreates the schema on every startup.

Be aware this is a fragile pattern in general. On a shared or persistent database, where data accumulates between runs, assertions on specific IDs or exact record counts break for reasons that have nothing to do with the code under test. This is exactly why `@Transactional` rollback and controlled test data matter in a real pipeline.

### Test Valid Customer Creation

```java
@Test
public void createCustomer_validCustomer_returnsCreated() throws Exception {
  // Step 1: Create a Customer object
  Customer newCustomer = Customer.builder()
      .firstName("Clint").lastName("Barton")
      .email("clint@avengers.com").contactNo("12345678")
      .jobTitle("Special Agent").yearOfBirth(1975)
      .build();

  // Step 2: Convert the Java object to a JSON-formatted String
  // Produces: {"firstName":"Clint","lastName":"Barton",...}
  String newCustomerAsJSON = objectMapper.writeValueAsString(newCustomer);

  // Step 3: Build the POST request
  // .contentType tells Spring to deserialize the body back into a Customer
  RequestBuilder request = MockMvcRequestBuilders.post("/customers")
      .contentType(MediaType.APPLICATION_JSON)
      .content(newCustomerAsJSON);

  // Step 4: Perform and assert
  mockMvc.perform(request)
      .andExpect(status().isCreated())
      .andExpect(content().contentType(MediaType.APPLICATION_JSON))
      .andExpect(jsonPath("$.id").exists())
      .andExpect(jsonPath("$.firstName").value("Clint"))
      .andExpect(jsonPath("$.lastName").value("Barton"));
}
```

Note `$.id` is asserted with `.exists()` rather than a specific number. Asserting an exact ID is brittle — it breaks the moment the DataLoader changes or tests run in a different order. Assert that the field exists and holds a valid value, not that it equals 5.

Because `@Transactional` is on the class, this record is rolled back after the test. The database is left exactly as it was.

> 📖 **Self Reading — JSON is always text on the wire:** `objectMapper.writeValueAsString()` is doing programmatically what Postman does visually — serialising your object to a JSON string, setting the Content-Type header, and sending it. There is no such thing as sending a Java object over HTTP.

### Test Invalid Customer Creation

```java
@Test
public void createCustomer_invalidCustomer_returnsBadRequest() throws Exception {
  // firstName and lastName are blank — violates @NotBlank
  // email is malformed — violates @Email
  Customer invalidCustomer = Customer.builder()
      .firstName("  ")
      .lastName("  ")
      .email("not-a-valid-email")
      .contactNo("12345678")
      .jobTitle("Manager")
      .yearOfBirth(1990)
      .build();

  String invalidCustomerAsJSON = objectMapper.writeValueAsString(invalidCustomer);

  RequestBuilder request = MockMvcRequestBuilders.post("/customers")
      .contentType(MediaType.APPLICATION_JSON)
      .content(invalidCustomerAsJSON);

  mockMvc.perform(request)
      .andExpect(status().isBadRequest())
      .andExpect(content().contentType(MediaType.APPLICATION_JSON));
}
```

This is the most valuable test in the lesson. A unit test cannot catch this. Only an integration test sending a real HTTP request through the full stack can confirm that the `@Valid` annotation and the `@NotBlank` and `@Email` constraints from Lesson 3.17 are correctly wired and rejecting bad input before it ever reaches the service layer.

---

## Wrap-Up

- Unit tests are fast and isolated. They test your logic, with dependencies mocked.
- Integration tests are slower and realistic. They test that everything is wired together correctly.
- Mockito mocks the repository so you can test the service. MockMvc fakes the HTTP layer so you can test the controller through the full stack.
- Aim for many unit tests, fewer integration tests, very few end-to-end tests.
- In a real project these run automatically on every push. That is why speed and isolation are not academic concerns.

---

END