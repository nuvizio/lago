# Lago Billing System - Deep Technical Analysis

## Executive Summary

Lago is a sophisticated open-source billing and usage metering platform built with Ruby on Rails (API) and Go (event processing). The system implements a complex multi-tenant architecture supporting various billing models, real-time event processing, and advanced aggregation mechanisms using both PostgreSQL and ClickHouse databases.

## 1. Event Processing and Aggregation Engines

### 1.1 Dual Storage Architecture

Lago implements a **dual storage strategy** for event processing:

#### PostgreSQL Store (`Events::Stores::PostgresStore`)
- **Primary use**: Real-time event ingestion and basic aggregations
- **Implementation**: ActiveRecord-based with JSONB properties storage
- **Aggregation methods**: Sum, count, max, unique_count, weighted_sum, latest
- **Key features**:
  - JSONB property filtering with `@>` operators
  - Timezone-aware date calculations
  - Prorated calculations using duration ratios
  - Event deduplication via `transaction_id` uniqueness

#### ClickHouse Store (`Events::Stores::ClickhouseStore`)
- **Primary use**: High-performance analytics and complex aggregations
- **Implementation**: Columnar storage with optimized analytical queries
- **Key advantages**:
  - Decimal precision (38,26) for financial calculations
  - Advanced SQL with CTEs for complex aggregations
  - Efficient handling of large event volumes
  - Optimized for time-series data patterns

### 1.2 Event Enrichment Pipeline (Go Events Processor)

The Go-based events processor implements a **Kafka-driven pipeline**:

```
Raw Events → Event Enrichment → ClickHouse Storage → Billing Calculation
```

**Key Components**:
- **Event Enrichment Service**: Adds subscription, plan, and charge context
- **Cache Service**: Redis-based charge caching for performance
- **Subscription Refresh**: Handles subscription state changes
- **Dead Letter Queue**: Failed event handling with retry logic

**Event Flow**:
1. Events ingested via HTTP API or Kafka
2. Go processor enriches events with billing context
3. Events stored in both PostgreSQL and ClickHouse
4. Real-time billing calculations triggered

### 1.3 Aggregation Types and Implementation

#### Sum Aggregation (`BillableMetrics::Aggregations::SumService`)
```ruby
def compute_aggregation(options: {})
  aggregation = event_store.sum
  result.aggregation = aggregation
  result.pay_in_advance_aggregation = compute_pay_in_advance_aggregation
  result.count = event_store.count
  result.options = {running_total: running_total(options)}
  result
end
```

#### Unique Count Aggregation (`BillableMetrics::Aggregations::UniqueCountService`)
- **Operation Types**: Supports "add" and "remove" operations
- **Active Property Tracking**: Maintains state of unique properties
- **Pay-in-advance Logic**: Real-time unique count validation

#### Weighted Sum Aggregation
- **Time-based weighting**: Events weighted by duration in billing period
- **Proration support**: Automatic proration for partial periods
- **Custom intervals**: Configurable weighting intervals (seconds)

## 2. Charge Models Implementation

### 2.1 Graduated Pricing (`ChargeModels::GraduatedService`)

**Algorithm**:
```ruby
def compute_amount
  amount_details.fetch(:graduated_ranges).sum { |e| e[:total_with_flat_amount] }
end
```

**Key Features**:
- **Tiered pricing**: Different rates per usage tier
- **Flat amounts**: Per-tier fixed fees
- **Range validation**: Automatic tier boundary handling
- **Projected amounts**: Forward-looking cost calculations

**Implementation Details**:
- Ranges defined with `from_value`, `to_value`, `per_unit_amount`, `flat_amount`
- Supports infinite upper bounds (`to_value: nil`)
- Handles partial tier usage with proration

### 2.2 Volume Pricing (`ChargeModels::VolumeService`)

**Algorithm**:
```ruby
def compute_amount
  return 0 if units.zero?
  per_unit_total_amount + flat_unit_amount
end
```

