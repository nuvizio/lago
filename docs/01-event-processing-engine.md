# Event Processing & Aggregation Engine

## Overview

The Event Processing & Aggregation Engine is the core component of Lago's billing system, responsible for ingesting, validating, transforming, and aggregating usage events from various sources. It implements a dual-database architecture using PostgreSQL for operational data and ClickHouse for analytical workloads.

## Architecture

### Dual Database Strategy

```
┌─────────────────┐    ┌─────────────────┐
│   PostgreSQL    │    │   ClickHouse    │
│   (OLTP)        │    │   (OLAP)        │
├─────────────────┤    ├─────────────────┤
│ • Events        │    │ • Events        │
│ • Subscriptions │    │ • Aggregations  │
│ • Invoices      │    │ • Analytics     │
│ • Real-time     │    │ • Historical    │
│   queries       │    │   analysis      │
└─────────────────┘    └─────────────────┘
```

### Event Processing Pipeline

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Events    │───▶│  Enrichment │───▶│ Aggregation │───▶│   Storage   │
│   Ingestion │    │   Engine    │    │   Engine    │    │   Layer     │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
       │                  │                  │                  │
       ▼                  ▼                  ▼                  ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ Validation  │    │ Kafka       │    │ ClickHouse  │    │ PostgreSQL  │
│   Rules     │    │ Streaming   │    │ Analytics   │    │  Metadata   │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

## Core Components

### 1. Event Ingestion Layer

**Event Schema Definition:**
```ruby
# api/app/models/event.rb
class Event < ApplicationRecord
  include Events::Common
  
  belongs_to :organization
  belongs_to :customer, optional: true
  
  validates :transaction_id, presence: true, uniqueness: { scope: :organization_id }
  validates :code, presence: true
  validates :timestamp, presence: true
  validates :properties, presence: true
  
  scope :from_datetime, ->(datetime) { where('timestamp >= ?', datetime) }
  scope :to_datetime, ->(datetime) { where('timestamp <= ?', datetime) }
  scope :by_code, ->(code) { where(code: code) }
  scope :by_customer, ->(customer_id) { where(customer_id: customer_id) }
end
```

**Event Validation Rules:**
```ruby
# api/app/services/events/validate_service.rb
module Events
  class ValidateService < BaseService
    def validate(params)
      # Transaction ID validation
      return result.single_validation_failure!(field: :transaction_id, error_code: 'value_is_mandatory') if params[:transaction_id].blank?
      
      # Code validation
      return result.single_validation_failure!(field: :code, error_code: 'value_is_mandatory') if params[:code].blank?
      
      # Timestamp validation
      return result.single_validation_failure!(field: :timestamp, error_code: 'value_is_mandatory') if params[:timestamp].blank?
      
      # Customer validation
      if params[:external_customer_id].present?
        customer = Customer.find_by(external_id: params[:external_customer_id], organization: organization)
        return result.single_validation_failure!(field: :external_customer_id, error_code: 'customer_not_found') unless customer
      end
      
      # Properties validation
      return result.single_validation_failure!(field: :properties, error_code: 'value_is_mandatory') if params[:properties].blank?
      
      result.event = Event.new(params)
      result
    end
  end
end
```

### 2. Event Enrichment Pipeline (Go)

