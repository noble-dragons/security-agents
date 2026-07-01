---
name: 'Security Testing'
description: 'Writing JUnit5/Mockito unit and integration tests that prove a Checkmarx fix closes the vulnerability.'
applyTo: '**/src/test/**/*.java'
---

# Security Testing

Authoritative for: **security tests that prove a remediation**. Every fix ships
with tests that **fail on the vulnerable code and pass on the fix**.

Stack: JUnit 5 (Jupiter), Mockito, and Spring Boot test slices / MockMvc where a
web layer is involved. Match the repo's existing test conventions.

## Principles

- **Negative + positive:** one test proves the exploit is blocked, another
  proves legitimate input still works (no over-blocking / regression).
- **Assert the security property**, not incidental behavior (e.g. that the query
  is parameterized / the payload is encoded / the path is contained).
- **Boundary cases:** encoded payloads, null/empty, oversize, unicode, `../`,
  absolute URLs, private IPs — per category.
- Keep tests deterministic and independent.

## Unit test example — path traversal (CWE-22)

```java
@Test
void rejectsTraversalOutsideBaseDir() {
    assertThrows(SecurityException.class,
        () -> service.readUserFile("../../etc/passwd"));
}

@Test
void allowsLegitimateFileWithinBaseDir() {
    assertDoesNotThrow(() -> service.readUserFile("report-2026.pdf"));
}
```

## Web-layer example — XSS encoding (CWE-79)

```java
@WebMvcTest(CommentController.class)
class CommentControllerTest {
    @Autowired MockMvc mvc;

    @Test
    void escapesScriptInReflectedInput() throws Exception {
        mvc.perform(get("/comments").param("q", "<script>alert(1)</script>"))
           .andExpect(status().isOk())
           .andExpect(content().string(not(containsString("<script>"))))
           .andExpect(content().string(containsString("&lt;script&gt;")));
    }
}
```

## SQL injection (CWE-89)

Prove injection input is treated as data, not SQL:

```java
@Test
void injectionPayloadReturnsNoRowsAndDoesNotError() {
    List<Order> result = repo.findByCustomer("' OR '1'='1");
    assertTrue(result.isEmpty()); // treated as a literal, not a predicate
}
```

## SSRF (CWE-918)

```java
@Test
void blocksRequestToPrivateAddress() {
    assertThrows(SecurityException.class,
        () -> fetcher.fetch("http://169.254.169.254/latest/meta-data/"));
}
```

## Crypto (CWE-327/330)

Assert the algorithm/parameters and that randomness varies:

```java
@Test
void usesUniqueIvPerEncryption() {
    byte[] a = crypto.encrypt("x");
    byte[] b = crypto.encrypt("x");
    assertFalse(Arrays.equals(a, b)); // random IV => different ciphertext
}
```

## Authorization (CWE-862/352)

Use Spring Security test support:

```java
@Test
@WithMockUser(roles = "USER")
void forbidsNonAdminFromAdminEndpoint() throws Exception {
    mvc.perform(post("/admin/purge").with(csrf()))
       .andExpect(status().isForbidden());
}
```

## Coverage expectation

For each remediated finding: at least one exploit-blocked test and one
legitimate-path test, referencing the `Similarity ID` in a comment for
traceability. Don't weaken assertions to make a test pass — fix the code.
