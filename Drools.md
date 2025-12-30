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

## Rule Organization & Hierarchy

Understanding how Drools organizes rules is critical for building scalable, maintainable systems. This section explains the hierarchy and relationships between different Drools artifacts.

### The Complete Hierarchy

```
Project (Maven/Gradle)
│
├── kmodule.xml (Configuration)
│   └── Defines Knowledge Bases and Sessions
│
├── Knowledge Base (KieBase)
│   ├── Contains compiled rules from multiple packages
│   ├── Immutable once built
│   └── Multiple sessions can share one KieBase
│
├── Packages (Logical Grouping)
│   ├── Organize rules by domain/functionality
│   ├── Defined by 'package' declaration in DRL files
│   └── Example: com.company.claims.validation
│
├── DRL Files (.drl)
│   ├── Physical files containing rule definitions
│   ├── One file can contain multiple rules
│   ├── Located in src/main/resources
│   └── Belong to one package per file
│
├── Rules (Individual Rule Definitions)
│   ├── Smallest unit of business logic
│   ├── Defined with 'rule "name" ... end'
│   ├── Can belong to rule groups
│   └── Multiple rules per DRL file
│
├── Rule Groups (Execution Control)
│   ├── agenda-group: Sequential execution groups
│   ├── activation-group: Only one rule fires
│   ├── ruleflow-group: Controlled by process flow
│   └── Rules in same DRL file can belong to different groups
│
└── Other Rule Formats
    ├── Templates (.drl with placeholders)
    ├── Decision Tables (.xls, .xlsx, .csv)
    ├── Rule Units (Drools 7+)
    └── DMN (Decision Model Notation)
```

### 1. DRL Files (Drools Rule Language)

**Definition**: Physical `.drl` files that contain rule definitions written in Drools Rule Language.

**Location**: `src/main/resources/` (follows package structure)

**Structure**:
```drl
// Package declaration (required)
package com.company.claims.validation;

// Imports
import com.company.domain.Claim;
import com.company.domain.Policy;
import java.time.LocalDate;

// Global variables
global org.slf4j.Logger logger;

// Functions (reusable code)
function double calculateTax(double amount) {
    return amount * 0.05;
}

// Rule 1
rule "Validate claim amount"
when
    $claim: Claim(amount <= 0)
then
    logger.warn("Invalid amount for claim: {}", $claim.getId());
end

// Rule 2
rule "Check policy expiration"
when
    $claim: Claim($policyNum: policyNumber)
    $policy: Policy(number == $policyNum, expirationDate < LocalDate.now())
then
    $claim.setStatus("DENIED");
end

// More rules...
```

**Key Points**:
- One package per file (all rules in file belong to same package)
- Can have multiple rules per file
- Naming convention: Use descriptive names like `ClaimValidation.drl`, `PricingRules.drl`
- Rules in same file are NOT automatically related - grouping is logical only

### 2. Packages (Logical Organization)

**Definition**: Logical namespace for grouping related rules, similar to Java packages.

**Purpose**:
- Organize rules by business domain
- Enable selective loading (include/exclude packages)
- Prevent naming conflicts

**Declaration**:
```drl
package com.company.claims.medical.validation;
```

**Example Package Structure**:
```
src/main/resources/
└── com/
    └── company/
        └── claims/
            ├── common/
            │   ├── CommonValidation.drl        # package: com.company.claims.common
            │   └── DateUtilities.drl
            │
            ├── medical/
            │   ├── validation/
            │   │   ├── MedicalValidation.drl   # package: com.company.claims.medical.validation
            │   │   └── DiagnosisRules.drl
            │   └── pricing/
            │       └── MedicalPricing.drl      # package: com.company.claims.medical.pricing
            │
            └── dental/
                ├── validation/
                │   └── DentalValidation.drl    # package: com.company.claims.dental.validation
                └── pricing/
                    └── DentalPricing.drl
```

**Loading Specific Packages**:
```xml
<!-- kmodule.xml -->
<kbase name="medicalRulesKB" packages="com.company.claims.medical">
    <!-- Only loads rules from com.company.claims.medical.* packages -->
    <ksession name="medicalSession"/>
</kbase>

<kbase name="dentalRulesKB" packages="com.company.claims.dental">
    <!-- Only loads rules from com.company.claims.dental.* packages -->
    <ksession name="dentalSession"/>
</kbase>

<kbase name="allRulesKB" packages="com.company.claims">
    <!-- Loads ALL rules from com.company.claims.* (medical + dental + common) -->
    <ksession name="allClaimsSession"/>
</kbase>
```