**Main Event Processor:**
```go
// events-processor/main.go
package main

import (
    "context"
    "log"
    
    "github.com/getlago/lago/events-processor/models"
    "github.com/getlago/lago/events-processor/utils"
    "github.com/twmb/franz-go/pkg/kgo"
)

type EventProcessor struct {
    db       *models.DB
    kafkaClient *kgo.Client
    logger   *log.Logger
}

func (ep *EventProcessor) ProcessEvent(ctx context.Context, event models.Event) error {
    // Step 1: Validate event
    if err := ep.validateEvent(event); err != nil {
        return err
    }
    
    // Step 2: Enrich event with customer data
    enrichedEvent, err := ep.enrichEvent(ctx, event)
    if err != nil {
        return err
    }
    
    // Step 3: Store in PostgreSQL
    if err := ep.storeEvent(ctx, enrichedEvent); err != nil {
        return err
    }
    
    // Step 4: Publish to Kafka for aggregation
    if err := ep.publishToKafka(ctx, enrichedEvent); err != nil {
        return err
    }
    
    return nil
}

func (ep *EventProcessor) enrichEvent(ctx context.Context, event models.Event) (models.EnrichedEvent, error) {
    // Fetch customer information
    customer, err := ep.db.GetCustomer(ctx, event.CustomerID)
    if err != nil {
        return models.EnrichedEvent{}, err
    }
    
    // Fetch subscription information
    subscriptions, err := ep.db.GetActiveSubscriptions(ctx, event.CustomerID, event.Timestamp)
    if err != nil {
        return models.EnrichedEvent{}, err
    }
    
    // Fetch billable metrics
    billableMetric, err := ep.db.GetBillableMetric(ctx, event.OrganizationID, event.Code)
    if err != nil {
        return models.EnrichedEvent{}, err
    }
    
    return models.EnrichedEvent{
        Event:        event,
        Customer:     customer,
        Subscriptions: subscriptions,
        BillableMetric: billableMetric,
        EnrichedAt:   utils.Now(),
    }, nil
}
```

**Event Storage Implementation:**
```go
// events-processor/models/stores.go
package models

import (
    "context"
    "time"
    
    "gorm.io/gorm"
)

type EventStore interface {
    StoreEvent(ctx context.Context, event EnrichedEvent) error
    GetEvents(ctx context.Context, filter EventFilter) ([]Event, error)
    GetAggregatedUsage(ctx context.Context, params AggregationParams) (UsageResult, error)
}

type PostgresEventStore struct {
    db *gorm.DB
}

func (s *PostgresEventStore) StoreEvent(ctx context.Context, event EnrichedEvent) error {
    return s.db.WithContext(ctx).Create(&Event{
        TransactionID:   event.TransactionID,
        OrganizationID: event.OrganizationID,
        CustomerID:    event.CustomerID,
        SubscriptionID: event.SubscriptionID,
        Code:          event.Code,
        Timestamp:     event.Timestamp,
        Properties:    event.Properties,
        CreatedAt:     time.Now(),
    }).Error
}

func (s *PostgresEventStore) GetAggregatedUsage(ctx context.Context, params AggregationParams) (UsageResult, error) {
    var result UsageResult
    
    query := s.db.WithContext(ctx).
        Model(&Event{}).
        Where("organization_id = ?", params.OrganizationID).
        Where("code = ?", params.Code).
        Where("timestamp >= ?", params.FromDate).
        Where("timestamp <= ?", params.ToDate)
    
    if params.CustomerID != "" {
        query = query.Where("customer_id = ?", params.CustomerID)
    }
    
    if params.SubscriptionID != "" {
        query = query.Where("subscription_id = ?", params.SubscriptionID)
    }
    
    switch params.AggregationType {
    case "sum":
        var sum float64
        err := query.Select("SUM((properties->>?)::float)", params.PropertyName).Scan(&sum).Error
        result.Value = sum
        return result, err
        
    case "count":
        var count int64
        err := query.Count(&count).Error
        result.Value = float64(count)
        return result, err
        
    case "unique_count":
        var uniqueCount int64
        err := query.Distinct(params.PropertyName).Count(&uniqueCount).Error
        result.Value = float64(uniqueCount)
        return result, err
    }
    
    return result, nil
}
```

### 3. Aggregation Engine

