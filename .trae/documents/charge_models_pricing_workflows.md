# Charge Models - Pricing Workflows and Process Flows

## 1. Overall Pricing System Architecture

```mermaid
graph TB
    subgraph "Input Layer"
        A[Raw Events] --> B[Event Validation]
        C[Subscription Data] --> D[Plan Configuration]
        E[Charge Configuration] --> F[Pricing Rules]
    end
    
    subgraph "Processing Layer"
        B --> G[Aggregation Engine]
        D --> G
        F --> G
        G --> H[Charge Model Factory]
        H --> I[Standard Model]
        H --> J[Graduated Model]
        H --> K[Volume Model]
        H --> L[Package Model]
        H --> M[Percentage Model]
        H --> N[Graduated Percentage Model]
        H --> O[Prorated Graduated Model]
    end
    
    subgraph "Calculation Layer"
        I --> P[Amount Calculation]
        J --> P
        K --> P
        L --> P
        M --> P
        N --> P
        O --> P
        P --> Q[Proration Engine]
        Q --> R[Currency Conversion]
        R --> S[Tax Application]
    end
    
    subgraph "Output Layer"
        S --> T[Fee Generation]
        T --> U[Invoice Integration]
        U --> V[Billing Records]
    end
```

## 2. Charge Model Selection Workflow

```mermaid
graph TD
    A[Charge Request] --> B{Has Filters?}
    B -->|Yes| C[Process Filters]
    B -->|No| D[Use Base Properties]
    
    C --> E[Filter Matching]
    E --> F{Filter Match Found?}
    F -->|Yes| G[Apply Filter Properties]
    F -->|No| H[Apply Default Properties]
    
    G --> I[Charge Model Factory]
    H --> I
    D --> I
    
    I --> J{Charge Model Type}
    J -->|Standard| K[Standard Service]
    J -->|Graduated| L{Prorated?}
    J -->|Volume| M[Volume Service]
    J -->|Package| N[Package Service]
    J -->|Percentage| O[Percentage Service]
    J -->|Graduated Percentage| P[Graduated Percentage Service]
    J -->|Custom| Q[Custom Service]
    J -->|Dynamic| R[Dynamic Service]
    
    L -->|Yes + Has Aggregator| S[Prorated Graduated Service]
    L -->|No| T[Regular Graduated Service]
    
    K --> U[Calculate Amount]
    S --> U
    T --> U
    M --> U
    N --> U
    O --> U
    P --> U
    Q --> U
    R --> U
    
    U --> V[Return Result]
```

## 3. Graduated Charge Model Processing Flow

```mermaid
graph TD
    A[Graduated Model Request] --> B[Extract Ranges]
    B --> C[Sort Ranges by from_value]
    C --> D[Initialize Variables]
    
    D --> E[remaining_units = total_units]
    E --> F[total_amount = 0]
    F --> G[priced_units = 0]
    
    G --> H{More Ranges?}
    H -->|Yes| I[Get Current Range]
    I --> J[Calculate Tier Capacity]
    J --> K[capacity = to_value - priced_units]
    
    K --> L[Calculate Units in Tier]
    L --> M[units_in_tier = min(remaining_units, capacity)]
    
    M --> N{units_in_tier > 0?}
    N -->|Yes| O[Calculate Tier Amount]
    O --> P[tier_amount = (units_in_tier × per_unit) + flat_amount]
    P --> Q[Update Totals]
    Q --> R[total_amount += tier_amount]
    R --> S[remaining_units -= units_in_tier]
    S --> T[priced_units += units_in_tier]
    
    N -->|No| U{remaining_units <= 0?}
    U -->|Yes| V[Break Loop]
    U -->|No| H
    T --> U
    
    H -->|No| W[Return total_amount]
    V --> W
```

## 4. Percentage Charge Model with Min/Max Processing