**Characteristics**:
- **Single tier application**: Entire usage billed at applicable tier rate
- **Tier matching**: Finds appropriate tier based on total usage
- **Unit amount calculation**: Average cost per unit including flat fees

### 2.3 Package Pricing (`ChargeModels::PackageService`)

**Algorithm**:
```ruby
def compute_amount
  return 0 if paid_units.negative?
  package_count = paid_units.fdiv(per_package_size).ceil
  package_count * per_package_unit_amount
end
```

**Features**:
- **Free units**: Configurable free usage allowance
- **Package sizing**: Fixed units per package
- **Ceiling calculation**: Always round up partial packages
- **Paid units calculation**: Usage minus free allowance

### 2.4 Percentage Pricing (`ChargeModels::PercentageService`)

**Advanced Features**:
- **Transaction-level min/max**: Per-transaction amount limits
- **Free units**: Both per-event and per-aggregation free allowances
- **Fixed fees**: Per-transaction fixed amounts
- **Rate application**: Percentage of transaction value

**Complex Logic**:
```ruby
def compute_amount_with_transaction_min_max
  remaining_free_events = free_units_per_events
  remaining_free_amount = free_units_per_total_aggregation
  
  events_values.reduce(0) do |total_amount, event_value|
    # Apply free units logic
    # Apply rate and fixed amounts
    # Apply min/max constraints
    total_amount + event_amount
  end
end
```

## 3. Invoice Generation and Fee Calculation Workflows

### 3.1 Invoice Calculation Service (`Invoices::CalculateFeesService`)

**Workflow**:
1. **Boundary Calculation**: Determine billing period boundaries
2. **Subscription Fee Creation**: Base subscription charges
3. **Charge Fee Creation**: Usage-based charges
4. **Credit Application**: Coupons and prepaid credits
5. **Tax Computation**: Tax calculation and application
6. **Final Amount Calculation**: Total invoice amount

**Key Boundaries**:
```ruby
boundaries = BillingPeriodBoundaries.new(
  from_datetime: invoice_subscription.from_datetime,
  to_datetime: invoice_subscription.to_datetime,
  charges_from_datetime: invoice_subscription.charges_from_datetime,
  charges_to_datetime: invoice_subscription.charges_to_datetime,
  timestamp: invoice_subscription.timestamp,
  charges_duration: date_service.charges_duration_in_days
)
```

### 3.2 Fee Creation Service (`Fees::ChargeService`)

**Responsibilities**:
- **Aggregation**: Event aggregation based on billable metric type
- **Charge Model Application**: Apply appropriate pricing model
- **Proration**: Handle partial billing periods
- **Tax Application**: Apply taxes to individual fees
- **Currency Conversion**: Handle multi-currency scenarios

**Fee Types**:
- **Subscription Fees**: Recurring base charges
- **Charge Fees**: Usage-based charges
- **True-up Fees**: Minimum commitment adjustments
- **Credit Note Fees**: Refund and credit applications

### 3.3 Tax Computation (`Invoices::ComputeTaxesAndTotalsService`)

**Tax Flow**:
1. **Fee-level taxes**: Applied to individual line items
2. **Invoice-level taxes**: Applied to subtotal
3. **Provider taxes**: External tax service integration
4. **Tax status tracking**: Pending, succeeded, failed states

**Tax Types**:
- **Applied taxes**: Direct tax application
- **Provider taxes**: External tax calculation (Avalara, Anrok)
- **Custom tax rates**: Organization-specific tax rules

## 4. Usage Metering and Aggregation Logic

### 4.1 Event Aggregation Patterns

#### Count Aggregation
- **Simple counting**: Number of events in period
- **Grouped counting**: Count by property groups
- **Filtered counting**: Apply property filters

#### Sum Aggregation
- **Property summation**: Sum of numeric properties
- **Grouped summation**: Sum by property groups
- **Prorated summation**: Time-weighted sums