**Weighted Sum Aggregation (ClickHouse):**
```ruby
# api/app/services/events/stores/clickhouse/weighted_sum_query.rb
module Events
  module Stores
    module Clickhouse
      class WeightedSumQuery < BaseService
        def initialize(aggregation:, charges_filters:)
          @aggregation = aggregation
          @charges_filters = charges_filters
        end

        def call
          query = build_base_query
          query = add_filters(query)
          query = add_group_by(query)
          query = add_order_by(query)
          
          result.query = query
          result
        end

        private

        def build_base_query
          <<-SQL
            SELECT 
              customer_id,
              subscription_id,
              code,
              SUM(CASE 
                WHEN properties.#{sanitized_property_name} IS NOT NULL 
                THEN properties.#{sanitized_property_name}::Float64 * (timestamp - LAG(timestamp) OVER (PARTITION BY customer_id, subscription_id ORDER BY timestamp))
                ELSE 0 
              END) as weighted_sum,
              COUNT(*) as event_count,
              MIN(timestamp) as first_event_at,
              MAX(timestamp) as last_event_at
            FROM events
            WHERE organization_id = '#{organization_id}'
              AND code = '#{code}'
              AND timestamp >= '#{from_datetime}'
              AND timestamp <= '#{to_datetime}'
          SQL
        end

        def add_filters(query)
          if customer_id.present?
            query += " AND customer_id = '#{customer_id}'"
          end
          
          if subscription_id.present?
            query += " AND subscription_id = '#{subscription_id}'"
          end
          
          query
        end

        def add_group_by(query)
          <<-SQL
            #{query}
            GROUP BY customer_id, subscription_id, code
          SQL
        end

        def add_order_by(query)
          <<-SQL
            #{query}
            ORDER BY weighted_sum DESC
          SQL
        end
      end
    end
  end
end
```

**Unique Count Aggregation with Proration:**
```ruby
# api/app/services/events/stores/postgres/unique_count_query.rb
module Events
  module Stores
    module Postgres
      class UniqueCountQuery < BaseService
        def initialize(aggregation:, charges_filters:)
          @aggregation = aggregation
          @charges_filters = charges_filters
        end

        def call
          query = build_prorated_query
          result.query = query
          result
        end

        private

        def build_prorated_query
          <<-SQL
            WITH customer_states AS (
              SELECT 
                customer_id,
                subscription_id,
                properties->>'#{sanitized_property_name}' as unique_value,
                timestamp,
                LAG(properties->>'#{sanitized_property_name}') OVER (
                  PARTITION BY customer_id, subscription_id 
                  ORDER BY timestamp
                ) as previous_value,
                CASE 
                  WHEN properties->>'#{sanitized_property_name}' IS DISTINCT FROM 
                       LAG(properties->>'#{sanitized_property_name}') OVER (
                         PARTITION BY customer_id, subscription_id 
                         ORDER BY timestamp
                       )
                  THEN 1 
                  ELSE 0 
                END as state_changed
              FROM events
              WHERE organization_id = '#{organization_id}'
                AND code = '#{code}'
                AND timestamp >= '#{from_datetime}'
                AND timestamp <= '#{to_datetime}'
                AND properties->>'#{sanitized_property_name}' IS NOT NULL
            ),
            prorated_periods AS (
              SELECT 
                customer_id,
                subscription_id,
                unique_value,
                timestamp as start_time,
                LEAD(timestamp) OVER (
                  PARTITION BY customer_id, subscription_id 
                  ORDER BY timestamp
                ) as end_time,
                CASE 
                  WHEN state_changed = 1 OR previous_value IS NULL THEN 'added'
                  ELSE 'continued'
                END as state_type
              FROM customer_states
              WHERE state_changed = 1 OR previous_value IS NULL
            )
            SELECT 
              customer_id,
              subscription_id,
              COUNT(DISTINCT unique_value) as unique_count,
              SUM(
                EXTRACT(EPOCH FROM (
                  COALESCE(end_time, '#{to_datetime}') - start_time
                )) / 86400.0
              ) as prorated_days,
              COUNT(*) as state_changes
            FROM prorated_periods
            GROUP BY customer_id, subscription_id
          SQL
        end
      end
    end
  end
end
```

### 4. Real-time Processing Pipeline