**Best Practices**:
- Mirror business domain structure
- Use hierarchical naming: `domain.subdomain.function`
- Keep package names consistent with folder structure
- Don't make packages too granular (balance between organization and complexity)

### 3. Rules (Individual Business Logic)

**Definition**: The atomic unit of business logic in Drools. Each rule has a unique name within its package.

**Anatomy**:
```drl
rule "Unique Rule Name Within Package"
    // Attributes (optional)
    salience 100
    agenda-group "validation"
    no-loop true
when
    // LHS - Conditions (what to match)
    Pattern1()
    Pattern2()
then
    // RHS - Actions (what to do)
    System.out.println("Rule fired");
end
```

**Rule Identity**:
- **Name**: Must be unique within the package
- **Package + Name = Fully Qualified Rule Name**
- Different packages can have rules with same name

```drl
// File: com/company/claims/medical/Validation.drl
package com.company.claims.medical;
rule "Validate Amount"  // Full name: com.company.claims.medical.Validate Amount
when
    // medical validation logic
then
    // ...
end

// File: com/company/claims/dental/Validation.drl
package com.company.claims.dental;
rule "Validate Amount"  // Full name: com.company.claims.dental.Validate Amount
when
    // dental validation logic (different from medical)
then
    // ...
end
```

**Relationships Between Rules**:
- Rules are independent by default
- Can communicate through:
  - **Working Memory**: One rule inserts/modifies facts that trigger other rules
  - **Rule Groups**: Organize execution order
  - **Salience**: Control priority
  - **Agenda**: Shared execution queue

### 4. Rule Groups (Execution Control)

Rule groups control WHEN and HOW rules execute, not WHERE they're defined.

#### **A. agenda-group** (Sequential Execution)

**Purpose**: Group rules for sequential, controlled execution.

**Behavior**:
- Rules in an agenda group only fire when that group has "focus"
- Use to implement multi-stage processing

```drl
// In ValidationRules.drl
rule "Check required fields"
    agenda-group "validation"  // Stage 1
when
    $claim: Claim(claimId == null || amount == null)
then
    $claim.setStatus("INVALID");
end

// In PricingRules.drl
rule "Calculate price"
    agenda-group "pricing"  // Stage 2
when
    $claim: Claim(status == "VALID")
then
    $claim.setPrice(calculatePrice($claim));
end

// In ApprovalRules.drl
rule "Approve claim"
    agenda-group "approval"  // Stage 3
when
    $claim: Claim(status == "PRICED", price < 10000)
then
    $claim.setStatus("APPROVED");
end
```

**Execution**:
```java
KieSession session = kContainer.newKieSession();
session.insert(claim);

// Execute stages in order
session.getAgenda().getAgendaGroup("validation").setFocus();
session.fireAllRules();

session.getAgenda().getAgendaGroup("pricing").setFocus();
session.fireAllRules();

session.getAgenda().getAgendaGroup("approval").setFocus();
session.fireAllRules();
```

**Key Point**: Rules in different DRL files and packages can belong to same agenda-group.

#### **B. activation-group** (Mutual Exclusion)

**Purpose**: Only ONE rule in the group can fire. First activated rule cancels others.

**Use Case**: Decision routing where only one path should execute.

```drl
// Multiple files, same activation-group
rule "Route small claims"
    activation-group "claim-routing"
    salience 100
when
    Claim(amount < 1000)
then
    // Auto-approve
end

rule "Route medium claims"
    activation-group "claim-routing"
    salience 50
when
    Claim(amount >= 1000, amount < 10000)
then
    // Manual review
end

rule "Route large claims"
    activation-group "claim-routing"
    salience 10
when
    Claim(amount >= 10000)
then
    // Senior review
end
```

**Behavior**: Once one rule fires, all other rules in the activation-group are cancelled.

#### **C. ruleflow-group** (Process-Driven)

**Purpose**: Rules activated by BPMN process flows.

**Use Case**: Complex workflows with branching, loops, and conditional execution.

```drl
rule "Validation rule"
    ruleflow-group "validation-phase"
when
    $claim: Claim()
then
    // Validation logic
end
```

**Controlled by**: BPMN process definition, not direct Java code.

#### **Comparison Table**