```mermaid
graph TD
    A[Percentage Model Request] --> B{Has Min/Max?}
    B -->|Yes| C[Check License]
    B -->|No| D[Basic Calculation]
    
    C --> E{License Valid?}
    E -->|Yes| F[Per-Transaction Processing]
    E -->|No| D
    
    F --> G[Initialize Variables]
    G --> H[remaining_free_events = free_units_per_events]
    I --> J[remaining_free_amount = free_units_per_total_aggregation]
    
    J --> K{More Events?}
    K -->|Yes| L[Get Event Value]
    L --> M{Free Units Available?}
    
    M -->|Yes| N[Apply Free Events]
    N --> O[remaining_free_events -= 1]
    O --> P{remaining_free_amount > 0?}
    
    P -->|Yes| Q{remaining_free_amount > value?}
    Q -->|Yes| R[Use All Free Amount]
    R --> S[remaining_free_amount -= value]
    S --> T[event_amount = 0]
    
    Q -->|No| U[Partial Free Amount]
    U --> V[value -= remaining_free_amount]
    V --> W[remaining_free_amount = 0]
    W --> X[remaining_free_events = 0]
    
    M -->|No| Y[Calculate Event Amount]
    Y --> Z[base_amount = (value × rate) ÷ 100]
    Z --> AA[base_amount += fixed_amount]
    
    AA --> AB[Apply Min/Max]
    AB --> AC{amount < min_amount?}
    AC -->|Yes| AD[event_amount = min_amount]
    AC -->|No| AE{amount > max_amount?}
    AE -->|Yes| AF[event_amount = max_amount]
    AE -->|No| AG[event_amount = base_amount]
    
    AD --> AH[Add to Total]
    AG --> AH
    AF --> AH
    T --> AH
    X --> AH
    
    AH --> AI[total_amount += event_amount]
    AI --> K
    
    K -->|No| AJ[Return total_amount]
    D --> AK[Basic Percentage Calc]
    AK --> AJ
```

## 5. Prorated Graduated Processing Flow

```mermaid
graph TD
    A[Prorated Graduated Request] --> B[Initialize Variables]
    B --> C[full_units = per_event_aggregation]
    C --> D[prorated_units = per_event_prorated_aggregation]
    
    D --> E[index = 0]
    E --> F[overflow = 0]
    F --> G[full_sum = 0]
    G --> H[prorated_sum = 0]
    I --> J[result_amount = 0]
    
    J --> K{Events to Process OR Overflow?}
    K -->|Yes| L[Determine Current Range]
    L --> M[range = range(full_sum, overflow, next_full_unit)]
    
    M --> N{overflow > 0?}
    N -->|Yes| O[Handle Previous Overflow]
    O --> P[prorated_sum += overflow × coefficient]
    P --> Q{range[:to_value] AND full_sum >= to_value?}
    
    Q -->|Yes| R[Calculate New Overflow]
    R --> S[overflow = full_sum - to_value]
    S --> T[Adjust Prorated Sum]
    T --> U[prorated_sum -= overflow × coefficient]
    U --> V[Apply to Result]
    V --> W[result_amount += prorated_sum × per_unit_amount]
    W --> X[Reset prorated_sum = 0]
    
    Q -->|No| Y{No More Events?}
    Y -->|Yes| Z[Break Loop]
    
    N -->|No| AA[Process Current Event]
    AA --> AB{prorated_units[index].nil?}
    AB -->|Yes| Z
    AB -->|No| AC[Update Sums]
    
    AC --> AD[full_sum += full_units[index]]
    AD --> AE[Update Max]
    AE --> AF[max_full_sum = max(max_full_sum, full_sum)]
    AF --> AG[prorated_sum += prorated_units[index]]
    
    AG --> AH[index += 1]
    AH --> AI{Skip Overflow Calc?}
    AI -->|Yes| K
    AI -->|No| AJ[Calculate Overflow]
    
    AJ --> AK[overflow = calculate_overflow(full_sum, to_value, from_value)]
    AK --> AL[Adjust Prorated Sum]
    AL --> AM[prorated_sum -= overflow × coefficient]
    AM --> AN[Apply to Result]
    AN --> AO[result_amount += prorated_sum × per_unit_amount]
    AO --> AP[Reset prorated_sum = 0]
    
    AP --> K
    K -->|No| AQ[Apply Final Amount]
    AQ --> AR[result_amount += prorated_sum × per_unit_amount]
    AR --> AS[Add Flat Amounts]
    AS --> AT[result_with_flat_amount(result_amount, full_sum, max_full_sum)]
    AT --> AU[Return Final Amount]
```

## 6. Currency Conversion and Precision Workflow