**Kafka Integration:**
```ruby
# api/app/services/events/process_batch_service.rb
module Events
  class ProcessBatchService < BaseService
    def initialize(organization_id:, events:)
      @organization_id = organization_id
      @events = events
    end

    def call
      # Validate all events first
      validated_events = validate_events
      
      # Store in PostgreSQL for operational queries
      store_in_postgres(validated_events)
      
      # Publish to Kafka for real-time processing
      publish_to_kafka(validated_events)
      
      # Update real-time aggregations
      update_real_time_aggregations(validated_events)
      
      result.events = validated_events
      result
    end

    private

    def validate_events
      @events.map do |event_params|
        validation_service = Events::ValidateService.new(result.user, event_params)
        validation_result = validation_service.validate(event_params)
        
        if validation_result.success?
          validation_result.event
        else
          # Log validation failure
          Rails.logger.error("Event validation failed: #{validation_result.errors}")
          nil
        end
      end.compact
    end

    def publish_to_kafka(events)
      events.each do |event|
        KafkaProducer.produce(
          topic: 'events.ingested',
          key: event.transaction_id,
          value: {
            organization_id: event.organization_id,
            customer_id: event.customer_id,
            subscription_id: event.subscription_id,
            code: event.code,
            timestamp: event.timestamp,
            properties: event.properties,
            created_at: event.created_at
          }.to_json
        )
      end
    end

    def update_real_time_aggregations(events)
      events.group_by(&:code).each do |code, code_events|
        # Update Redis cache for real-time dashboards
        RealTimeAggregationJob.perform_later(
          organization_id: @organization_id,
          code: code,
          events: code_events
        )
      end
    end
  end
end
```

**Real-time Aggregation Job:**
```ruby
# api/app/jobs/real_time_aggregation_job.rb
class RealTimeAggregationJob < ApplicationJob
  queue_as :default
  
  def perform(organization_id:, code:, events:)
    # Get current aggregation from Redis
    current_aggregation = Redis.current.hget("real_time:#{organization_id}:#{code}", "current_value")&.to_f || 0
    
    # Calculate new aggregation based on event type
    new_value = case code
    when 'api_calls'
      current_aggregation + events.count
    when 'gb_seconds'
      current_aggregation + events.sum { |e| e.properties['value'].to_f }
    when 'active_users'
      # Get unique user count
      unique_users = events.map { |e| e.properties['user_id'] }.uniq.count
      [current_aggregation, unique_users].max
    else
      current_aggregation
    end
    
    # Update Redis with new value
    Redis.current.hset("real_time:#{organization_id}:#{code}", "current_value", new_value)
    Redis.current.hset("real_time:#{organization_id}:#{code}", "last_updated", Time.current.to_i)
    
    # Set expiration to 24 hours
    Redis.current.expire("real_time:#{organization_id}:#{code}", 86400)
  end
end
```

## Performance Optimizations

### 1. Database Partitioning

**PostgreSQL Partitioning Strategy:**
```sql
-- Create partitioned events table
CREATE TABLE events (
    id BIGSERIAL,
    organization_id UUID NOT NULL,
    customer_id UUID,
    subscription_id UUID,
    transaction_id VARCHAR(255) NOT NULL,
    code VARCHAR(255) NOT NULL,
    timestamp TIMESTAMP NOT NULL,
    properties JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id, organization_id, timestamp)
) PARTITION BY RANGE (timestamp);

-- Create monthly partitions
CREATE TABLE events_2024_01 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE events_2024_02 PARTITION OF events
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Create indexes on partitions
CREATE INDEX idx_events_2024_01_org_code ON events_2024_01 (organization_id, code);
CREATE INDEX idx_events_2024_01_customer ON events_2024_01 (customer_id, timestamp);
CREATE INDEX idx_events_2024_01_subscription ON events_2024_01 (subscription_id, timestamp);
CREATE INDEX idx_events_2024_01_timestamp ON events_2024_01 (timestamp);

-- GIN index for JSONB properties
CREATE INDEX idx_events_2024_01_properties ON events_2024_01 USING GIN (properties);
```

