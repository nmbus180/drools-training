# Java Expert Guide for Drools Development: 3-Day Mastery

## Table of Contents
1. [Java Fundamentals for Drools](#java-fundamentals-for-drools)
2. [Object-Oriented Programming Essentials](#object-oriented-programming-essentials)
3. [Java Collections Framework](#java-collections-framework)
4. [Lambda Expressions & Functional Programming](#lambda-expressions--functional-programming)
5. [Date & Time API](#date--time-api)
6. [Streams API](#streams-api)
7. [Exception Handling](#exception-handling)
8. [Generics](#generics)
9. [Maven Commands](#maven-commands)
10. [Java Command Line](#java-command-line)
11. [Testing with JUnit 5](#testing-with-junit-5)
12. [Design Patterns for Drools](#design-patterns-for-drools)
13. [Best Practices](#best-practices)

---

## Java Fundamentals for Drools

### Why Java Matters for Drools

Drools rules execute Java code in the `then` clause and work with Java objects (facts). Understanding Java is critical for:
- Creating domain objects (POJOs) that rules match against
- Writing actions in rule consequences
- Managing Drools sessions and containers
- Testing rule behavior
- Integrating Drools into your claim adjudication engine

### Java Version for Your Project

Your project uses **Java 17** (LTS version) with Maven compiler configured for Java 11+ syntax.

```xml
<properties>
    <maven.compiler.release>17</maven.compiler.release>
</properties>
```

---

## Object-Oriented Programming Essentials

### 1. Classes and Objects

**Class**: Blueprint for creating objects
**Object**: Instance of a class

```java
// Class definition
public class Claim {
    // Fields (state)
    private String claimId;
    private double amount;
    private String status;

    // Constructor
    public Claim(String claimId, double amount) {
        this.claimId = claimId;
        this.amount = amount;
        this.status = "PENDING";
    }

    // Methods (behavior)
    public void approve() {
        this.status = "APPROVED";
    }

    // Getters and Setters (required for Drools pattern matching!)
    public String getClaimId() {
        return claimId;
    }

    public double getAmount() {
        return amount;
    }

    public void setAmount(double amount) {
        this.amount = amount;
    }

    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }
}

// Creating objects
Claim claim1 = new Claim("C123", 5000.0);
Claim claim2 = new Claim("C124", 15000.0);
```

**Critical for Drools**: Getters are required! Drools accesses properties through getter methods:
```drl
rule "Example"
when
    Claim(amount > 1000)  // Calls getAmount()
then
    // ...
end
```

### 2. Encapsulation

Hide internal state and require all interaction through methods:

```java
public class Passport {
    private LocalDate expiresOn;  // private field

    // Public getter
    public LocalDate getExpiresOn() {
        return expiresOn;
    }

    // Public setter with validation
    public void setExpiresOn(LocalDate expiresOn) {
        if (expiresOn == null) {
            throw new IllegalArgumentException("Expiry date cannot be null");
        }
        this.expiresOn = expiresOn;
    }

    // Derived property (no setter needed)
    public boolean isExpired() {
        return expiresOn.isBefore(LocalDate.now());
    }
}
```

### 3. Inheritance

Create specialized classes from general ones:

```java
// Base class
public class VisaApplication {
    protected int applicationId;
    protected String passportNumber;
    protected LocalDate visitStartDate;

    public int getApplicationId() {
        return applicationId;
    }

    public String getPassportNumber() {
        return passportNumber;
    }
}

// Derived class
public class FamilyVisaApplication extends VisaApplication {
    private List<String> familyMemberPassports;

    public List<String> getFamilyMemberPassports() {
        return familyMemberPassports;
    }

    public int getFamilySize() {
        return familyMemberPassports.size();
    }
}
```

Drools can match against base or derived types:
```drl
rule "Match any visa application"
when
    $app: VisaApplication()  // Matches both VisaApplication and FamilyVisaApplication
then
    // ...
end

rule "Match only family applications"
when
    $app: FamilyVisaApplication()  // Only matches FamilyVisaApplication
then
    // ...
end
```

### 4. Interfaces

Define contracts that classes must implement:

```java
public interface Validatable {
    boolean isValid();
    String getValidationMessage();
}

public class Claim implements Validatable {
    private boolean valid;
    private String validationMessage;

    @Override
    public boolean isValid() {
        return valid;
    }

    @Override
    public String getValidationMessage() {
        return validationMessage;
    }
}
```

### 5. Enums

Type-safe constants (perfect for status values):

```java
public enum Validation {
    UNKNOWN,
    PASSED,
    FAILED
}

public class Passport {
    private Validation validation = Validation.UNKNOWN;

    public Validation getValidation() {
        return validation;
    }

    public void setValidation(Validation validation) {
        this.validation = validation;
    }
}
```

Use in Drools:
```drl
rule "Process failed passports"
when
    $passport: Passport(validation == Validation.FAILED)
then
    System.out.println("Passport failed validation");
end
```

### 6. Static Members

Shared across all instances:

```java
public class ApplicationRepository {
    // Static constants
    public static final String SARAH_PASSPORT_NUMBER = "CA-SARAH-1";
    public static final String SIMON_PASSPORT_NUMBER = "CA-SIMON-2";

    // Static method (utility)
    public static List<Passport> getPassports() {
        List<Passport> passports = new ArrayList<>();
        // Create and return passports
        return passports;
    }
}

// Usage - no object needed
String passportNum = ApplicationRepository.SARAH_PASSPORT_NUMBER;
List<Passport> passports = ApplicationRepository.getPassports();
```

---

## Java Collections Framework

Collections are essential for managing multiple facts in Drools.

### 1. List (Ordered, Allows Duplicates)

```java
import java.util.ArrayList;
import java.util.List;
import java.util.Arrays;

// Create empty list
List<Claim> claims = new ArrayList<>();

// Add elements
claims.add(claim1);
claims.add(claim2);
claims.add(claim3);

// Create with initial values
List<String> statuses = Arrays.asList("PENDING", "APPROVED", "DENIED");
List<String> statuses2 = List.of("PENDING", "APPROVED", "DENIED");  // Java 9+ (immutable)

// Access by index
Claim firstClaim = claims.get(0);

// Size
int count = claims.size();

// Check if empty
if (claims.isEmpty()) {
    System.out.println("No claims");
}

// Check contains
if (claims.contains(claim1)) {
    System.out.println("Found claim");
}

// Remove
claims.remove(claim1);
claims.remove(0);  // Remove by index

// Iterate
for (Claim claim : claims) {
    System.out.println(claim.getClaimId());
}

// Clear all
claims.clear();
```

**Common in Drools:**
```java
// Insert multiple facts
List<Passport> passports = ApplicationRepository.getPassports();
for (Passport passport : passports) {
    session.insert(passport);
}

// Or use forEach
passports.forEach(session::insert);
```

### 2. Set (Unique Elements)

```java
import java.util.HashSet;
import java.util.Set;

// Create set
Set<String> uniqueStatuses = new HashSet<>();

// Add (duplicates ignored)
uniqueStatuses.add("PENDING");
uniqueStatuses.add("APPROVED");
uniqueStatuses.add("PENDING");  // Won't be added again
// Size is 2, not 3

// Check membership
if (uniqueStatuses.contains("PENDING")) {
    // ...
}

// Remove
uniqueStatuses.remove("PENDING");
```

### 3. Map (Key-Value Pairs)

```java
import java.util.HashMap;
import java.util.Map;

// Create map
Map<String, Claim> claimMap = new HashMap<>();

// Put entries
claimMap.put("C123", claim1);
claimMap.put("C124", claim2);

// Get by key
Claim claim = claimMap.get("C123");

// Check if key exists
if (claimMap.containsKey("C123")) {
    // ...
}

// Check if value exists
if (claimMap.containsValue(claim1)) {
    // ...
}

// Iterate over entries
for (Map.Entry<String, Claim> entry : claimMap.entrySet()) {
    String claimId = entry.getKey();
    Claim claim = entry.getValue();
    System.out.println(claimId + ": " + claim.getAmount());
}

// Iterate over keys
for (String claimId : claimMap.keySet()) {
    // ...
}

// Iterate over values
for (Claim claim : claimMap.values()) {
    // ...
}

// Remove
claimMap.remove("C123");

// Size
int count = claimMap.size();
```

**Common in Drools:**
```java
// Use as global for shared state
Map<String, Object> context = new HashMap<>();
session.setGlobal("context", context);
```

```drl
global java.util.Map context;

rule "Store result"
when
    $claim: Claim()
then
    context.put($claim.getClaimId(), $claim);
end
```

### 4. Collection Utility Methods

```java
import java.util.Collections;

// Sort
List<Claim> claims = new ArrayList<>();
Collections.sort(claims, (c1, c2) ->
    Double.compare(c1.getAmount(), c2.getAmount()));

// Reverse
Collections.reverse(claims);

// Shuffle
Collections.shuffle(claims);

// Find min/max
Claim maxClaim = Collections.max(claims,
    (c1, c2) -> Double.compare(c1.getAmount(), c2.getAmount()));

// Check if empty
if (claims.isEmpty()) {
    // ...
}
```

---

## Lambda Expressions & Functional Programming

Java 8+ feature that makes code more concise. Essential for modern Java and Drools integration.

### Lambda Syntax

```java
// Traditional anonymous class
Comparator<Claim> comparator = new Comparator<Claim>() {
    @Override
    public int compare(Claim c1, Claim c2) {
        return Double.compare(c1.getAmount(), c2.getAmount());
    }
};

// Lambda expression (same thing, shorter)
Comparator<Claim> comparator = (c1, c2) ->
    Double.compare(c1.getAmount(), c2.getAmount());

// Even shorter with type inference
Comparator<Claim> comparator =
    (c1, c2) -> Double.compare(c1.getAmount(), c2.getAmount());
```

**Lambda Syntax:**
```
(parameters) -> expression
(parameters) -> { statements; }
```

### Common Lambda Patterns

```java
// No parameters
Runnable r = () -> System.out.println("Hello");

// One parameter (parentheses optional)
Consumer<String> printer = s -> System.out.println(s);
Consumer<String> printer2 = (s) -> System.out.println(s);  // Same

// Multiple parameters
BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;

// Multiple statements (need braces and return)
BiFunction<Integer, Integer, Integer> calculate = (a, b) -> {
    int sum = a + b;
    int doubled = sum * 2;
    return doubled;
};
```

### Method References

Shorthand for lambdas that just call a method:

```java
// Lambda
claims.forEach(claim -> System.out.println(claim));

// Method reference (equivalent)
claims.forEach(System.out::println);

// Instance method reference
claims.forEach(session::insert);

// Static method reference
List<String> numbers = Arrays.asList("1", "2", "3");
numbers.stream()
    .map(Integer::parseInt)  // Calls Integer.parseInt(String)
    .forEach(System.out::println);

// Constructor reference
Supplier<ArrayList<String>> listSupplier = ArrayList::new;
ArrayList<String> list = listSupplier.get();
```

### Functional Interfaces

Interface with single abstract method (can be used with lambdas):

```java
// Built-in functional interfaces
Consumer<T>      // void accept(T t)
Supplier<T>      // T get()
Function<T, R>   // R apply(T t)
Predicate<T>     // boolean test(T t)
BiFunction<T, U, R>  // R apply(T t, U u)

// Examples
Consumer<Claim> processor = claim -> {
    claim.setStatus("PROCESSED");
    System.out.println("Processed: " + claim.getClaimId());
};
processor.accept(claim1);

Predicate<Claim> highValueClaim = claim -> claim.getAmount() > 10000;
if (highValueClaim.test(claim1)) {
    System.out.println("High value claim");
}

Function<Claim, String> claimToString = claim ->
    "Claim " + claim.getClaimId() + ": $" + claim.getAmount();
String description = claimToString.apply(claim1);

Supplier<LocalDate> today = LocalDate::now;
LocalDate date = today.get();
```

---

## Date & Time API

Java 8+ Date/Time API (java.time package). Your Drools project uses this extensively.

### LocalDate (Date without time)

```java
import java.time.LocalDate;
import java.time.Month;
import java.time.Period;

// Get current date
LocalDate today = LocalDate.now();

// Create specific date
LocalDate date1 = LocalDate.of(2024, 1, 15);
LocalDate date2 = LocalDate.of(2024, Month.JANUARY, 15);

// Parse from string
LocalDate date3 = LocalDate.parse("2024-01-15");

// Get components
int year = date1.getYear();          // 2024
Month month = date1.getMonth();      // JANUARY
int monthValue = date1.getMonthValue();  // 1
int day = date1.getDayOfMonth();     // 15

// Comparisons
boolean isBefore = date1.isBefore(today);
boolean isAfter = date1.isAfter(today);
boolean isEqual = date1.isEqual(today);
// or use equals()
boolean same = date1.equals(today);

// Add/subtract
LocalDate nextWeek = today.plusDays(7);
LocalDate nextMonth = today.plusMonths(1);
LocalDate nextYear = today.plusYears(1);
LocalDate lastWeek = today.minusDays(7);

// Period between dates
Period period = Period.between(date1, today);
int days = period.getDays();
int months = period.getMonths();
int years = period.getYears();

// Day of week
DayOfWeek dayOfWeek = date1.getDayOfWeek();  // MONDAY, TUESDAY, etc.
```

**In Drools:**
```java
public class Passport {
    private LocalDate expiresOn;

    public boolean isExpired() {
        return expiresOn.isBefore(LocalDate.now());
    }
}
```

```drl
rule "Expired passport"
when
    Passport(isExpired())  // Calls the method
then
    System.out.println("Passport expired");
end
```

### LocalDateTime (Date with time)

```java
import java.time.LocalDateTime;

// Current date and time
LocalDateTime now = LocalDateTime.now();

// Create specific
LocalDateTime dt = LocalDateTime.of(2024, 1, 15, 14, 30, 45);

// Parse
LocalDateTime dt2 = LocalDateTime.parse("2024-01-15T14:30:45");

// Get components
LocalDate date = dt.toLocalDate();
LocalTime time = dt.toLocalTime();
int hour = dt.getHour();
int minute = dt.getMinute();

// Manipulate
LocalDateTime later = now.plusHours(2).plusMinutes(30);
```

### LocalTime (Time without date)

```java
import java.time.LocalTime;

LocalTime now = LocalTime.now();
LocalTime time = LocalTime.of(14, 30);  // 2:30 PM
LocalTime time2 = LocalTime.of(14, 30, 45);  // with seconds
```

### ZonedDateTime (Date/time with timezone)

```java
import java.time.ZonedDateTime;
import java.time.ZoneId;

ZonedDateTime now = ZonedDateTime.now();
ZonedDateTime nyTime = ZonedDateTime.now(ZoneId.of("America/New_York"));
```

### Formatting

```java
import java.time.format.DateTimeFormatter;

LocalDate date = LocalDate.now();

// Predefined formatters
String iso = date.format(DateTimeFormatter.ISO_DATE);  // 2024-01-15

// Custom format
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("MM/dd/yyyy");
String formatted = date.format(formatter);  // 01/15/2024

// Parse with formatter
LocalDate parsed = LocalDate.parse("01/15/2024", formatter);
```

---

## Streams API

Process collections declaratively (functional style). Very powerful for data transformation.

### Basic Stream Operations

```java
import java.util.stream.Collectors;

List<Claim> claims = getAllClaims();

// Filter - keep elements matching predicate
List<Claim> highValueClaims = claims.stream()
    .filter(claim -> claim.getAmount() > 10000)
    .collect(Collectors.toList());

// Map - transform each element
List<String> claimIds = claims.stream()
    .map(Claim::getClaimId)
    .collect(Collectors.toList());

// Map to different type
List<Double> amounts = claims.stream()
    .map(Claim::getAmount)
    .collect(Collectors.toList());

// forEach - perform action on each
claims.stream()
    .forEach(claim -> System.out.println(claim.getClaimId()));

// Or with method reference
claims.stream()
    .forEach(System.out::println);

// Count
long count = claims.stream()
    .filter(claim -> claim.getStatus().equals("APPROVED"))
    .count();

// anyMatch - check if any element matches
boolean hasHighValue = claims.stream()
    .anyMatch(claim -> claim.getAmount() > 100000);

// allMatch - check if all elements match
boolean allApproved = claims.stream()
    .allMatch(claim -> claim.getStatus().equals("APPROVED"));

// noneMatch - check if no elements match
boolean noPending = claims.stream()
    .noneMatch(claim -> claim.getStatus().equals("PENDING"));

// findFirst - get first element
Optional<Claim> firstClaim = claims.stream()
    .filter(claim -> claim.getAmount() > 5000)
    .findFirst();

// findAny - get any element (useful for parallel streams)
Optional<Claim> anyClaim = claims.stream()
    .filter(claim -> claim.getAmount() > 5000)
    .findAny();
```

### Chaining Operations

```java
// Get IDs of approved high-value claims
List<String> claimIds = claims.stream()
    .filter(claim -> claim.getAmount() > 10000)
    .filter(claim -> claim.getStatus().equals("APPROVED"))
    .map(Claim::getClaimId)
    .collect(Collectors.toList());

// Get total amount of approved claims
double total = claims.stream()
    .filter(claim -> claim.getStatus().equals("APPROVED"))
    .mapToDouble(Claim::getAmount)
    .sum();

// Average
double average = claims.stream()
    .mapToDouble(Claim::getAmount)
    .average()
    .orElse(0.0);

// Max/Min
Optional<Claim> maxClaim = claims.stream()
    .max((c1, c2) -> Double.compare(c1.getAmount(), c2.getAmount()));

// Or using Comparator.comparing
Optional<Claim> maxClaim2 = claims.stream()
    .max(Comparator.comparing(Claim::getAmount));

Optional<Claim> minClaim = claims.stream()
    .min(Comparator.comparing(Claim::getAmount));
```

### Sorting

```java
// Sort by amount (ascending)
List<Claim> sorted = claims.stream()
    .sorted(Comparator.comparing(Claim::getAmount))
    .collect(Collectors.toList());

// Sort descending
List<Claim> sortedDesc = claims.stream()
    .sorted(Comparator.comparing(Claim::getAmount).reversed())
    .collect(Collectors.toList());

// Multiple sort criteria
List<Claim> sorted2 = claims.stream()
    .sorted(Comparator.comparing(Claim::getStatus)
                      .thenComparing(Claim::getAmount))
    .collect(Collectors.toList());
```

### Grouping

```java
// Group claims by status
Map<String, List<Claim>> claimsByStatus = claims.stream()
    .collect(Collectors.groupingBy(Claim::getStatus));

// Count by status
Map<String, Long> countsByStatus = claims.stream()
    .collect(Collectors.groupingBy(
        Claim::getStatus,
        Collectors.counting()
    ));

// Sum by status
Map<String, Double> amountsByStatus = claims.stream()
    .collect(Collectors.groupingBy(
        Claim::getStatus,
        Collectors.summingDouble(Claim::getAmount)
    ));
```

### Collectors

```java
// To List
List<Claim> list = claims.stream().collect(Collectors.toList());

// To Set
Set<Claim> set = claims.stream().collect(Collectors.toSet());

// To Map
Map<String, Claim> map = claims.stream()
    .collect(Collectors.toMap(
        Claim::getClaimId,  // key
        claim -> claim      // value
    ));

// Joining strings
String claimIds = claims.stream()
    .map(Claim::getClaimId)
    .collect(Collectors.joining(", "));
// Result: "C123, C124, C125"

String withPrefix = claims.stream()
    .map(Claim::getClaimId)
    .collect(Collectors.joining(", ", "Claims: [", "]"));
// Result: "Claims: [C123, C124, C125]"

// Partitioning (split into two groups)
Map<Boolean, List<Claim>> partitioned = claims.stream()
    .collect(Collectors.partitioningBy(
        claim -> claim.getAmount() > 10000
    ));
List<Claim> highValue = partitioned.get(true);
List<Claim> lowValue = partitioned.get(false);
```

### Optional

Handle potentially null values safely:

```java
Optional<Claim> optionalClaim = claims.stream()
    .filter(claim -> claim.getClaimId().equals("C123"))
    .findFirst();

// Check if present
if (optionalClaim.isPresent()) {
    Claim claim = optionalClaim.get();
    System.out.println(claim.getAmount());
}

// Or use ifPresent
optionalClaim.ifPresent(claim ->
    System.out.println(claim.getAmount())
);

// Or provide default
Claim claim = optionalClaim.orElse(new Claim("DEFAULT", 0));

// Or throw exception
Claim claim2 = optionalClaim.orElseThrow(() ->
    new RuntimeException("Claim not found")
);

// Map if present
Optional<String> claimId = optionalClaim.map(Claim::getClaimId);
```

---

## Exception Handling

### Try-Catch-Finally

```java
public void processClaim(String claimId) {
    try {
        Claim claim = claimRepository.findById(claimId);

        if (claim.getAmount() < 0) {
            throw new IllegalArgumentException("Amount cannot be negative");
        }

        claim.setStatus("PROCESSED");
        claimRepository.save(claim);

    } catch (ClaimNotFoundException e) {
        System.err.println("Claim not found: " + claimId);
        logger.error("Claim not found", e);

    } catch (IllegalArgumentException e) {
        System.err.println("Invalid claim data: " + e.getMessage());

    } catch (Exception e) {
        // Catch all other exceptions
        System.err.println("Unexpected error: " + e.getMessage());
        e.printStackTrace();

    } finally {
        // Always executes (even if exception thrown)
        System.out.println("Processing attempt completed");
    }
}
```

### Try-with-Resources

Automatically closes resources (AutoCloseable):

```java
// Traditional way
KieSession session = kContainer.newKieSession();
try {
    session.insert(claim);
    session.fireAllRules();
} finally {
    session.dispose();  // Must remember to dispose
}

// Try-with-resources (recommended)
try (KieSession session = kContainer.newKieSession()) {
    session.insert(claim);
    session.fireAllRules();
} // session.dispose() called automatically
```

### Custom Exceptions

```java
public class ClaimValidationException extends Exception {
    private String claimId;

    public ClaimValidationException(String message, String claimId) {
        super(message);
        this.claimId = claimId;
    }

    public String getClaimId() {
        return claimId;
    }
}

// Usage
public void validateClaim(Claim claim) throws ClaimValidationException {
    if (claim.getAmount() > MAX_AMOUNT) {
        throw new ClaimValidationException(
            "Amount exceeds maximum",
            claim.getClaimId()
        );
    }
}

// Calling code
try {
    validateClaim(claim);
} catch (ClaimValidationException e) {
    System.err.println("Validation failed for claim " + e.getClaimId());
}
```

### Checked vs Unchecked Exceptions

**Checked Exceptions** (must catch or declare):
- `Exception` and subclasses (except `RuntimeException`)
- Must be caught or declared with `throws`
- Example: `IOException`, `SQLException`

**Unchecked Exceptions** (optional to catch):
- `RuntimeException` and subclasses
- Don't need to be caught or declared
- Example: `NullPointerException`, `IllegalArgumentException`

```java
// Checked exception - must declare
public Claim loadClaim(String id) throws ClaimNotFoundException {
    // ...
}

// Unchecked exception - no declaration needed
public void processClaim(Claim claim) {
    if (claim == null) {
        throw new IllegalArgumentException("Claim cannot be null");
    }
}
```

---

## Generics

Type parameters for classes and methods. Provides type safety.

### Generic Classes

```java
// Generic class
public class Box<T> {
    private T content;

    public void set(T content) {
        this.content = content;
    }

    public T get() {
        return content;
    }
}

// Usage
Box<String> stringBox = new Box<>();
stringBox.set("Hello");
String value = stringBox.get();  // No cast needed

Box<Claim> claimBox = new Box<>();
claimBox.set(claim);
Claim retrievedClaim = claimBox.get();
```

### Generic Methods

```java
public class Utils {
    // Generic method
    public static <T> T getFirst(List<T> list) {
        if (list.isEmpty()) {
            return null;
        }
        return list.get(0);
    }

    // Multiple type parameters
    public static <K, V> Map<K, V> createMap(K key, V value) {
        Map<K, V> map = new HashMap<>();
        map.put(key, value);
        return map;
    }
}

// Usage
List<String> names = Arrays.asList("Alice", "Bob");
String first = Utils.getFirst(names);

Map<String, Integer> map = Utils.createMap("age", 30);
```

### Bounded Type Parameters

```java
// Upper bound - T must extend Number
public class NumberBox<T extends Number> {
    private T number;

    public double doubleValue() {
        return number.doubleValue();  // Can call Number methods
    }
}

// Usage
NumberBox<Integer> intBox = new NumberBox<>();
NumberBox<Double> doubleBox = new NumberBox<>();
// NumberBox<String> stringBox = new NumberBox<>();  // Compile error!

// Multiple bounds
public class ComparableBox<T extends Comparable<T> & Serializable> {
    // T must implement both Comparable and Serializable
}
```

### Wildcards

```java
// Unknown type
public void printList(List<?> list) {
    for (Object item : list) {
        System.out.println(item);
    }
}

// Upper bounded wildcard - extends
public double sumNumbers(List<? extends Number> numbers) {
    double sum = 0;
    for (Number num : numbers) {
        sum += num.doubleValue();
    }
    return sum;
}
// Can call with List<Integer>, List<Double>, etc.

// Lower bounded wildcard - super
public void addIntegers(List<? super Integer> list) {
    list.add(1);
    list.add(2);
}
// Can call with List<Integer>, List<Number>, List<Object>
```

---

## Maven Commands

Maven is your build tool. Essential commands:

### Building & Compilation

```bash
# Clean (delete target directory)
mvn clean

# Compile main source code
mvn compile

# Compile main and test source code
mvn test-compile

# Clean and compile
mvn clean compile

# Run tests
mvn test

# Run specific test class
mvn test -Dtest=StatelessPassportValidationTest

# Run specific test method
mvn test -Dtest=StatelessPassportValidationTest#testStep1_recordSystemOut_correctOutput

# Skip tests
mvn compile -DskipTests
mvn install -DskipTests

# Package (create JAR)
mvn package

# Package without tests
mvn package -DskipTests

# Install to local Maven repository
mvn install

# Clean, compile, test, and package
mvn clean package
```

### Running Applications

```bash
# Run a main class
mvn exec:java -Dexec.mainClass="io.github.aasaru.drools.section03.StatelessPassportValidation"

# With arguments
mvn exec:java -Dexec.mainClass="io.github.aasaru.drools.section03.StatelessPassportValidation" -Dexec.args="3"

# Alternative (if configured in pom.xml)
mvn exec:exec
```

### Dependency Management

```bash
# Download all dependencies
mvn dependency:resolve

# Show dependency tree
mvn dependency:tree

# Show all dependencies (including transitive)
mvn dependency:list

# Update snapshots
mvn clean install -U

# Check for dependency updates
mvn versions:display-dependency-updates

# Check for plugin updates
mvn versions:display-plugin-updates
```

### Other Useful Commands

```bash
# Show effective POM (with inherited values)
mvn help:effective-pom

# Validate project
mvn validate

# Generate project site documentation
mvn site

# Show Maven version
mvn -version
mvn --version

# Verbose output
mvn -X compile  # Debug output
mvn -e compile  # Show error stack traces

# Offline mode (use only local repository)
mvn -o compile

# Continue build despite test failures
mvn install -Dmaven.test.failure.ignore=true

# Run in quiet mode
mvn -q compile
```

### Maven Wrapper (mvnw)

Your project includes Maven Wrapper - guarantees everyone uses same Maven version:

```bash
# On Windows
mvnw.cmd clean compile
mvnw.cmd test

# On Linux/Mac
./mvnw clean compile
./mvnw test
```

### Multi-Module Projects

```bash
# Build all modules
mvn clean install

# Build specific module
mvn clean install -pl module-name

# Build module and dependencies
mvn clean install -pl module-name -am

# Build without dependencies
mvn clean install -pl module-name -N
```

---

## Java Command Line

### Running Java Applications

```bash
# Basic syntax
java ClassName

# With package
java com.example.Main

# With classpath
java -cp target/classes com.example.Main

# With classpath and JAR files
java -cp "target/classes;lib/*" com.example.Main

# With memory settings
java -Xms512m -Xmx1024m com.example.Main

# Your project uses this pattern:
java -Xms1024M -Xmx1024M -cp target/classes io.github.aasaru.drools.section03.StatelessPassportValidation
```

### JVM Options

```bash
# Memory settings
-Xms512m       # Initial heap size
-Xmx2048m      # Maximum heap size
-XX:MaxPermSize=256m   # PermGen space (Java 7)
-XX:MaxMetaspaceSize=256m  # Metaspace (Java 8+)

# Garbage collection
-XX:+UseG1GC              # Use G1 collector
-XX:+PrintGCDetails       # Print GC details
-XX:+PrintGCDateStamps    # Print GC timestamps

# Debug
-Xdebug
-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005

# System properties
-Dproperty.name=value
-Dfile.encoding=UTF-8
-Duser.timezone=UTC

# Enable assertions
-ea          # Enable all assertions
-da          # Disable all assertions
```

### Checking Java Installation

```bash
# Show Java version
java -version

# Show detailed version
java -fullversion

# Show JVM version
java -version 2>&1 | head -1

# Which Java executable
where java          # Windows
which java          # Linux/Mac

# Show JAVA_HOME
echo %JAVA_HOME%    # Windows
echo $JAVA_HOME     # Linux/Mac
```

### Compiling Java

```bash
# Compile single file
javac MyClass.java

# Compile with specific Java version
javac -source 17 -target 17 MyClass.java

# Compile with classpath
javac -cp lib/* MyClass.java

# Compile to specific directory
javac -d bin src/MyClass.java

# Compile multiple files
javac src/**/*.java

# Show warnings
javac -Xlint:all MyClass.java
```

### Creating JARs

```bash
# Create JAR
jar cf myapp.jar -C target/classes .

# Create JAR with manifest
jar cfm myapp.jar manifest.txt -C target/classes .

# View JAR contents
jar tf myapp.jar

# Extract JAR
jar xf myapp.jar

# Run JAR
java -jar myapp.jar
```

---

## Testing with JUnit 5

Your project uses JUnit 5 (Jupiter). Essential for testing Drools rules.

### Basic Test Structure

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.DisplayName;
import static org.junit.jupiter.api.Assertions.*;

public class ClaimServiceTest {

    private ClaimService claimService;
    private KieContainer kieContainer;

    @BeforeEach
    void setUp() {
        // Runs before each test
        claimService = new ClaimService();
        kieContainer = KieServices.Factory.get().getKieClasspathContainer();
    }

    @AfterEach
    void tearDown() {
        // Runs after each test
        // Clean up resources if needed
    }

    @Test
    void testHighValueClaimApproval() {
        // Arrange
        Claim claim = new Claim("C123", 75000);

        // Act
        claimService.processClaim(claim);

        // Assert
        assertEquals("APPROVED", claim.getStatus());
    }

    @Test
    @DisplayName("Low value claims should be auto-approved")
    void testLowValueClaimAutoApproval() {
        Claim claim = new Claim("C124", 500);
        claimService.processClaim(claim);
        assertEquals("AUTO_APPROVED", claim.getStatus());
    }
}
```

### Assertions

```java
// Equality
assertEquals(expected, actual);
assertEquals(expected, actual, "Custom message");
assertNotEquals(value1, value2);

// Boolean
assertTrue(condition);
assertTrue(condition, "Should be true");
assertFalse(condition);

// Null checks
assertNull(object);
assertNotNull(object);

// Same object
assertSame(object1, object2);
assertNotSame(object1, object2);

// Arrays
assertArrayEquals(expectedArray, actualArray);

// Exceptions
assertThrows(IllegalArgumentException.class, () -> {
    claim.setAmount(-100);
});

Exception exception = assertThrows(ClaimException.class, () -> {
    claimService.processInvalidClaim(claim);
});
assertEquals("Invalid claim", exception.getMessage());

// Multiple assertions (all executed even if one fails)
assertAll(
    () -> assertEquals("C123", claim.getClaimId()),
    () -> assertEquals(5000, claim.getAmount()),
    () -> assertEquals("APPROVED", claim.getStatus())
);

// Timeout
assertTimeout(Duration.ofSeconds(1), () -> {
    claimService.processClaim(claim);
});
```

### AssertJ (Fluent Assertions)

Your project uses AssertJ for more readable assertions:

```java
import static org.assertj.core.api.Assertions.*;

@Test
void testWithAssertJ() {
    Claim claim = new Claim("C123", 5000);

    // Simple assertions
    assertThat(claim.getClaimId()).isEqualTo("C123");
    assertThat(claim.getAmount()).isGreaterThan(1000);
    assertThat(claim.getStatus()).isNotNull();

    // String assertions
    assertThat(claim.getClaimId())
        .startsWith("C")
        .endsWith("123")
        .hasSize(4);

    // Collection assertions
    List<String> statuses = Arrays.asList("PENDING", "APPROVED");
    assertThat(statuses)
        .hasSize(2)
        .contains("PENDING", "APPROVED")
        .doesNotContain("DENIED");

    // Exception assertions
    assertThatThrownBy(() -> claim.setAmount(-100))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("negative");

    // Object assertions
    assertThat(claim)
        .isNotNull()
        .hasFieldOrPropertyWithValue("claimId", "C123")
        .hasFieldOrPropertyWithValue("amount", 5000.0);
}
```

### Testing Drools Rules

```java
@Test
void testPassportValidationRule() {
    // Given
    KieSession session = kieContainer.newKieSession();
    Passport passport = Passport.newBuilder()
        .withPassportNumber("TEST-001")
        .withExpiresOn(LocalDate.now().minusDays(1))  // Expired
        .withUnusedVisaPages(5)
        .build();

    // When
    session.insert(passport);
    int rulesFired = session.fireAllRules();

    // Then
    assertThat(rulesFired).isGreaterThan(0);
    assertThat(passport.getValidation()).isEqualTo(Validation.FAILED);
    assertThat(passport.getCause()).contains("expired");

    session.dispose();
}
```

### Parameterized Tests

Test same logic with different inputs:

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;
import org.junit.jupiter.params.provider.CsvSource;

@ParameterizedTest
@ValueSource(doubles = {100.0, 500.0, 999.99})
void testLowValueClaims(double amount) {
    Claim claim = new Claim("C123", amount);
    claimService.processClaim(claim);
    assertEquals("AUTO_APPROVED", claim.getStatus());
}

@ParameterizedTest
@CsvSource({
    "100, AUTO_APPROVED",
    "5000, MANUAL_REVIEW",
    "50000, SENIOR_REVIEW"
})
void testClaimRouting(double amount, String expectedStatus) {
    Claim claim = new Claim("C123", amount);
    claimService.processClaim(claim);
    assertEquals(expectedStatus, claim.getStatus());
}
```

### Test Lifecycle Annotations

```java
import org.junit.jupiter.api.*;

@BeforeAll
static void initAll() {
    // Runs once before all tests in class
    System.out.println("Setting up test suite");
}

@BeforeEach
void init() {
    // Runs before each test
    System.out.println("Setting up test");
}

@Test
void test1() {
    // Test code
}

@AfterEach
void tearDown() {
    // Runs after each test
    System.out.println("Cleaning up test");
}

@AfterAll
static void tearDownAll() {
    // Runs once after all tests in class
    System.out.println("Tearing down test suite");
}
```

---

## Design Patterns for Drools

### 1. Builder Pattern

Create complex objects step-by-step. Used extensively in your training code:

```java
public class Passport {
    private String passportNumber;
    private String name;
    private LocalDate expiresOn;
    private int unusedVisaPages;

    // Private constructor
    private Passport() {
    }

    // Static factory method
    public static PassportBuilder newBuilder() {
        return new PassportBuilder();
    }

    // Inner builder class
    public static final class PassportBuilder {
        private String passportNumber;
        private String name;
        private LocalDate expiresOn;
        private int unusedVisaPages;

        private PassportBuilder() {
        }

        public PassportBuilder withPassportNumber(String passportNumber) {
            this.passportNumber = passportNumber;
            return this;  // Return this for chaining
        }

        public PassportBuilder withName(String name) {
            this.name = name;
            return this;
        }

        public PassportBuilder withExpiresOn(LocalDate expiresOn) {
            this.expiresOn = expiresOn;
            return this;
        }

        public PassportBuilder withUnusedVisaPages(int unusedVisaPages) {
            this.unusedVisaPages = unusedVisaPages;
            return this;
        }

        public Passport build() {
            Passport passport = new Passport();
            passport.passportNumber = this.passportNumber;
            passport.name = this.name;
            passport.expiresOn = this.expiresOn;
            passport.unusedVisaPages = this.unusedVisaPages;
            return passport;
        }
    }
}

// Usage - fluent and readable
Passport passport = Passport.newBuilder()
    .withPassportNumber("CA-SARAH-1")
    .withName("Sarah Murphy")
    .withExpiresOn(LocalDate.of(2025, Month.DECEMBER, 17))
    .withUnusedVisaPages(5)
    .build();
```

**Why use Builder?**
- Readable code
- Immutable objects (no setters after construction)
- Optional parameters
- Validation in build() method

### 2. Repository Pattern

Separate data access logic:

```java
public class ApplicationRepository {

    public static List<Passport> getPassports() {
        List<Passport> passports = new ArrayList<>();

        passports.add(Passport.newBuilder()
            .withPassportNumber("CA-SARAH-1")
            .withName("Sarah Murphy")
            .withExpiresOn(LocalDate.of(2025, Month.DECEMBER, 17))
            .withUnusedVisaPages(5)
            .build());

        // More passports...

        return passports;
    }

    public static List<VisaApplication> getVisaApplications() {
        // Return visa applications
        return new ArrayList<>();
    }
}

// Usage - main class doesn't know where data comes from
List<Passport> passports = ApplicationRepository.getPassports();
```

### 3. Factory Pattern

Create objects without specifying exact class:

```java
public class ValidationFactory {

    public static Validation createValidation(boolean passed, String cause) {
        if (passed) {
            return Validation.PASSED;
        } else {
            return Validation.FAILED;
        }
    }
}
```

### 4. Singleton Pattern

Ensure only one instance exists:

```java
public class DroolsEngine {
    private static DroolsEngine instance;
    private KieContainer kieContainer;

    // Private constructor
    private DroolsEngine() {
        KieServices ks = KieServices.Factory.get();
        kieContainer = ks.getKieClasspathContainer();
    }

    // Public accessor
    public static synchronized DroolsEngine getInstance() {
        if (instance == null) {
            instance = new DroolsEngine();
        }
        return instance;
    }

    public KieSession createSession(String sessionName) {
        return kieContainer.newKieSession(sessionName);
    }
}

// Usage
DroolsEngine engine = DroolsEngine.getInstance();
KieSession session = engine.createSession("mySession");
```

### 5. Strategy Pattern

Define family of algorithms, encapsulate each, make them interchangeable:

```java
public interface ClaimProcessor {
    void process(Claim claim);
}

public class AutoApprovalProcessor implements ClaimProcessor {
    @Override
    public void process(Claim claim) {
        claim.setStatus("AUTO_APPROVED");
    }
}

public class ManualReviewProcessor implements ClaimProcessor {
    @Override
    public void process(Claim claim) {
        claim.setStatus("MANUAL_REVIEW");
    }
}

public class ClaimService {
    private Map<String, ClaimProcessor> processors = new HashMap<>();

    public ClaimService() {
        processors.put("LOW", new AutoApprovalProcessor());
        processors.put("HIGH", new ManualReviewProcessor());
    }

    public void processClaim(Claim claim) {
        String category = categorize(claim);
        ClaimProcessor processor = processors.get(category);
        processor.process(claim);
    }
}
```

---

## Best Practices

### 1. Naming Conventions

```java
// Classes: PascalCase (noun)
public class ClaimProcessor
public class VisaApplication

// Interfaces: PascalCase (adjective or noun)
public interface Validatable
public interface ClaimRepository

// Methods: camelCase (verb)
public void processClaim()
public boolean isValid()
public String getClaimId()

// Variables: camelCase (noun)
private String claimId;
private double totalAmount;

// Constants: UPPER_SNAKE_CASE
public static final String DEFAULT_STATUS = "PENDING";
public static final int MAX_RETRY_COUNT = 3;

// Packages: lowercase
package io.github.aasaru.drools.domain;
```

### 2. Code Organization

```java
// Order in class:
// 1. Static fields
// 2. Instance fields
// 3. Constructors
// 4. Static methods
// 5. Instance methods
// 6. Getters/Setters
// 7. equals/hashCode/toString
// 8. Inner classes

public class Claim {
    // 1. Static fields
    public static final String PENDING = "PENDING";

    // 2. Instance fields
    private String claimId;
    private double amount;

    // 3. Constructors
    public Claim(String claimId, double amount) {
        this.claimId = claimId;
        this.amount = amount;
    }

    // 4. Static methods
    public static Claim createDefault() {
        return new Claim("DEFAULT", 0);
    }

    // 5. Instance methods
    public void approve() {
        // ...
    }

    // 6. Getters/Setters
    public String getClaimId() {
        return claimId;
    }

    // 7. equals/hashCode/toString
    @Override
    public boolean equals(Object o) {
        // ...
    }

    @Override
    public int hashCode() {
        return Objects.hash(claimId);
    }

    @Override
    public String toString() {
        return "Claim{id=" + claimId + ", amount=" + amount + "}";
    }

    // 8. Inner classes
    public static class Builder {
        // ...
    }
}
```

### 3. Immutability

Prefer immutable objects when possible:

```java
// Immutable class
public final class Passport {  // final class can't be extended
    private final String passportNumber;  // final fields
    private final String name;
    private final LocalDate expiresOn;

    public Passport(String passportNumber, String name, LocalDate expiresOn) {
        this.passportNumber = passportNumber;
        this.name = name;
        this.expiresOn = expiresOn;
    }

    // Only getters, no setters
    public String getPassportNumber() {
        return passportNumber;
    }

    public String getName() {
        return name;
    }

    public LocalDate getExpiresOn() {
        return expiresOn;
    }
}
```

### 4. Equals and HashCode

Always override both together:

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;  // Same reference
    if (o == null || getClass() != o.getClass()) return false;  // Null or different class

    Passport passport = (Passport) o;
    return Objects.equals(passportNumber, passport.passportNumber);
}

@Override
public int hashCode() {
    return Objects.hash(passportNumber);
}
```

### 5. toString()

Provide meaningful string representation:

```java
@Override
public String toString() {
    return String.format("Passport[no:%s, name:%s]", passportNumber, name);
}

// Or using string concatenation
@Override
public String toString() {
    return "Claim{id=" + claimId +
           ", amount=" + amount +
           ", status=" + status + "}";
}
```

### 6. Null Safety

```java
// Check for null
if (claim != null) {
    claim.process();
}

// Use Optional
Optional<Claim> optionalClaim = findClaim(id);
optionalClaim.ifPresent(Claim::process);

// Use Objects.requireNonNull
public void setClaim(Claim claim) {
    this.claim = Objects.requireNonNull(claim, "Claim cannot be null");
}

// Use default values
String status = claim != null ? claim.getStatus() : "UNKNOWN";

// Java 9+: Objects.requireNonNullElse
String status = Objects.requireNonNullElse(claim.getStatus(), "UNKNOWN");
```

### 7. Use Final

```java
// Final variables (can't be reassigned)
final String claimId = "C123";
final List<Claim> claims = new ArrayList<>();
claims.add(claim);  // OK - list itself not final, just reference

// Final method parameters
public void processClaim(final Claim claim) {
    // claim = new Claim();  // Compile error
    claim.setStatus("PROCESSED");  // OK
}

// Final methods (can't be overridden)
public final void validate() {
    // ...
}

// Final classes (can't be extended)
public final class ImmutableClaim {
    // ...
}
```

### 8. Resource Management

```java
// Always close resources
try (KieSession session = kContainer.newKieSession();
     BufferedReader reader = new BufferedReader(new FileReader("file.txt"))) {

    // Use resources
    session.insert(claim);
    session.fireAllRules();

    String line = reader.readLine();

} // Automatically closed

// Or manually in finally
KieSession session = null;
try {
    session = kContainer.newKieSession();
    session.insert(claim);
    session.fireAllRules();
} finally {
    if (session != null) {
        session.dispose();
    }
}
```

### 9. Logging

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class ClaimService {
    private static final Logger logger = LoggerFactory.getLogger(ClaimService.class);

    public void processClaim(Claim claim) {
        logger.info("Processing claim: {}", claim.getClaimId());

        try {
            // Process
            logger.debug("Claim amount: {}", claim.getAmount());
            claim.setStatus("APPROVED");
            logger.info("Claim approved: {}", claim.getClaimId());

        } catch (Exception e) {
            logger.error("Error processing claim: " + claim.getClaimId(), e);
        }
    }
}
```

---

## Quick Reference

### Common Java Patterns in Drools

```java
// 1. Create domain object
Passport passport = Passport.newBuilder()
    .withPassportNumber("CA-001")
    .withExpiresOn(LocalDate.now().plusYears(5))
    .build();

// 2. Get Drools container
KieServices ks = KieServices.Factory.get();
KieContainer kContainer = ks.getKieClasspathContainer();

// 3. Create session
KieSession session = kContainer.newKieSession("sessionName");

// 4. Insert facts
session.insert(passport);

// 5. Fire rules
int rulesFired = session.fireAllRules();

// 6. Dispose session
session.dispose();

// 7. Process collections
List<Passport> passports = ApplicationRepository.getPassports();
passports.forEach(session::insert);

// 8. Filter and transform
List<String> validPassportNumbers = passports.stream()
    .filter(p -> !p.isExpired())
    .map(Passport::getPassportNumber)
    .collect(Collectors.toList());
```

### Essential Shortcuts

```java
// Import static
import static java.util.Arrays.asList;
List<String> list = asList("A", "B", "C");

// Import static for assertions
import static org.junit.jupiter.api.Assertions.*;
assertEquals(expected, actual);

// Diamond operator (type inference)
List<String> list = new ArrayList<>();  // Instead of new ArrayList<String>()

// Multi-catch
try {
    // code
} catch (IOException | SQLException e) {
    // handle both
}

// String operations
String joined = String.join(", ", "A", "B", "C");  // "A, B, C"
boolean empty = str.isEmpty();
boolean blank = str.isBlank();  // Java 11+

// Collection factories (Java 9+)
List<String> list = List.of("A", "B", "C");  // Immutable
Set<String> set = Set.of("A", "B", "C");     // Immutable
Map<String, Integer> map = Map.of("A", 1, "B", 2);  // Immutable
```

---

## Day-by-Day Learning Plan

### Day 1: Foundations
✅ Classes, objects, methods
✅ Collections (List, Set, Map)
✅ Basic I/O and exceptions
✅ Maven basics (compile, test, run)
✅ JUnit basics

### Day 2: Advanced Features
✅ Lambda expressions
✅ Streams API
✅ Date/Time API
✅ Generics
✅ Design patterns (Builder, Repository)

### Day 3: Integration & Best Practices
✅ Drools-Java integration patterns
✅ Testing strategies
✅ Performance considerations
✅ Production-ready code

---

## Conclusion

This guide covers all Java concepts needed to master Drools development. Key takeaways:

1. **POJOs are facts** - Understand objects, getters, setters
2. **Collections everywhere** - List, Map, Set in every Drools app
3. **Functional style** - Lambdas and streams make code cleaner
4. **Builder pattern** - Create complex domain objects elegantly
5. **Testing is critical** - JUnit 5 for rule validation
6. **Maven is your friend** - Know the commands
7. **Modern Java features** - Use Java 8+ features liberally

Practice these concepts with your training code (section03-section08) and you'll be ready to build production claim adjudication engines with Drools! 🚀