```mermaid
graph TD
    A[Amount Calculation] --> B{Has Pricing Unit?}
    B -->|Yes| C[Build PricingUnitUsage]
    B -->|No| D[Standard Currency Conversion]
    
    C --> E[Extract Pricing Unit]
    E --> F[Calculate Conversion]
    F --> G[rounded_amount = amount.round(pricing_unit.exponent)]
    G --> H[amount_cents = rounded_amount × pricing_unit.subunit_to_unit]
    H --> I[precise_amount_cents = amount × pricing_unit.subunit_to_unit.to_d]
    I --> J[unit_amount_cents = unit_amount × pricing_unit.subunit_to_unit]
    
    D --> K[Standard Conversion]
    K --> L[rounded_amount = amount.round(currency.exponent)]
    L --> M[amount_cents = rounded_amount × currency.subunit_to_unit]
    M --> N[precise_amount_cents = amount × currency.subunit_to_unit.to_d]
    N --> O[unit_amount_cents = unit_amount × currency.subunit_to_unit]
    
    J --> P[Create PricingUnitUsage]
    O --> Q[Set Direct Values]
    
    P --> R[Convert to Target Currency]
    Q --> R
    
    R --> S[Calculate Adjusted Amounts]
    S --> T[adjusted_amount = amount_cents × conversion_rate ÷ pricing_unit.subunit_to_unit]
    T --> U[adjusted_unit_amount = unit_amount_cents × conversion_rate ÷ pricing_unit.subunit_to_unit]
    
    U --> V[Apply Final Conversion]
    V --> W[amount_cents = adjusted_amount.round(target_currency.exponent) × target_currency.subunit_to_unit]
    W --> X[precise_amount_cents = adjusted_amount × target_currency.subunit_to_unit.to_d]
    X --> Y[unit_amount_cents = adjusted_unit_amount × target_currency.subunit_to_unit]
    Y --> Z[precise_unit_amount = adjusted_unit_amount]
    
    Z --> AA[Return Converted Values]
```

## 7. Filter Processing and Grouping Workflow

```mermaid
graph TD
    A[Charge with Filters] --> B{Has Filters?}
    B -->|Yes| C[Process Each Filter]
    B -->|No| D[Process Base Properties]
    
    C --> E[Iterate through Filters]
    E --> F[Get Filter Properties]
    F --> G[Match Events to Filter]
    G --> H{Events Match Filter?}
    
    H -->|Yes| I[Apply Filter Properties]
    H -->|No| J[Skip Filter]
    
    I --> K[Process with Filter]
    K --> L[Calculate Filter Amount]
    L --> M[Create Filter Fee]
    M --> N[Add to Results]
    
    J --> O{More Filters?}
    O -->|Yes| E
    O -->|No| P[Process Unmatched Events]
    
    P --> Q[Create Default Filter]
    Q --> R[Apply Base Properties]
    R --> S[Process Default Fee]
    S --> T[Add to Results]
    
    D --> U[Process Single Fee]
    U --> V[Calculate Base Amount]
    V --> W[Create Base Fee]
    W --> X[Add to Results]
    
    N --> Y[Return All Fees]
    T --> Y
    X --> Y
```

## 8. Tax Application Workflow

```mermaid
graph TD
    A[Fee Creation] --> B{Apply Taxes?}
    B -->|Yes| C[Initialize Tax Service]
    B -->|No| D[Skip Tax Calculation]
    
    C --> E[Fees::ApplyTaxesService.call(fee: new_fee)]
    E --> F{Tax Calculation Success?}
    
    F -->|Yes| G[Update Fee with Taxes]
    G --> H[taxes_amount_cents = calculated_taxes]
    H --> I[taxes_precise_amount_cents = precise_taxes]
    I --> J[Update Total Amounts]
    
    F -->|No| K[Handle Tax Error]
    K --> L[Log Error]
    L --> M[Raise Exception]
    
    D --> N[Continue with Base Amount]
    J --> N
    M --> O[Fail Fee Creation]
    
    N --> P[Return Fee with Taxes]
```

## 9. Cache Integration Workflow

```mermaid
graph TD
    A[Charge Processing Request] --> B[Initialize Cache Middleware]
    B --> C[Generate Cache Key]
    C --> D{Cache Enabled?}
    
    D -->|Yes| E[Check Cache]
    E --> F{Cache Hit?}
    F -->|Yes| G[Return Cached Result]
    F -->|No| H[Process Charge]
    
    D -->|No| H
    
    H --> I[Execute Charge Model]
    I --> J[Calculate Amount]
    J --> K{Cache Enabled?}
    
    K -->|Yes| L[Store in Cache]
    L --> M[Return Result]
    K -->|No| M
    
    G --> M
```

## 10. Error Handling and Recovery Workflow