**ClickHouse Partitioning Strategy:**
```sql
-- Create ClickHouse table with partitioning
CREATE TABLE events
(
    organization_id UUID,
    customer_id UUID,
    subscription_id UUID,
    transaction_id String,
    code String,
    timestamp DateTime,
    properties String,
    created_at DateTime DEFAULT now()
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (organization_id, code, timestamp)
TTL timestamp + INTERVAL 1 YEAR;

-- Create materialized view for aggregations
CREATE MATERIALIZED VIEW events_hourly_mv
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (organization_id, code, toStartOfHour(timestamp))
AS SELECT
    organization_id,
    code,
    toStartOfHour(timestamp) as hour,
    count() as event_count,
    sum(JSONExtractFloat(properties, 'value')) as total_value
FROM events
GROUP BY organization_id, code, hour;
```

### 2. Connection Pooling

**PostgreSQL Connection Pooling:**
```ruby
# config/database.yml
production:
  primary:
    adapter: postgresql
    database: lago_production
    username: <%= ENV['DATABASE_USERNAME'] %>
    password: <%= ENV['DATABASE_PASSWORD'] %>
    host: <%= ENV['DATABASE_HOST'] %>
    port: <%= ENV['DATABASE_PORT'] || 5432 %>
    pool: <%= ENV['DATABASE_POOL'] || 25 %>
    checkout_timeout: 5
    reaping_frequency: 10
    dead_connection_timeout: 30
    
  clickhouse:
    adapter: clickhouse
    database: lago_analytics
    username: <%= ENV['CLICKHOUSE_USERNAME'] %>
    password: <%= ENV['CLICKHOUSE_PASSWORD'] %>
    host: <%= ENV['CLICKHOUSE_HOST'] %>
    port: <%= ENV['CLICKHOUSE_PORT'] || 8123 %>
    pool: <%= ENV['CLICKHOUSE_POOL'] || 10 %>
```

**Go Connection Pooling:**
```go
// events-processor/models/db.go
package models

import (
    "database/sql"
    "time"
    
    "github.com/jackc/pgx/v5/stdlib"
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
)

func NewPostgresDB(connectionString string) (*gorm.DB, error) {
    sqlDB, err := sql.Open("pgx", connectionString)
    if err != nil {
        return nil, err
    }
    
    // Configure connection pool
    sqlDB.SetMaxIdleConns(10)
    sqlDB.SetMaxOpenConns(100)
    sqlDB.SetConnMaxLifetime(time.Hour)
    sqlDB.SetConnMaxIdleTime(10 * time.Minute)
    
    dialector := postgres.New(postgres.Config{
        Conn: sqlDB,
    })
    
    return gorm.Open(dialector, &gorm.Config{
        PrepareStmt: true,
        CreateBatchSize: 1000,
    })
}
```

### 3. Caching Strategy

**Multi-level Caching:**
```ruby
# config/initializers/redis.rb
require 'redis'

# Layer 1: Hot data cache (1-5 minutes)
HOT_CACHE = Redis.new(
  url: ENV['REDIS_HOT_URL'] || 'redis://localhost:6379/0',
  timeout: 0.1,
  reconnect_attempts: 3
)

# Layer 2: Warm data cache (1-24 hours)
WARM_CACHE = Redis.new(
  url: ENV['REDIS_WARM_URL'] || 'redis://localhost:6379/1',
  timeout: 0.5,
  reconnect_attempts: 2
)

# Layer 3: Cold data cache (1-30 days)
COLD_CACHE = Redis.new(
  url: ENV['REDIS_COLD_URL'] || 'redis://localhost:6379/2',
  timeout: 1.0,
  reconnect_attempts: 1
)
```