| Aspect | agenda-group | activation-group | ruleflow-group |
|--------|-------------|------------------|----------------|
| **Execution** | All matching rules in group | Only first activated rule | Controlled by BPMN process |
| **Control** | Java code sets focus | Automatic (first wins) | Process engine |
| **Use Case** | Sequential stages | Either/or decisions | Complex workflows |
| **Multiple Fires** | Yes, all rules | No, only one | Depends on process |

### 5. Rule Sets vs Rule Groups

**Common Confusion**: These terms are often used interchangeably but have different meanings:

#### **Rule Set** (Informal Concept)
- **Definition**: Informal term for a collection of related rules
- **Not a Drools construct**: No special syntax or behavior
- **Usage**: "The medical validation rule set" = all rules for medical validation
- **Implementation**: Typically a package or multiple DRL files

#### **Rule Group** (Formal Construct)
- **Definition**: Drools-specific mechanism (agenda-group, activation-group, etc.)
- **Has behavior**: Controls execution order and mutual exclusion
- **Syntax**: Defined as rule attribute

**Example**:
```drl
// "Claim validation rule set" = informal collection
package com.company.claims.validation;

// Rule 1 belongs to "validation" agenda-group (formal)
rule "Check amount"
    agenda-group "validation"
when
    Claim(amount <= 0)
then
    // ...
end

// Rule 2 belongs to same agenda-group
rule "Check dates"
    agenda-group "validation"
when
    Claim(serviceDate == null)
then
    // ...
end
```

### 6. Templates (Dynamic Rule Generation)

**Definition**: DRL files with placeholders that generate multiple rules from external data.

**Purpose**:
- Create many similar rules with different parameters
- Allow business users to configure rules via spreadsheets/databases
- Avoid code duplication

**Template Structure**:
```drl
template header
claimType
minAmount
maxAmount
approvalStatus

package com.company.claims.templates;

template "Claim Approval Template"

rule "Approve @{claimType} claims between @{minAmount} and @{maxAmount}"
when
    $claim: Claim(
        type == "@{claimType}",
        amount >= @{minAmount},
        amount < @{maxAmount}
    )
then
    modify($claim) { setStatus("@{approvalStatus}") }
end

end template
```

**Data Source (CSV, Excel, Database)**:
```csv
claimType,minAmount,maxAmount,approvalStatus
MEDICAL,0,1000,AUTO_APPROVED
MEDICAL,1000,10000,MANUAL_REVIEW
MEDICAL,10000,999999,SENIOR_REVIEW
DENTAL,0,500,AUTO_APPROVED
DENTAL,500,999999,MANUAL_REVIEW
```

**Generated Rules** (3 rules from template + 2 data rows = 5 rules):
```drl
rule "Approve MEDICAL claims between 0 and 1000"
when
    $claim: Claim(type == "MEDICAL", amount >= 0, amount < 1000)
then
    modify($claim) { setStatus("AUTO_APPROVED") }
end

rule "Approve MEDICAL claims between 1000 and 10000"
when
    $claim: Claim(type == "MEDICAL", amount >= 1000, amount < 10000)
then
    modify($claim) { setStatus("MANUAL_REVIEW") }
end

// ... 3 more rules
```

**Key Concept: 1 Template = N Rules**

⚠️ **Important**: A template does NOT define a single rule. Instead:
- **1 Template + N Data Rows = N Generated Rules**
- Each data row creates one complete rule from the template pattern
- Think of it like a cookie cutter: one cutter (template) makes many cookies (rules) from different dough (data)

**Example Breakdown**:
```
Template: 1 rule pattern
Data Rows: 5 rows
Result: 5 individual rules (not 1 rule!)
```

**Why Use Templates?**
- **Same logic, different parameters** - Same rule structure, different values
- **Many similar rules** - Would require 100+ manually written rules otherwise
- **Business-configurable** - Business users maintain data, not code
- **Maintainability** - Change template once, all generated rules update

**When NOT to Use Templates:**
- Rules have completely different logic
- Only need a few rules (overhead not worth it)
- Business logic too complex for parameterization

**Relationship to Other Concepts**:
- Templates generate regular rules (indistinguishable from hand-written DRL rules)
- Generated rules belong to specified package
- Can assign generated rules to rule groups
- One template can generate hundreds of rules
- Multiple templates needed for different rule patterns

### Template Reusability: Same Template vs New Template

**Question**: If you created a template for Singapore X airline delay compensation, should you reuse it for Lufthansa in Germany or create a new one?

**Answer**: **Reuse the SAME template** if the business logic is identical - just feed it different data!

#### Example: Single Template for Multiple Airlines