```mermaid
graph TD
    A[Charge Processing] --> B{Try Processing}
    B --> C[Execute Charge Model]
    C --> D{Calculation Success?}
    
    D -->|Yes| E[Continue Processing]
    D -->|No| F[Capture Error]
    
    F --> G{Error Type}
    G -->|Validation Error| H[Return Validation Error]
    G -->|Calculation Error| I[Return Calculation Error]
    G -->|Aggregation Error| J[Return Aggregation Error]
    G -->|Currency Error| K[Return Currency Error]
    
    H --> L[Log Error Details]
    I --> L
    J --> L
    K --> L
    
    L --> M[Set Error on Result]
    M --> N[Return Error Result]
    
    E --> O{More Processing?}
    O -->|Yes| P[Continue to Next Step]
    O -->|No| Q[Return Success Result]
    
    P --> R[Apply Proration]
    R --> S[Apply Currency Conversion]
    S --> T[Apply Taxes]
    T --> U[Generate Fee]
    U --> Q
```

## 11. Real-time Pricing Calculation Flow

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant Aggregation
    participant ChargeModel
    participant Cache
    participant Currency
    participant Tax
    participant Fee
    
    Client->>API: POST /calculate-charge
    API->>Cache: Check for cached result
    Cache-->>API: Cache miss
    API->>Aggregation: Request aggregation
    Aggregation->>Aggregation: Process events
    Aggregation-->>API: Return aggregation result
    API->>ChargeModel: Create charge model instance
    ChargeModel->>ChargeModel: Calculate base amount
    ChargeModel-->>API: Return amount details
    API->>Currency: Convert if needed
    Currency-->>API: Return converted amounts
    API->>Tax: Apply taxes
    Tax-->>API: Return tax amounts
    API->>Fee: Generate fee
    Fee-->>API: Return fee object
    API->>Cache: Store result
    API-->>Client: Return pricing result
```

## 12. Batch Processing Workflow

```mermaid
graph TD
    A[Batch Processing Request] --> B[Initialize Batch Processor]
    B --> C[Load Configuration]
    C --> D[Validate Charges]
    
    D --> E[Create Processing Queue]
    E --> F[Partition by Model Type]
    
    F --> G[Process Standard Charges]
    F --> H[Process Graduated Charges]
    F --> I[Process Volume Charges]
    F --> J[Process Package Charges]
    F --> K[Process Percentage Charges]
    
    G --> L[Aggregate Results]
    H --> L
    I --> L
    J --> L
    K --> L
    
    L --> M[Apply Currency Conversions]
    M --> N[Apply Taxes]
    N --> O[Generate Fees]
    
    O --> P{All Successful?}
    P -->|Yes| Q[Commit Transaction]
    P -->|No| R[Rollback Transaction]
    
    Q --> S[Return Results]
    R --> T[Return Error]
```

## 13. Configuration Management Workflow

```mermaid
graph TD
    A[Configuration Change] --> B{Change Type}
    B -->|Charge Model| C[Validate Model Properties]
    B -->|Pricing Rules| D[Validate Pricing Logic]
    B -->|Currency Settings| E[Validate Currency Configuration]
    B -->|Tax Settings| F[Validate Tax Configuration]
    
    C --> G{Validation Success?}
    G -->|Yes| H[Update Configuration]
    G -->|No| I[Return Validation Error]
    
    D --> J{Validation Success?}
    J -->|Yes| H
    J -->|No| I
    
    E --> K{Validation Success?}
    K -->|Yes| H
    K -->|No| I
    
    F --> L{Validation Success?}
    L -->|Yes| H
    L -->|No| I
    
    H --> M[Clear Related Cache]
    M --> N[Update Audit Log]
    N --> O[Notify Subscribers]
    O --> P[Return Success]
    
    I --> Q[Log Error]
    Q --> R[Return Error Response]
```

## 14. Monitoring and Alerting Workflow

```mermaid
graph TD
    A[System Monitoring] --> B[Collect Metrics]
    B --> C[Processing Time]
    B --> D[Error Rate]
    B --> E[Cache Hit Rate]
    B --> F[Currency Conversion Rate]
    
    C --> G{Threshold Exceeded?}
    D --> H{Error Rate High?}
    E --> I{Cache Performance Low?}
    F --> J{Conversion Issues?}
    
    G -->|Yes| K[Alert: Performance Degradation]
    H -->|Yes| L[Alert: High Error Rate]
    I -->|Yes| M[Alert: Cache Issues]
    J -->|Yes| N[Alert: Currency Issues]
    
    K --> O[Log Alert]
    L --> O
    M --> O
    N --> O
    
    O --> P[Send Notification]
    P --> Q[Create Incident]
    Q --> R[Investigate Issue]
    
    R --> S{Issue Resolved?}
    S -->|Yes| T[Close Incident]
    S -->|No| U[Escalate Issue]
    
    T --> V[Update Documentation]
    U --> W[Higher Level Support]
```

These comprehensive workflows ensure reliable, efficient, and maintainable pricing operations across all charge models while providing clear visibility into