**Cache Implementation:**
```ruby
# api/app/services/events/cache_service.rb
module Events
  class CacheService < BaseService
    HOT_CACHE_TTL = 5.minutes
    WARM_CACHE_TTL = 6.hours
    COLD_CACHE_TTL = 7.days
    
    def get_aggregation(organization_id:, code:, customer_id: nil, subscription_id: nil)
      cache_key = build_cache_key(organization_id, code, customer_id, subscription_id)
      
      # Try hot cache first
      if (value = HOT_CACHE.get(cache_key))
        return JSON.parse(value)
      end
      
      # Try warm cache
      if (value = WARM_CACHE.get(cache_key))
        # Promote to hot cache
        HOT_CACHE.setex(cache_key, HOT_CACHE_TTL, value)
        return JSON.parse(value)
      end
      
      # Try cold cache
      if (value = COLD_CACHE.get(cache_key))
        # Promote to warm cache
        WARM_CACHE.setex(cache_key, WARM_CACHE_TTL, value)
        HOT_CACHE.setex(cache_key, HOT_CACHE_TTL, value)
        return JSON.parse(value)
      end
      
      nil
    end
    
    def set_aggregation(organization_id:, code:, customer_id: nil, subscription_id: nil, value:)
      cache_key = build_cache_key(organization_id, code, customer_id, subscription_id)
      json_value = value.to_json
      
      # Store in all cache layers
      HOT_CACHE.setex(cache_key, HOT_CACHE_TTL, json_value)
      WARM_CACHE.setex(cache_key, WARM_CACHE_TTL, json_value)
      COLD_CACHE.setex(cache_key, COLD_CACHE_TTL, json_value)
    end
    
    private
    
    def build_cache_key(organization_id, code, customer_id, subscription_id)
      parts = [organization_id, code, customer_id, subscription_id].compact
      "events:aggregation:#{parts.join(':')}"
    end
  end
end
```

## Monitoring and Observability

### 1. Metrics Collection

**Application Metrics:**
```ruby
# api/app/services/events/metrics_service.rb
module Events
  class MetricsService < BaseService
    def track_event_processed(event, processing_time)
      # Custom metrics
      Yabeda.events.processed.increment({
        organization_id: event.organization_id,
        code: event.code,
        status: 'success'
      })
      
      Yabeda.events.processing_time.measure({
        organization_id: event.organization_id,
        code: event.code
      }, processing_time)
      
      # Datadog metrics
      StatsD.increment('events.processed', tags: [
        "organization:#{event.organization_id}",
        "code:#{event.code}",
        "status:success"
      ])
      
      StatsD.histogram('events.processing_time', processing_time, tags: [
        "organization:#{event.organization_id}",
        "code:#{event.code}"
      ])
    end
    
    def track_aggregation_calculation(aggregation_type, calculation_time, event_count)
      Yabeda.events.aggregation_calculation.increment({
        type: aggregation_type
      })
      
      Yabeda.events.aggregation_time.measure({
        type: aggregation_type
      }, calculation_time)
      
      Yabeda.events.aggregation_events.measure({
        type: aggregation_type
      }, event_count)
    end
  end
end
```

### 2. Distributed Tracing

**OpenTelemetry Integration:**
```ruby
# config/initializers/opentelemetry.rb
require 'opentelemetry/sdk'
require 'opentelemetry/exporter/otlp'
require 'opentelemetry/instrumentation/all'

OpenTelemetry::SDK.configure do |c|
  c.service_name = 'lago-events-processor'
  c.service_version = ENV['APP_VERSION'] || '1.0.0'
  
  # Export to collector
  c.add_span_processor(
    OpenTelemetry::SDK::Trace::Export::BatchSpanProcessor.new(
      OpenTelemetry::Exporter::OTLP::Exporter.new(
        endpoint: ENV['OTEL_EXPORTER_OTLP_ENDPOINT'] || 'http://localhost:4317'
      )
    )
  )
  
  # Enable all instrumentation
  c.use_all()
end
```