**One Template for ALL Airlines:**
```drl
template header
airline
country
delayMinutes
compensationAmount

package com.company.travel.compensation;

template "Flight Delay Compensation"

rule "Compensate @{airline} passengers for @{delayMinutes} min delay"
when
    $flight: Flight(
        airline == "@{airline}",
        country == "@{country}",
        delayMinutes >= @{delayMinutes}
    )
    $passenger: Passenger(flightId == $flight.id)
then
    modify($passenger) {
        setCompensation(@{compensationAmount})
    }
end

end template
```

**Data for BOTH Airlines (CSV):**
```csv
airline,country,delayMinutes,compensationAmount
Singapore X,Singapore,60,100
Singapore X,Singapore,120,200
Singapore X,Singapore,180,300
Lufthansa,Germany,60,250
Lufthansa,Germany,120,400
Lufthansa,Germany,180,600
Air France,France,60,250
British Airways,UK,120,350
```

**Result**: 8 rules generated from ONE template!
- 3 rules for Singapore X
- 3 rules for Lufthansa
- 1 rule for Air France
- 1 rule for British Airways

#### Decision Matrix: Same Template vs New Template

| Scenario | Use Same Template | Create New Template |
|----------|-------------------|---------------------|
| Same logic, different values | ✅ Yes | ❌ No |
| Different compensation amounts | ✅ Yes | ❌ No |
| Different airlines/countries | ✅ Yes | ❌ No |
| Different regulations (EU vs Asia) | ❌ No | ✅ Yes |
| Different conditions (weather exemptions) | ❌ No | ✅ Yes |
| Different actions (voucher vs cash) | ❌ No | ✅ Yes |

#### ✅ Use SAME Template When:

- **Same business logic** (delay → compensation)
- **Same fields/conditions** (airline, delay minutes, passenger)
- **Only values differ** (compensation amounts, airline names, countries)
- **Want centralized maintenance** (change template once, affects all rules)
- **Rules scale horizontally** (adding more airlines, not more complexity)

#### ❌ Create NEW Template When:

