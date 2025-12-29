# Drools Expert-Level Guide: From Zero to Hero in 3 Days

## Table of Contents
1. [What is Drools?](#what-is-drools)
2. [Core Architecture & Concepts](#core-architecture--concepts)
3. [Rule Language (DRL)](#rule-language-drl)
4. [Working Memory & Fact Management](#working-memory--fact-management)
5. [Pattern Matching & Constraints](#pattern-matching--constraints)
6. [Rule Execution & Control](#rule-execution--control)
7. [Advanced Features](#advanced-features)
8. [Sessions & Knowledge Bases](#sessions--knowledge-bases)
9. [Best Practices for Production Systems](#best-practices-for-production-systems)
10. [Claim Adjudication Patterns](#claim-adjudication-patterns)
11. [Performance Optimization](#performance-optimization)
12. [Testing & Debugging](#testing--debugging)

---

## What is Drools?

**Drools** is a Business Rules Management System (BRMS) and rule engine that implements the **Rete algorithm** for efficient pattern matching. It's the core technology behind Red Hat Decision Manager.

### Why Drools for Claim Adjudication?

In claim adjudication systems, you need to:
- **Separate business logic from application code** - Policy rules change frequently
- **Handle complex decision trees** - Claims have hundreds of validation rules
- **Process rules efficiently** - Evaluate thousands of claims quickly
- **Maintain audit trails** - Know which rules fired and why
- **Enable business users** - Allow policy writers to define rules
- **Manage rule versions** - Different policy effective dates

**Key Advantage**: Instead of hard-coding `if-else` statements that require developer changes and redeployment, Drools lets you externalize rules that can be modified, versioned, and deployed independently.

### Drools vs Traditional Programming

**Traditional Approach:**
```java
if (claim.getAmount() > 10000 && claim.getType().equals("medical")) {
    if (claim.getDiagnosisCode().startsWith("Z")) {
        if (claimant.getPlan().equals("premium")) {
            claim.setStatus("APPROVED");
        } else {
            claim.setStatus("MANUAL_REVIEW");
        }
    }
}
// Hundreds more if-else blocks...
```

**Drools Approach:**
```drl
rule "Approve high-value medical claims for premium members"
when
    $claim: Claim(
        amount > 10000,
        type == "medical",
        diagnosisCode.startsWith("Z")
    )
    Claimant(
        claimId == $claim.id,
        plan == "premium"
    )
then
    modify($claim) { setStatus("APPROVED") }
end
```

**Benefits:**
- Declarative (what to do, not how)
- Self-documenting
- Business-readable
- Maintainable
- Testable in isolation

---

## Core Architecture & Concepts

### The Rete Algorithm

Drools uses the **Rete algorithm** for pattern matching, which creates a network of nodes that efficiently matches facts against rule conditions.

**Key Insight**: Instead of evaluating each rule against each fact every time, Rete builds a discrimination network that remembers partial matches and only re-evaluates what changed.

```
Facts → Working Memory → Pattern Matching (Rete) → Agenda → Rule Execution
```

### Core Components

#### 1. **Facts (Domain Objects)**
Plain Old Java Objects (POJOs) that represent your data. These are inserted into working memory.

```java
public class Passport {
    private String passportNumber;
    private LocalDate expiryDate;
    private int unusedVisaPages;

    public boolean isExpired() {
        return expiryDate.isBefore(LocalDate.now());
    }
}
```

#### 2. **Working Memory**
The data store where facts live during rule execution. Think of it as the "context" for rule evaluation.

```java
// Insert facts into working memory
session.insert(passport);
session.insert(visaApplication);
```

#### 3. **Rules**
Defined in `.drl` (Drools Rule Language) files. Each rule has:
- **LHS (Left-Hand Side)**: The `when` clause - conditions/patterns
- **RHS (Right-Hand Side)**: The `then` clause - actions

```drl
rule "Rule Name"
when
    // Conditions (patterns to match)
then
    // Actions (what to do)
end
```

#### 4. **Knowledge Base (KieBase)**
A repository of compiled rules, processes, and functions. It's immutable once built.

#### 5. **Session (KieSession)**
A runtime instance where you insert facts and fire rules. Two types:
- **Stateless**: Fire-and-forget, processes facts once
- **Stateful**: Maintains state across multiple fire cycles

#### 6. **Agenda**
A queue of activated rules waiting to be executed. Uses conflict resolution strategies to determine firing order.

---

## Rule Language (DRL)

### Basic Rule Structure

```drl
package com.example.rules;

import com.example.domain.Claim;
import com.example.domain.Policy;

// Global variables accessible in all rules
global org.slf4j.Logger logger;

// Rule definition
rule "Rule name must be unique"
    // Optional attributes
    salience 100          // Priority (higher fires first)
    agenda-group "validation"  // Group for control
    no-loop true          // Prevent infinite loops
    lock-on-active true   // Lock during rule execution
when
    // Pattern matching conditions
    $claim: Claim(amount > 1000)
    $policy: Policy(policyNumber == $claim.policyNumber)
then
    // Actions (Java code)
    logger.info("Rule fired for claim: " + $claim.getId());
    $claim.setStatus("APPROVED");
    update($claim);  // Notify working memory of change
end
```

### Rule Attributes

#### **salience** (Priority)
Controls rule execution order. Higher values execute first.

```drl
rule "High priority rule"
    salience 100
when
    // conditions
then
    // actions
end

rule "Low priority rule"
    salience 10
when
    // conditions
then
    // actions
end
```

**Use Case**: In claims, you might want validation rules to fire before approval rules.

#### **no-loop**
Prevents the same rule from firing again if it modifies facts that match its conditions.

```drl
rule "Update claim amount"
    no-loop true
when
    $claim: Claim(status == "PENDING")
then
    modify($claim) { setAmount($claim.getAmount() * 1.1) }
    // Without no-loop, this would trigger the rule again!
end
```

#### **lock-on-active**
More strict than `no-loop`. Prevents rule activation even if other rules modify matching facts.

```drl
rule "Calculate final amount"
    agenda-group "calculation"
    lock-on-active true
when
    $claim: Claim(status == "CALCULATING")
then
    // This rule won't refire even if other rules in "calculation" modify $claim
    modify($claim) { setFinalAmount($claim.getAmount() + $claim.getTax()) }
end
```

#### **agenda-group**
Groups rules for sequential execution.

```drl
rule "Validate claim"
    agenda-group "validation"
when
    $claim: Claim()
then
    // validation logic
end

rule "Approve claim"
    agenda-group "approval"
when
    $claim: Claim(validation == "PASSED")
then
    // approval logic
end
```

```java
// Control execution order
session.getAgenda().getAgendaGroup("validation").setFocus();
session.fireAllRules();
session.getAgenda().getAgendaGroup("approval").setFocus();
session.fireAllRules();
```

#### **activation-group**
Only ONE rule in the group can fire. First activated rule cancels others.

```drl
rule "Small claim"
    activation-group "claim-routing"
    salience 100
when
    Claim(amount < 1000)
then
    // Auto-approve
end

rule "Medium claim"
    activation-group "claim-routing"
    salience 50
when
    Claim(amount >= 1000, amount < 10000)
then
    // Manual review
end

rule "Large claim"
    activation-group "claim-routing"
    salience 10
when
    Claim(amount >= 10000)
then
    // Senior review
end
```

#### **duration**
Delays rule execution (requires rules to be in a separate thread).

```drl
rule "Timeout unpaid claims"
    duration 30d  // Fire 30 days after activation
when
    $claim: Claim(status == "PENDING_PAYMENT")
then
    modify($claim) { setStatus("CANCELLED") }
end
```

---

## Working Memory & Fact Management

### Inserting Facts

```java
KieSession session = kContainer.newKieSession("mySession");

// Insert facts
FactHandle passportHandle = session.insert(passport);
FactHandle applicationHandle = session.insert(application);
```

### Updating Facts (Notifying Rule Engine)

**Critical**: When you modify a fact's properties, you MUST notify the working memory!

```drl
rule "Update claim amount"
when
    $claim: Claim(status == "APPROVED")
then
    $claim.setDiscount(0.1);
    update($claim);  // or modify($claim) { setDiscount(0.1) }
end
```

**Three Ways to Update:**

1. **update()** - Generic update notification
```drl
then
    $claim.setStatus("APPROVED");
    update($claim);
end
```

2. **modify()** - Cleaner syntax with property setters
```drl
then
    modify($claim) {
        setStatus("APPROVED"),
        setApprovedDate(LocalDate.now())
    }
end
```

3. **Java API**
```java
Claim claim = ...;
claim.setStatus("APPROVED");
session.update(claimHandle, claim);
```

### Retracting Facts

```drl
rule "Remove processed claims"
when
    $claim: Claim(status == "PROCESSED")
then
    retract($claim);  // Remove from working memory
end
```

### Logical Insertion (insertLogical)

Creates a fact that exists ONLY as long as the rule that created it remains true.

```drl
rule "Create validation failure"
when
    $claim: Claim(amount > maxAmount)
then
    ValidationError error = new ValidationError("Amount exceeds maximum");
    insertLogical(error);  // Auto-removed if claim.amount becomes valid
end
```

**Use Case**: Temporary derived facts that shouldn't persist if conditions change.

---

## Pattern Matching & Constraints

### Basic Patterns

Match facts by type and properties:

```drl
when
    // Match any Claim
    Claim()

    // Match Claim with specific property
    Claim(amount > 1000)

    // Multiple constraints (AND)
    Claim(amount > 1000, status == "PENDING")

    // Bind to variable
    $claim: Claim(amount > 1000)
then
    System.out.println("Found claim: " + $claim.getId());
end
```

### Constraint Operators

```drl
when
    // Equality
    Claim(status == "APPROVED")
    Claim(status != "DENIED")

    // Comparison
    Claim(amount > 1000)
    Claim(amount >= 1000)
    Claim(amount < 5000)
    Claim(amount <= 5000)

    // String operations
    Claim(diagnosisCode.startsWith("Z"))
    Claim(diagnosisCode.endsWith("1"))
    Claim(diagnosisCode.contains("ABC"))
    Claim(diagnosisCode matches "Z\\d{3}")  // Regex

    // Collection operations
    Claim(serviceLines.size() > 0)
    Claim(serviceLines contains $serviceLine)

    // Null checks
    Claim(approvedBy == null)
    Claim(approvedBy != null)

    // Range
    Claim(amount > 1000 && amount < 5000)
    Claim(amount >= 1000 && amount <= 5000)
then
    // actions
end
```

### Property Navigation

Access nested properties:

```drl
when
    $claim: Claim(
        claimant.membership.plan == "PREMIUM",
        claimant.address.state == "CA"
    )
then
    // actions
end
```

### Binding Variables

Capture values for use in conditions or actions:

```drl
rule "Calculate discount"
when
    $claim: Claim($amt: amount, $amt > 1000)
    $policy: Policy(policyNumber == $claim.policyNumber, $discount: discountRate)
then
    double newAmount = $amt * (1 - $discount);
    modify($claim) { setFinalAmount(newAmount) }
end
```

### Joining Facts (Correlating Data)

Match multiple facts with related properties:

```drl
rule "Match claim to policy"
when
    $claim: Claim($policyNum: policyNumber)
    $policy: Policy(number == $policyNum)
then
    System.out.println("Found policy " + $policy + " for claim " + $claim);
end
```

### Conditional Elements

#### **and** (implicit)
Multiple patterns on separate lines are implicitly AND'd:

```drl
when
    $claim: Claim(amount > 1000)
    $policy: Policy(status == "ACTIVE")
then
    // Both conditions must match
end
```

#### **or**
Match if ANY pattern is true:

```drl
when
    $claim: Claim(amount > 10000) or
    $claim: Claim(type == "EMERGENCY")
then
    // Fire if amount > 10000 OR type is EMERGENCY
end
```

#### **not**
Match when pattern does NOT exist:

```drl
rule "No active policy"
when
    $claim: Claim($policyNum: policyNumber)
    not Policy(number == $policyNum, status == "ACTIVE")
then
    modify($claim) { setStatus("DENIED") }
end
```

#### **exists**
Match if at least one fact exists (don't bind it):

```drl
rule "Has any high-value claim"
when
    exists Claim(amount > 100000)
then
    System.out.println("Alert: High-value claim detected!");
end
```

#### **forall**
Universal quantifier - ALL facts matching first pattern must match second:

```drl
rule "All claims approved"
when
    forall(
        $claim: Claim(claimantId == "12345")
        Claim(this == $claim, status == "APPROVED")
    )
then
    System.out.println("All claims for claimant 12345 are approved");
end
```

### Accumulate (Aggregation)

Collect and aggregate data:

```drl
rule "Total claim amount per policy"
when
    $policy: Policy($policyNum: number)
    $total: Number(doubleValue > 10000) from accumulate(
        $claim: Claim(policyNumber == $policyNum, $amt: amount),
        sum($amt)
    )
then
    System.out.println("Policy " + $policyNum + " has total claims: " + $total);
end
```

**Accumulate Functions:**
- `sum()`, `average()`, `min()`, `max()`
- `count()`
- `collectList()`, `collectSet()`

```drl
rule "Count pending claims"
when
    $count: Number() from accumulate(
        Claim(status == "PENDING"),
        count()
    )
then
    System.out.println("Pending claims: " + $count);
end
```

### Collect (List of Matching Facts)

```drl
rule "Get all claims for processing"
when
    $claims: List() from collect(Claim(status == "PENDING"))
then
    System.out.println("Found " + $claims.size() + " pending claims");
    for (Object obj : $claims) {
        Claim claim = (Claim) obj;
        // Process each claim
    }
end
```

### From (Inline Evaluation)

Match facts from a source other than working memory:

```drl
rule "Check policy claims"
when
    $policy: Policy($claims: claims)  // claims is a List<Claim> property
    $claim: Claim(amount > 1000) from $claims
then
    System.out.println("Found high-value claim in policy's claim list");
end
```

---

## Rule Execution & Control

### Firing Rules

```java
// Fire all activated rules
int rulesFired = session.fireAllRules();

// Fire up to N rules
int rulesFired = session.fireAllRules(100);

// Fire until condition
session.fireUntilHalt();  // Fires continuously until halt() called
```

### Agenda Control

The **Agenda** is where activated rules wait to be executed.

```java
// Get agenda
Agenda agenda = session.getAgenda();

// Focus on agenda group
agenda.getAgendaGroup("validation").setFocus();
session.fireAllRules();

// Clear agenda (cancel all activations)
agenda.clear();

// Check active group
String activeGroup = agenda.getFocused().getName();
```

### Rule Flow (ruleflow-group)

Use with BPMN processes to control rule execution order:

```drl
rule "Validation rule"
    ruleflow-group "validation-phase"
when
    $claim: Claim()
then
    // validation logic
end
```

### Conflict Resolution

When multiple rules are activated, Drools uses this order to decide which fires first:

1. **Salience** (highest first)
2. **LIFO** (Last In, First Out) - most recently activated
3. **Rule loading order**

### Event Listeners

Monitor rule execution:

```java
session.addEventListener(new DefaultAgendaEventListener() {
    @Override
    public void afterMatchFired(AfterMatchFiredEvent event) {
        String ruleName = event.getMatch().getRule().getName();
        System.out.println("Rule fired: " + ruleName);
    }
});
```

---

## Advanced Features

### Functions

Define reusable functions in DRL:

```drl
package com.example.rules;

import com.example.domain.Claim;

function double calculateTax(double amount, String state) {
    if ("CA".equals(state)) {
        return amount * 0.0725;
    } else if ("TX".equals(state)) {
        return amount * 0.0625;
    }
    return amount * 0.05;
}

rule "Add tax to claim"
when
    $claim: Claim($amt: amount, $state: state)
then
    double tax = calculateTax($amt, $state);
    modify($claim) { setTax(tax) }
end
```

### Queries

Define queries to fetch facts from working memory:

```drl
query "claims over amount"
    $claim: Claim(amount > $minAmount)
end

query "approved claims for policy"(String policyNum)
    $claim: Claim(
        policyNumber == policyNum,
        status == "APPROVED"
    )
end
```

```java
// Execute query
QueryResults results = session.getQueryResults("approved claims for policy", "P12345");
for (QueryResultsRow row : results) {
    Claim claim = (Claim) row.get("$claim");
    System.out.println("Claim: " + claim.getId());
}
```

### Global Variables

Share objects across all rules:

```java
Logger logger = LoggerFactory.getLogger("rules");
session.setGlobal("logger", logger);

Map<String, Object> context = new HashMap<>();
session.setGlobal("context", context);
```

```drl
global org.slf4j.Logger logger;
global java.util.Map context;

rule "Log and store"
when
    $claim: Claim(amount > 10000)
then
    logger.info("High-value claim: {}", $claim.getId());
    context.put($claim.getId(), $claim);
end
```

**Warning**: Globals are NOT facts in working memory. Changes to globals don't trigger rules.

### Templates

Generate rules from external data:

```drl
template header
claimType
minAmount
maxAmount
status

package com.example.rules;

template "Claim routing template"

rule "Route @{row.rowNumber}"
when
    $claim: Claim(
        type == "@{claimType}",
        amount >= @{minAmount},
        amount < @{maxAmount}
    )
then
    modify($claim) { setStatus("@{status}") }
end

end template
```

### Decision Tables (Excel/CSV)

Define rules in spreadsheet format. Useful for business users:

| Condition | Condition | Action |
|-----------|-----------|--------|
| Claim Type | Amount | Status |
| MEDICAL | > 10000 | MANUAL_REVIEW |
| MEDICAL | <= 10000 | AUTO_APPROVED |
| DENTAL | > 5000 | MANUAL_REVIEW |

Drools converts this to DRL rules.

### OOPath (Object-Oriented Path)

Navigate object graphs reactively:

```drl
rule "Find expired passport in application"
when
    $app: VisaApplication( /passport[expired == true] )
then
    System.out.println("Application has expired passport");
end
```

### Rule Units

Encapsulate rules with their data sources (Drools 7+):

```java
public class ClaimValidationUnit implements RuleUnit {
    private DataSource<Claim> claims;
    private DataSource<Policy> policies;

    // getters and setters
}
```

```drl
unit ClaimValidationUnit;

rule "Validate claim"
when
    $claim: /claims[ amount > 1000 ]
    $policy: /policies[ number == $claim.policyNumber ]
then
    // validation logic
end
```

---

## Sessions & Knowledge Bases

### kmodule.xml Configuration

Located in `src/main/resources/META-INF/kmodule.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<kmodule xmlns="http://www.drools.org/xsd/kmodule">

    <!-- Knowledge Base -->
    <kbase name="claimRulesKB" packages="com.example.claims">

        <!-- Stateful Session -->
        <ksession name="claimSession" type="stateful" default="true">
            <clockType type="realtime"/>
        </ksession>

        <!-- Stateless Session -->
        <ksession name="validationSession" type="stateless"/>
    </kbase>

    <!-- Separate KB for different rule sets -->
    <kbase name="pricingRulesKB" packages="com.example.pricing">
        <ksession name="pricingSession"/>
    </kbase>

</kmodule>
```

### Building Knowledge Container

```java
// Get KieServices (singleton)
KieServices ks = KieServices.Factory.get();

// Build from classpath (uses kmodule.xml)
KieContainer kContainer = ks.getKieClasspathContainer();

// Or build programmatically
KieFileSystem kfs = ks.newKieFileSystem();
kfs.write("src/main/resources/rules/claim-rules.drl", ruleContent);
KieBuilder kBuilder = ks.newKieBuilder(kfs);
kBuilder.buildAll();
KieContainer kContainer = ks.newKieContainer(kBuilder.getKieModule().getReleaseId());
```

### Stateless vs Stateful Sessions

#### **Stateless Session**
- Fire-and-forget
- No memory of previous executions
- Good for: validation, scoring, stateless decisions

```java
StatelessKieSession session = kContainer.newStatelessKieSession("validationSession");

// Execute (automatically fires all rules)
session.execute(claim);

// Or with multiple facts
session.execute(Arrays.asList(claim, policy));
```

#### **Stateful Session**
- Maintains working memory across multiple operations
- Facts persist until retracted
- Good for: complex workflows, event processing, reasoning over time

```java
KieSession session = kContainer.newKieSession("claimSession");

// Insert facts
session.insert(claim);
session.insert(policy);

// Fire rules
session.fireAllRules();

// Insert more facts
session.insert(newClaim);
session.fireAllRules();

// Clean up
session.dispose();
```

### Session Lifecycle

```java
try (KieSession session = kContainer.newKieSession()) {
    // Session automatically disposed
    session.insert(claim);
    session.fireAllRules();
} // session.dispose() called automatically
```

### Thread Safety

- **KieBase**: Thread-safe, can be shared
- **KieSession**: NOT thread-safe, create one per thread
- Use **StatelessKieSession** for concurrent processing

```java
// Shared across threads
private final KieContainer kContainer = KieServices.Factory.get()
    .getKieClasspathContainer();

// Per-thread session
public void processClaim(Claim claim) {
    KieSession session = kContainer.newKieSession();
    try {
        session.insert(claim);
        session.fireAllRules();
    } finally {
        session.dispose();
    }
}
```

---

## Best Practices for Production Systems

### 1. Rule Organization

**Package Structure:**
```
rules/
├── common/
│   ├── validation.drl
│   └── utilities.drl
├── claims/
│   ├── medical/
│   │   ├── validation.drl
│   │   ├── pricing.drl
│   │   └── approval.drl
│   └── dental/
│       ├── validation.drl
│       └── pricing.drl
└── policies/
    └── eligibility.drl
```

**Package Declarations:**
```drl
package com.company.claims.medical.validation;
```

### 2. Naming Conventions

```drl
// Good: Descriptive, action-oriented
rule "Deny claims exceeding annual maximum"
rule "Apply copay for preventive services"
rule "Flag claims requiring medical review"

// Bad: Vague, non-descriptive
rule "Rule 1"
rule "Check amount"
rule "Do stuff"
```

### 3. Rule Documentation

```drl
rule "Apply network discount for in-network providers"
    // Purpose: Reduces claim amount by network-negotiated discount
    // Effective: 2024-01-01
    // Author: Claims Team
    // Policy Reference: POL-2024-015
when
    $claim: Claim(
        providerType == "IN_NETWORK",
        $originalAmount: amount
    )
    $contract: ProviderContract(
        providerId == $claim.providerId,
        $discountRate: discountRate
    )
then
    double discountedAmount = $originalAmount * (1 - $discountRate);
    modify($claim) {
        setAmount(discountedAmount),
        setDiscountApplied($originalAmount - discountedAmount)
    }
end
```

### 4. Error Handling

```drl
rule "Handle missing policy data"
when
    $claim: Claim($policyNum: policyNumber)
    not Policy(number == $policyNum)
then
    ClaimError error = new ClaimError(
        $claim.getId(),
        "MISSING_POLICY",
        "Policy " + $policyNum + " not found"
    );
    insert(error);
    modify($claim) { setStatus("ERROR") }
end
```

### 5. Logging & Auditing

```drl
global org.slf4j.Logger logger;

rule "Approve high-value claim"
    salience 100
when
    $claim: Claim(amount > 50000, status == "PENDING")
then
    logger.info("Rule: Approve high-value claim | Claim ID: {} | Amount: {}",
        $claim.getId(), $claim.getAmount());

    AuditEntry audit = new AuditEntry()
        .setClaimId($claim.getId())
        .setAction("APPROVED")
        .setRuleName(drools.getRule().getName())
        .setTimestamp(LocalDateTime.now());
    insert(audit);

    modify($claim) { setStatus("APPROVED") }
end
```

### 6. Performance Considerations

**Avoid:**
```drl
// Bad: Iterates all claims for each policy
rule "Expensive rule"
when
    $policy: Policy()
    $claim: Claim(policyNumber == $policy.number)
then
    // actions
end
```

**Better:**
```drl
// Good: More selective pattern
rule "Better rule"
when
    $policy: Policy($policyNum: number, status == "ACTIVE")
    $claim: Claim(
        policyNumber == $policyNum,
        amount > 1000,
        status == "PENDING"
    )
then
    // actions
end
```

### 7. Testing Rules

```java
@Test
public void testHighValueClaimApproval() {
    KieSession session = kContainer.newKieSession();

    // Arrange
    Claim claim = new Claim()
        .setId("C123")
        .setAmount(75000)
        .setStatus("PENDING");
    Policy policy = new Policy()
        .setNumber("P456")
        .setMaxBenefit(100000);

    // Act
    session.insert(claim);
    session.insert(policy);
    int rulesFired = session.fireAllRules();

    // Assert
    assertEquals("APPROVED", claim.getStatus());
    assertTrue(rulesFired > 0);

    session.dispose();
}
```

### 8. Version Management

```drl
rule "Apply 2024 pricing rules"
    // Version: 2.0
    // Effective Date: 2024-01-01
    // Supersedes: v1.5
when
    $claim: Claim(
        serviceDate >= "2024-01-01",
        serviceCode == "99213"
    )
then
    modify($claim) { setAllowedAmount(150.00) }
end
```

### 9. Configuration Externalization

Don't hardcode values:

```drl
// Bad
rule "Check amount"
when
    Claim(amount > 10000)
then
    // ...
end

// Good - use configuration
global com.example.ClaimConfig config;

rule "Check amount"
when
    Claim($amt: amount)
    eval($amt > config.getHighValueThreshold())
then
    // ...
end
```

### 10. Rule Validation

Create meta-rules to validate data:

```drl
rule "Validate required claim fields"
when
    $claim: Claim(
        claimId == null ||
        serviceDate == null ||
        amount == null
    )
then
    ClaimError error = new ClaimError(
        $claim.getId(),
        "VALIDATION_ERROR",
        "Required fields missing"
    );
    insert(error);
    modify($claim) { setStatus("INVALID") }
end
```

---

## Claim Adjudication Patterns

### Pattern 1: Multi-Stage Processing

```drl
// Stage 1: Validation
rule "Validate claim data"
    agenda-group "validation"
when
    $claim: Claim(status == "SUBMITTED")
then
    // validation logic
    modify($claim) { setStatus("VALIDATED") }
end

// Stage 2: Pricing
rule "Calculate allowed amount"
    agenda-group "pricing"
when
    $claim: Claim(status == "VALIDATED")
then
    // pricing logic
    modify($claim) { setStatus("PRICED") }
end

// Stage 3: Adjudication
rule "Adjudicate claim"
    agenda-group "adjudication"
when
    $claim: Claim(status == "PRICED")
then
    // adjudication logic
    modify($claim) { setStatus("ADJUDICATED") }
end
```

```java
// Execute stages in order
session.insert(claim);

session.getAgenda().getAgendaGroup("validation").setFocus();
session.fireAllRules();

session.getAgenda().getAgendaGroup("pricing").setFocus();
session.fireAllRules();

session.getAgenda().getAgendaGroup("adjudication").setFocus();
session.fireAllRules();
```

### Pattern 2: Eligibility Checking

```drl
rule "Check member eligibility"
when
    $claim: Claim($memberId: memberId, $serviceDate: serviceDate)
    $member: Member(id == $memberId)
    $coverage: Coverage(
        memberId == $memberId,
        effectiveDate <= $serviceDate,
        terminationDate >= $serviceDate
    )
then
    modify($claim) { setEligible(true) }
end

rule "Deny ineligible claim"
when
    $claim: Claim(eligible == false)
then
    modify($claim) {
        setStatus("DENIED"),
        setDenialReason("Member not eligible on service date")
    }
end
```

### Pattern 3: Benefit Accumulation

```drl
rule "Check annual maximum"
when
    $claim: Claim($memberId: memberId, $amount: amount)
    $total: Number(doubleValue > 0) from accumulate(
        $c: Claim(
            memberId == $memberId,
            status == "PAID",
            serviceYear == $claim.getServiceYear()
        ),
        sum($c.getAmount())
    )
    $benefit: Benefit(memberId == $memberId, $maxBenefit: annualMaximum)
    eval($total + $amount > $maxBenefit)
then
    double remainingBenefit = $maxBenefit - $total;
    if (remainingBenefit > 0) {
        modify($claim) {
            setPayableAmount(remainingBenefit),
            setStatus("PARTIALLY_APPROVED")
        }
    } else {
        modify($claim) {
            setStatus("DENIED"),
            setDenialReason("Annual maximum exceeded")
        }
    }
end
```

### Pattern 4: Provider Network Discounts

```drl
rule "Apply in-network discount"
when
    $claim: Claim(
        $providerId: providerId,
        $amount: billedAmount
    )
    $provider: Provider(
        id == $providerId,
        networkStatus == "IN_NETWORK"
    )
    $contract: ProviderContract(
        providerId == $providerId,
        $discountRate: discountRate
    )
then
    double allowedAmount = $amount * (1 - $discountRate);
    modify($claim) {
        setAllowedAmount(allowedAmount),
        setDiscountAmount($amount - allowedAmount)
    }
end

rule "No discount for out-of-network"
when
    $claim: Claim($providerId: providerId, $amount: billedAmount)
    $provider: Provider(
        id == $providerId,
        networkStatus == "OUT_OF_NETWORK"
    )
then
    modify($claim) { setAllowedAmount($amount) }
end
```

### Pattern 5: Copay and Deductible

```drl
rule "Apply copay"
when
    $claim: Claim(
        $memberId: memberId,
        serviceType == "OFFICE_VISIT",
        $allowedAmount: allowedAmount
    )
    $plan: Plan(memberId == $memberId, $copay: officeVisitCopay)
then
    double payableAmount = $allowedAmount - $copay;
    modify($claim) {
        setCopayAmount($copay),
        setPayableAmount(payableAmount)
    }
end

rule "Apply remaining deductible"
when
    $claim: Claim($memberId: memberId, $allowedAmount: allowedAmount)
    $deductible: DeductibleAccumulator(
        memberId == $memberId,
        $remaining: remainingDeductible,
        $remaining > 0
    )
then
    double deductibleToApply = Math.min($allowedAmount, $remaining);
    double payableAmount = $allowedAmount - deductibleToApply;

    modify($claim) {
        setDeductibleAmount(deductibleToApply),
        setPayableAmount(payableAmount)
    }

    modify($deductible) {
        setRemainingDeductible($remaining - deductibleToApply)
    }
end
```

### Pattern 6: Duplicate Claim Detection

```drl
rule "Detect duplicate claim"
when
    $claim1: Claim(
        $claimId: id,
        $memberId: memberId,
        $serviceDate: serviceDate,
        $providerId: providerId,
        status == "SUBMITTED"
    )
    $claim2: Claim(
        id != $claimId,
        memberId == $memberId,
        serviceDate == $serviceDate,
        providerId == $providerId,
        status == "PAID"
    )
then
    modify($claim1) {
        setStatus("DENIED"),
        setDenialReason("Duplicate of claim " + $claim2.getId())
    }
end
```

### Pattern 7: Prior Authorization

```drl
rule "Require prior authorization for high-cost procedures"
when
    $claim: Claim(
        procedureCode in ("27447", "99285"),
        $authNum: priorAuthNumber
    )
    not PriorAuth(
        authorizationNumber == $authNum,
        status == "APPROVED"
    )
then
    modify($claim) {
        setStatus("DENIED"),
        setDenialReason("Prior authorization required but not found")
    }
end
```

### Pattern 8: Coordination of Benefits (COB)

```drl
rule "Apply COB for secondary claims"
when
    $claim: Claim(
        $memberId: memberId,
        coverageType == "SECONDARY",
        $allowedAmount: allowedAmount
    )
    $primaryPayment: Payment(
        claimId == $claim.getId(),
        payerType == "PRIMARY",
        $paidAmount: amount
    )
then
    double secondaryResponsibility = $allowedAmount - $paidAmount;
    modify($claim) {
        setPayableAmount(Math.max(0, secondaryResponsibility))
    }
end
```

---

## Performance Optimization

### 1. Pattern Optimization

**Most Restrictive First**: Place most selective constraints early.

```drl
// Bad: Checks all claims first
rule "Slow rule"
when
    $claim: Claim($memberId: memberId)
    Member(id == $memberId, planType == "RARE_PLAN")
then
    // ...
end

// Good: Filters rare members first
rule "Fast rule"
when
    $member: Member(planType == "RARE_PLAN", $memberId: id)
    $claim: Claim(memberId == $memberId)
then
    // ...
end
```

### 2. Avoid eval() When Possible

```drl
// Bad: eval() is expensive
rule "Check amount"
when
    $claim: Claim($amt: amount)
    eval($amt > 1000 && $amt < 5000)
then
    // ...
end

// Good: Use inline constraints
rule "Check amount"
when
    $claim: Claim(amount > 1000, amount < 5000)
then
    // ...
end
```

### 3. Use Indexed Properties

Ensure constraint fields have getters:

```java
public class Claim {
    private String status;  // Will be indexed

    public String getStatus() {  // Required for indexing
        return status;
    }
}
```

### 4. Batch Processing

```java
// Process claims in batches
List<Claim> claims = getClaimsToProcess();
int batchSize = 100;

for (int i = 0; i < claims.size(); i += batchSize) {
    KieSession session = kContainer.newKieSession();
    try {
        List<Claim> batch = claims.subList(i,
            Math.min(i + batchSize, claims.size()));

        batch.forEach(session::insert);
        session.fireAllRules();

        // Process results
    } finally {
        session.dispose();
    }
}
```

### 5. Session Pooling

```java
public class SessionPool {
    private final BlockingQueue<KieSession> pool;
    private final KieContainer kContainer;

    public SessionPool(KieContainer kContainer, int size) {
        this.kContainer = kContainer;
        this.pool = new LinkedBlockingQueue<>(size);
        for (int i = 0; i < size; i++) {
            pool.offer(kContainer.newKieSession());
        }
    }

    public KieSession borrowSession() throws InterruptedException {
        return pool.take();
    }

    public void returnSession(KieSession session) {
        pool.offer(session);
    }
}
```

### 6. Rule Compilation

```java
// Pre-compile rules at startup
KieServices ks = KieServices.Factory.get();
KieContainer kContainer = ks.getKieClasspathContainer();

// Verify compilation
Results results = kContainer.verify();
if (results.hasMessages(Message.Level.ERROR)) {
    throw new RuntimeException("Rule compilation errors: " + results.getMessages());
}
```

### 7. Memory Management

```java
// Dispose sessions promptly
try (KieSession session = kContainer.newKieSession()) {
    session.insert(claim);
    session.fireAllRules();
} // Automatically disposed

// Retract facts no longer needed
session.retract(factHandle);

// Clear agenda if needed
session.getAgenda().clear();
```

---

## Testing & Debugging

### Unit Testing Rules

```java
public class ClaimRulesTest {
    private KieContainer kContainer;

    @Before
    public void setup() {
        KieServices ks = KieServices.Factory.get();
        kContainer = ks.getKieClasspathContainer();
    }

    @Test
    public void testHighValueClaimApproval() {
        KieSession session = kContainer.newKieSession();

        // Given
        Claim claim = new Claim()
            .setAmount(75000)
            .setStatus("PENDING");

        // When
        session.insert(claim);
        int rulesFired = session.fireAllRules();

        // Then
        assertEquals("APPROVED", claim.getStatus());
        assertEquals(1, rulesFired);

        session.dispose();
    }

    @Test
    public void testLowValueClaimAutoApproval() {
        KieSession session = kContainer.newKieSession();

        Claim claim = new Claim()
            .setAmount(500)
            .setStatus("PENDING");

        session.insert(claim);
        session.fireAllRules();

        assertEquals("AUTO_APPROVED", claim.getStatus());

        session.dispose();
    }
}
```

### Testing with Event Listeners

```java
@Test
public void testRuleExecutionOrder() {
    KieSession session = kContainer.newKieSession();

    List<String> firedRules = new ArrayList<>();

    session.addEventListener(new DefaultAgendaEventListener() {
        @Override
        public void afterMatchFired(AfterMatchFiredEvent event) {
            firedRules.add(event.getMatch().getRule().getName());
        }
    });

    session.insert(claim);
    session.fireAllRules();

    // Verify execution order
    assertEquals("Validate claim", firedRules.get(0));
    assertEquals("Price claim", firedRules.get(1));
    assertEquals("Approve claim", firedRules.get(2));

    session.dispose();
}
```

### Debugging Rules

#### 1. **Enable Logging**

```xml
<!-- logback.xml -->
<configuration>
    <logger name="org.drools" level="DEBUG"/>
    <logger name="org.kie" level="DEBUG"/>
</configuration>
```

#### 2. **Debug Output in Rules**

```drl
rule "Debug rule"
when
    $claim: Claim($id: id, $amt: amount)
then
    System.out.println("Rule fired for claim: " + $id + ", amount: " + $amt);
    System.out.println("Current status: " + $claim.getStatus());
end
```

#### 3. **Agenda Inspection**

```java
Agenda agenda = session.getAgenda();
for (Match match : agenda.getMatches()) {
    System.out.println("Activated rule: " + match.getRule().getName());
}
```

#### 4. **Working Memory Dump**

```java
Collection<FactHandle> handles = session.getFactHandles();
for (FactHandle handle : handles) {
    Object fact = session.getObject(handle);
    System.out.println("Fact in working memory: " + fact);
}
```

#### 5. **Rule Coverage**

```java
@Test
public void testRuleCoverage() {
    KieSession session = kContainer.newKieSession();

    Set<String> firedRules = new HashSet<>();
    session.addEventListener(new DefaultAgendaEventListener() {
        @Override
        public void afterMatchFired(AfterMatchFiredEvent event) {
            firedRules.add(event.getMatch().getRule().getName());
        }
    });

    // Insert various test cases
    session.insert(testCase1);
    session.insert(testCase2);
    session.fireAllRules();

    // Verify all expected rules fired
    assertTrue(firedRules.contains("Rule 1"));
    assertTrue(firedRules.contains("Rule 2"));

    session.dispose();
}
```

### Integration Testing

```java
@SpringBootTest
public class ClaimProcessingIntegrationTest {

    @Autowired
    private ClaimService claimService;

    @Test
    public void testEndToEndClaimProcessing() {
        // Given
        Claim claim = createTestClaim();

        // When
        ClaimResult result = claimService.processClaim(claim);

        // Then
        assertEquals("APPROVED", result.getStatus());
        assertNotNull(result.getPaymentAmount());
        assertTrue(result.getPaymentAmount() > 0);
    }
}
```

---

## Common Patterns & Anti-Patterns

### ✅ Good Patterns

#### 1. **Separation of Concerns**
```drl
// Separate rules for different responsibilities
rule "Validate" agenda-group "validation" ...
rule "Price" agenda-group "pricing" ...
rule "Approve" agenda-group "approval" ...
```

#### 2. **Immutable Facts**
```java
// Create new objects instead of modifying
then
    Claim updatedClaim = $claim.withStatus("APPROVED");
    retract($claim);
    insert(updatedClaim);
end
```

#### 3. **Status-Driven Processing**
```drl
// Use status to control flow
rule "Step 1"
when
    $claim: Claim(status == "NEW")
then
    modify($claim) { setStatus("VALIDATED") }
end

rule "Step 2"
when
    $claim: Claim(status == "VALIDATED")
then
    modify($claim) { setStatus("PRICED") }
end
```

### ❌ Anti-Patterns

#### 1. **God Rules**
```drl
// Bad: One rule does everything
rule "Process everything"
when
    $claim: Claim()
then
    // 500 lines of code doing validation, pricing, approval...
end
```

**Fix**: Break into multiple focused rules.

#### 2. **Infinite Loops**
```drl
// Bad: Modifies its own condition without no-loop
rule "Dangerous rule"
when
    $claim: Claim(amount > 1000)
then
    modify($claim) { setAmount($claim.getAmount() * 1.1) }
    // Will fire forever!
end
```

**Fix**: Add `no-loop true` or use different status transitions.

#### 3. **Magic Numbers**
```drl
// Bad: Hardcoded values
rule "Check amount"
when
    Claim(amount > 10000)
then
    // ...
end
```

**Fix**: Use globals or configuration.

#### 4. **eval() Overuse**
```drl
// Bad: Complex logic in eval
rule "Complex check"
when
    eval(someComplexCalculation())
then
    // ...
end
```

**Fix**: Use functions or pre-compute in Java.

---

## Quick Reference Cheat Sheet

### Rule Structure
```drl
rule "Name"
    salience 100
    no-loop true
    agenda-group "group"
when
    Pattern(constraints)
then
    // actions
end
```

### Common Patterns
```drl
// Match fact
Claim()
Claim(amount > 1000)
$claim: Claim(amount > 1000)

// Join facts
$claim: Claim($policyNum: policyNumber)
Policy(number == $policyNum)

// NOT
not Claim(status == "DENIED")

// EXISTS
exists Claim(amount > 100000)

// OR
Claim(amount > 10000) or Claim(type == "URGENT")

// Accumulate
$total: Number() from accumulate(
    Claim($amt: amount),
    sum($amt)
)
```

### Actions
```drl
// Modify
modify($claim) { setStatus("APPROVED") }

// Insert
insert(new Visa())

// Retract
retract($claim)

// Insert logical
insertLogical(new ValidationError())
```

### Java API
```java
// Create session
KieSession session = kContainer.newKieSession();

// Insert facts
session.insert(claim);

// Fire rules
session.fireAllRules();

// Dispose
session.dispose();
```

---

## Next Steps

### Day 1: Fundamentals
- ✅ Understand Rete algorithm and architecture
- ✅ Learn rule syntax and basic patterns
- ✅ Practice with stateless sessions
- ✅ Write simple validation rules

### Day 2: Advanced Features
- ✅ Master accumulate, collect, from
- ✅ Understand agenda control
- ✅ Work with stateful sessions
- ✅ Implement complex rule flows

### Day 3: Production Readiness
- ✅ Performance optimization techniques
- ✅ Testing strategies
- ✅ Debugging approaches
- ✅ Claim adjudication patterns

### Resources
- Official Docs: https://docs.drools.org/
- Your training code: Study section03 → section08 progressively
- Practice: Write rules for your actual claim scenarios

---

## Conclusion

Drools separates **business logic** (rules) from **technical code** (Java), enabling:
- Faster policy changes
- Business user involvement
- Better testability
- Maintainable systems

**Key Success Factors:**
1. Think declaratively ("what" not "how")
2. Keep rules focused and single-purpose
3. Use agenda control for complex flows
4. Test thoroughly
5. Monitor performance

You're now equipped with expert-level Drools knowledge. Apply these concepts to your training code and real claim adjudication scenarios. Good luck! 🚀