**Go Tracing Implementation:**
```go
// events-processor/tracing/tracing.go
package tracing

import (
    "context"
    
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/sdk/resource"
    "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.4.0"
)

func InitTracer(serviceName, serviceVersion string) (*trace.TracerProvider, error) {
    ctx := context.Background()
    
    // Create exporter
    exporter, err := otlptracegrpc.New(ctx,
        otlptracegrpc.WithEndpoint("localhost:4317"),
        otlptracegrpc.WithInsecure(),
    )
    if err != nil {
        return nil, err
    }
    
    // Create resource
    res, err := resource.New(ctx,
        resource.WithAttributes(
            semconv.ServiceNameKey.String(serviceName),
            semconv.ServiceVersionKey.String(serviceVersion),
        ),
    )
    if err != nil {
        return nil, err
    }
    
    // Create tracer provider
    tp := trace.NewTracerProvider(
        trace.WithBatcher(exporter),
        trace.WithResource(res),
    )
    
    otel.SetTracerProvider(tp)
    return tp, nil
}

func StartSpan(ctx context.Context, name string, attrs ...attribute.KeyValue) (context.Context, trace.Span) {
    tracer := otel.Tracer("events-processor")
    return tracer.Start(ctx, name, trace.WithAttributes(attrs...))
}
```

## Scaling Considerations

### 1. Horizontal Scaling

**Kafka Partitioning Strategy:**
```yaml
# kafka-topics-config.yml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: events.ingested
  labels:
    strimzi.io/cluster: lago-kafka
spec:
  partitions: 64  # High partition count for parallel processing
  replicas: 3
  config:
    retention.ms: 604800000  # 7 days
    segment.ms: 86400000     # 1 day segments
    cleanup.policy: delete
```

**Consumer Group Scaling:**
```go
// events-processor/consumer/consumer.go
package consumer

import (
    "context"
    "sync"
    
    "github.com/twmb/franz-go/pkg/kgo"
)

type EventConsumer struct {
    client *kgo.Client
    processor EventProcessor
    logger   Logger
}

func NewEventConsumer(brokers []string, group string, processor EventProcessor) (*EventConsumer, error) {
    client, err := kgo.NewClient(
        kgo.SeedBrokers(brokers...),
        kgo.ConsumerGroup(group),
        kgo.ConsumeTopics("events.ingested"),
        kgo.OnPartitionsAssigned(func(ctx context.Context, client *kgo.Client, m map[string][]int32) {
            // Rebalance handling
            for topic, partitions := range m {
                logger.Printf("Assigned partitions for topic %s: %v", topic, partitions)
            }
        }),
    )
    if err != nil {
        return nil, err
    }
    
    return &EventConsumer{
        client:    client,
        processor: processor,
        logger:    logger,
    }, nil
}

func (c *EventConsumer) Start(ctx context.Context, workerCount int) error {
    var wg sync.WaitGroup
    
    // Start multiple workers for parallel processing
    for i := 0; i < workerCount; i++ {
        wg.Add(1)
        go func(workerID int) {
            defer wg.Done()
            c.processEvents(ctx, workerID)
        }(i)
    }
    
    wg.Wait()
    return nil
}

func (c *EventConsumer) processEvents(ctx context.Context, workerID int) {
    for {
        select {
        case <-ctx.Done():
            return
        default:
            // Fetch and process events
            fetches := c.client.PollFetches(ctx)
            if fetches.IsEmpty() {
                continue
            }
            
            fetches.EachPartition(func(p kgo.FetchTopicPartition) {
                for _, record := range p.Records {
                    if err := c.processRecord(ctx, record); err != nil {
                        c.logger.Printf("Worker %d: Failed to process record: %v", workerID, err)
                    }
                }
            })
        }
    }
}
```

### 2. Vertical Scaling

**Resource Allocation:**
```yaml
# kubernetes-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: events-processor
spec:
  replicas: 10
  selector:
    matchLabels:
      app: events-processor
  template:
    metadata:
      labels:
        app: events-processor
    spec:
      containers:
      - name: events-processor
        image: lago/events-processor:latest
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        env:
        - name: WORKER_COUNT
          value: "8"
        - name: BATCH_SIZE
          value: "1000"
        - name: MAX_MEMORY_USAGE
          value: "3Gi"
```

This comprehensive Event Processing & Aggregation Engine document provides the complete technical implementation details for building a scalable, high-performance billing event processing system. The architecture supports millions of events per second while maintaining data consistency and providing real-time analytics capabilities.