- **Different logic** (EU requires reason codes, Singapore doesn't)
- **Different conditions** (Germany has weather exemptions, Singapore doesn't)
- **Different actions** (EU gives cash, Asia gives vouchers)
- **Different regulations** (EU 261/2004 vs Singapore Consumer Protection Act)
- **Complex variations** (distance-based vs flat-rate compensation)

#### Real-World Example: When to Split Templates

**Scenario**: EU has different compensation rules than Asia

**Option 1: Different Templates** (Recommended when logic differs)

```drl
// Template 1: EU Compensation (complex EU 261/2004 rules)
template header
airline
minDelayMinutes
minDistance
shortHaulAmount
longHaulAmount

package com.company.travel.eu;

template "EU Flight Delay Compensation"

rule "EU compensation for @{airline}"
when
    $flight: Flight(
        airline == "@{airline}",
        departureCountry in ("DE", "FR", "IT", "ES", "UK"),
        delayMinutes >= @{minDelayMinutes},
        cancellationReason not in ("weather", "security", "air_traffic"),
        distance >= @{minDistance}
    )
    $passenger: Passenger(flightId == $flight.id)
then
    double amount = $flight.getDistance() < 1500
        ? @{shortHaulAmount}
        : @{longHaulAmount};
    modify($passenger) { setCompensation(amount) }
end

end template

// Template 2: Singapore Compensation (simpler flat-rate rules)
template header
airline
delayMinutes
compensationAmount

package com.company.travel.asia;

template "Singapore Flight Delay Compensation"

rule "Singapore compensation for @{airline}"
when
    $flight: Flight(
        airline == "@{airline}",
        departureCountry == "SG",
        delayMinutes >= @{delayMinutes}
    )
    $passenger: Passenger(flightId == $flight.id)
then
    modify($passenger) { setCompensation(@{compensationAmount}) }
end

end template
```

**Data for EU Template:**
```csv
airline,minDelayMinutes,minDistance,shortHaulAmount,longHaulAmount
Lufthansa,180,0,250,600
Air France,180,0,250,600
British Airways,180,0,220,550
```

**Data for Singapore Template:**
```csv
airline,delayMinutes,compensationAmount
Singapore X,60,100
Singapore X,120,200
Air Asia,60,50
```

**Option 2: Same Template** (If logic is truly identical)

```drl
// One universal template - only if ALL rules work the same way
template "Global Flight Delay Compensation"

rule "Compensate @{airline} - @{country} for @{delayMinutes}min delay"
when
    $flight: Flight(
        airline == "@{airline}",
        country == "@{country}",
        delayMinutes >= @{delayMinutes}
    )
    $passenger: Passenger(flightId == $flight.id)
then
    modify($passenger) { setCompensation(@{compensationAmount}) }
end

end template
```

#### Best Practice: Start Simple, Split When Needed

1. **Start with ONE template** for Singapore X
2. **Try adding Lufthansa data** to same template
3. **If it works cleanly** → Keep using same template ✅
4. **If you need complex if/else** in template → Split into separate templates ❌

**Rule of Thumb:**
- **Same template** = Same recipe, different ingredients (salt, sugar, spices)
- **Different template** = Different recipes entirely (cake vs pasta)

#### Summary: Template Reusability

```
Same Business Logic + Different Data = Reuse Same Template
Different Business Logic = Create New Template
```

**For your use case**: Singapore X → Lufthansa
- If both follow **same delay rules** → Add Lufthansa rows to existing template ✅
- If Germany has **different regulations** → Create separate EU template ❌

### 7. Decision Tables (Business-Friendly Rules)

**Definition**: Spreadsheet-based rule definitions that compile to DRL rules.

**Format**: Excel (.xls, .xlsx) or CSV

**Example Decision Table**:

| RuleSet | RuleTable ClaimApproval |||
|---------|----------|---------|---------|
| CONDITION | CONDITION | ACTION |
| Claim Type | Amount Range | Status |
| type == "$param" | amount >= $1 && amount < $2 | setStatus("$param"); |
| MEDICAL | 0, 1000 | AUTO_APPROVED |
| MEDICAL | 1000, 10000 | MANUAL_REVIEW |
| DENTAL | 0, 500 | AUTO_APPROVED |

**Compiles To**:
```drl
rule "ClaimApproval_10"
when
    $claim: Claim(type == "MEDICAL", amount >= 0, amount < 1000)
then
    $claim.setStatus("AUTO_APPROVED");
end

rule "ClaimApproval_11"
when
    $claim: Claim(type == "MEDICAL", amount >= 1000, amount < 10000)
then
    $claim.setStatus("MANUAL_REVIEW");
end

// ... more rules
```

**Relationship**:
- Decision tables are another way to define rules
- Each row generates one rule
- All generated rules belong to same package
- Can specify rule attributes (salience, agenda-group, etc.) in table

### 8. Knowledge Base (KieBase)

**Definition**: Container for all compiled rules, processes, and functions. Immutable once built.

**Role in Hierarchy**:
- Top-level runtime container
- Contains rules from one or more packages
- Multiple sessions can share one KieBase
- Built from DRL files, templates, and decision tables

**Configuration (kmodule.xml)**:
```xml
<kmodule xmlns="http://www.drools.org/xsd/kmodule">

    <!-- KieBase 1: Medical Claims -->
    <kbase name="medicalKB" packages="com.company.claims.medical">
        <ksession name="medicalSession"/>
    </kbase>

    <!-- KieBase 2: Dental Claims -->
    <kbase name="dentalKB" packages="com.company.claims.dental">
        <ksession name="dentalSession"/>
    </kbase>

    <!-- KieBase 3: All Claims -->
    <kbase name="allClaimsKB" packages="com.company.claims">
        <ksession name="allClaimsSession" default="true"/>
    </kbase>

</kmodule>
```

**Java Usage**:
```java
KieServices ks = KieServices.Factory.get();
KieContainer kContainer = ks.getKieClasspathContainer();

// Get specific KieBase
KieBase medicalKB = kContainer.getKieBase("medicalKB");
KieBase dentalKB = kContainer.getKieBase("dentalKB");

// Create sessions from KieBase
KieSession medicalSession = medicalKB.newKieSession();
KieSession dentalSession = dentalKB.newKieSession();
```

**Key Characteristics**:
- **Immutable**: Once built, cannot add/remove rules
- **Shareable**: Thread-safe, can be used by multiple sessions
- **Package Filter**: Includes only specified packages
- **Compilation**: Rules compiled once at build time

### 9. Sessions (KieSession)

**Definition**: Runtime execution context where facts are inserted and rules are fired.

**Types**:
- **Stateful**: Maintains working memory across multiple operations
- **Stateless**: Fire-and-forget, no memory retention

**Relationship to KieBase**:
```
KieBase (Blueprint)
    ├── Session 1 (Working Memory A)
    ├── Session 2 (Working Memory B)
    └── Session 3 (Working Memory C)
```

Multiple sessions can share same KieBase but have independent working memory.

### 10. Complete Example: Putting It All Together

**Project Structure**:
```
drools-claim-engine/
├── pom.xml
└── src/main/
    ├── java/
    │   └── com/company/claims/
    │       ├── domain/
    │       │   ├── Claim.java
    │       │   └── Policy.java
    │       └── service/
    │           └── ClaimService.java
    │
    └── resources/
        ├── META-INF/
        │   └── kmodule.xml                    # Defines KieBases and Sessions
        │
        └── com/company/claims/
            ├── validation/                    # Package: com.company.claims.validation
            │   ├── CommonValidation.drl       # 5 rules, agenda-group="validation"
            │   └── MedicalValidation.drl      # 3 rules, agenda-group="validation"
            │
            ├── pricing/                       # Package: com.company.claims.pricing
            │   ├── PricingRules.drl           # 8 rules, agenda-group="pricing"
            │   └── PricingTemplates.drl       # Template (generates 20 rules)
            │
            ├── approval/                      # Package: com.company.claims.approval
            │   ├── AutoApproval.drl           # 4 rules, activation-group="routing"
            │   └── ApprovalDecisionTable.xls  # Decision table (generates 15 rules)
            │
            └── reports/                       # Package: com.company.claims.reports
                └── Statistics.drl             # 6 rules, agenda-group="reporting"
```

**kmodule.xml**:
```xml
<kmodule xmlns="http://www.drools.org/xsd/kmodule">
    <kbase name="claimProcessingKB" packages="com.company.claims">
        <ksession name="claimSession" default="true"/>
    </kbase>
</kmodule>
```

**Rule Counts**:
- **Total DRL Files**: 7
- **Total Packages**: 4
- **Total Rules Written**: 26 (5+3+8+4+6)
- **Generated Rules**: 35 (20 from template + 15 from decision table)
- **Total Rules Loaded**: 61 rules in claimProcessingKB
- **Agenda Groups**: 4 (validation, pricing, routing, reporting)
- **Activation Groups**: 1 (routing)

**Execution Flow**:
```java
public class ClaimService {
    private KieContainer kContainer;

    public ClaimService() {
        KieServices ks = KieServices.Factory.get();
        this.kContainer = ks.getKieClasspathContainer();
    }

    public void processClaim(Claim claim, Policy policy) {
        KieSession session = kContainer.newKieSession();

        try {
            // Insert facts
            session.insert(claim);
            session.insert(policy);

            // Stage 1: Validation (8 rules: 5+3 from validation package)
            session.getAgenda().getAgendaGroup("validation").setFocus();
            session.fireAllRules();

            // Stage 2: Pricing (28 rules: 8 from DRL + 20 from template)
            session.getAgenda().getAgendaGroup("pricing").setFocus();
            session.fireAllRules();

            // Stage 3: Approval (19 rules: 4 from DRL + 15 from decision table)
            // activation-group="routing" ensures only 1 fires
            session.getAgenda().getAgendaGroup("approval").setFocus();
            session.fireAllRules();

            // Stage 4: Reporting (6 rules from reports package)
            session.getAgenda().getAgendaGroup("reporting").setFocus();
            session.fireAllRules();

        } finally {
            session.dispose();
        }
    }
}
```

### Summary: Key Takeaways

1. **DRL Files** = Physical files containing rule definitions
2. **Packages** = Logical namespaces for organizing rules
3. **Rules** = Individual business logic units (atomic)
4. **Rule Groups** = Execution control mechanisms (agenda-group, activation-group, etc.)
5. **Rule Sets** = Informal term for collection of related rules
6. **Templates** = Rule generators from external data
7. **Decision Tables** = Spreadsheet-based rule definitions
8. **KieBase** = Compiled rule container (immutable)
9. **KieSession** = Runtime execution context

**Hierarchy**:
```
Project
  └── kmodule.xml
      └── KieBase (contains)
          └── Packages (logical grouping)
              └── DRL Files (physical files)
                  └── Rules (individual logic)
                      └── Assigned to Rule Groups (execution control)
```

**Execution Flow**:
```
DRL Files → Compiled into → KieBase → Creates → KieSession → Fires → Rules
                                                                        ↓
                                                              Controlled by Rule Groups
```

This organization allows you to:
- Scale to thousands of rules
- Maintain rules independently
- Control execution order
- Enable business users to configure rules
- Test rules in isolation
- Deploy rules without code changes

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