#### Unique Count Aggregation
- **Property uniqueness**: Count unique property values
- **Operation tracking**: Add/remove operations
- **State management**: Track active unique properties

### 4.2 Advanced Aggregation Features

#### Prorated Aggregations
```ruby
def prorated_sum(period_duration:, persisted_duration: nil)
  ratio = if persisted_duration
    persisted_duration.fdiv(period_duration)
  else
    duration_ratio_sql("events.timestamp", to_datetime, period_duration)
  end
  
  sql = "SUM((#{property})::numeric * (#{ratio})::numeric)"
  connection.execute(sql).first["sum_result"]
end
```

#### Weighted Sum Aggregations
- **Time-based weighting**: Weight by duration in period
- **Custom intervals**: Configurable weighting intervals
- **Initial values**: Support for carry-over values

#### Grouped Aggregations
- **Multi-dimensional grouping**: Group by multiple properties
- **Hierarchical grouping**: Support for nested groups
- **Dynamic grouping**: Runtime group configuration

### 4.3 Real-time Usage Tracking (`DailyUsages::ComputeService`)

**Purpose**: Track daily usage for customer visibility
**Implementation**:
- **Cache-based**: Uses cached aggregations for performance
- **Diff calculation**: Compare with previous usage
- **Timezone handling**: Customer timezone awareness
- **Billing day exclusion**: Skip billing day computation

## 5. External Connections and Integrations

### 5.1 Payment Providers
- **Stripe**: Full integration with Stripe Billing
- **Adyen**: Payment processing via Adyen
- **GoCardless**: Direct debit processing
- **Custom providers**: Extensible payment provider framework

### 5.2 Tax Providers
- **Avalara**: Automated tax calculation
- **Anrok**: Sales tax automation
- **Custom tax rules**: Organization-specific tax logic

### 5.3 Accounting Integrations
- **NetSuite**: ERP integration
- **Xero**: Accounting software integration
- **QuickBooks**: Financial management integration

### 5.4 Data Export and Analytics
- **CSV exports**: Usage and billing data export
- **API access**: RESTful API for data access
- **Webhook notifications**: Real-time event notifications

## 6. Performance Optimizations

### 6.1 Caching Strategy
- **Charge caching**: Redis-based charge property caching
- **Aggregation caching**: Cached aggregation results
- **Subscription state**: Subscription boundary caching

### 6.2 Database Optimizations
- **Index strategy**: Optimized indexes for event queries
- **Partitioning**: Time-based partitioning for large datasets
- **Connection pooling**: Database connection optimization

### 6.3 Event Processing Optimization
- **Batch processing**: Kafka batch processing
- **Parallel processing**: Multi-worker event processing
- **Dead letter handling**: Failed event retry mechanism

## 7. Scalability Considerations

### 7.1 Horizontal Scaling
- **Kafka-based architecture**: Event-driven horizontal scaling
- **Database sharding**: Organization-based data partitioning
- **Microservices**: Separate services for different concerns

### 7.2 Data Volume Management
- **Event retention**: Configurable event retention policies
- **Aggregation archival**: Historical data archival
- **Real-time vs batch**: Hybrid real-time and batch processing

## 8. Security and Compliance

### 8.1 Data Security
- **Encryption**: Data encryption at rest and in transit
- **Access control**: Role-based access control
- **Audit logging**: Comprehensive audit trail

### 8.2 Financial Compliance
- **Audit trail**: Complete billing history
- **Tax compliance**: Multi-jurisdiction tax support
- **Revenue recognition**: ASC 606 compliance features

## Conclusion

Lago implements a sophisticated billing platform with:
- **Dual database architecture** for optimal performance
- **Flexible charge models** supporting complex pricing scenarios
- **Real-time event processing** with Go-based pipeline
- **Comprehensive tax and payment integrations**
- **Scalable architecture** supporting high-volume event processing

The system's modular design allows for extensive customization while maintaining performance and reliability at scale.