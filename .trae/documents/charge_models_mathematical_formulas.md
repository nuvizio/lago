# Charge Models - Mathematical Formulas and Algorithms

## 1. Standard Charge Model

### 1.1 Basic Formula

```
Total Amount = Units × Unit Price
```

### 1.2 Unit Amount Calculation

```
Unit Amount = Total Amount ÷ Total Units
```

### 1.3 Projected Amount

```
Projected Amount = Projected Units × Unit Price
```

### 1.4 Implementation Details

```ruby
def compute_amount
  (units * BigDecimal(properties["amount"]))
end

def compute_projected_amount
  projected_units * BigDecimal(properties["amount"])
end

def unit_amount
  total_units = aggregation_result.full_units_number || units
  return 0 if total_units.zero?
  
  compute_amount / total_units
end
```

## 2. Graduated Charge Model

### 2.1 Tier Processing Algorithm

```
For each tier in graduated_ranges:
  1. Calculate tier capacity: capacity = to_value - from_value + 1
  2. Determine units in tier: units_in_tier = min(remaining_units, capacity)
  3. Calculate tier amount: tier_amount = (units_in_tier × per_unit_amount) + flat_amount
  4. Update remaining units: remaining_units -= units_in_tier
  5. Add to total: total_amount += tier_amount
  
  Break if remaining_units <= 0
```

### 2.2 Mathematical Formula

```
Total Amount = Σ(tier_units[i] × per_unit_amount[i] + flat_amount[i])
```

### 2.3 Unit Amount Calculation

```
Unit Amount = Total Amount ÷ Total Units (including full_units_number)
```

### 2.4 Projected Amount Algorithm

```
remaining_units_to_price = projected_units
total_amount = 0
priced_units_count = 0

For each range:
  range_to = to_value || ∞
  tier_capacity = range_to - priced_units_count
  units_in_this_tier = min(remaining_units_to_price, tier_capacity)
  
  if units_in_this_tier > 0:
    range_amount = (units_in_this_tier × per_unit_amount) + flat_amount
    total_amount += range_amount
    remaining_units_to_price -= units_in_this_tier
    priced_units_count += units_in_this_tier
  
  Break if remaining_units_to_price <= 0
```

### 2.5 Implementation Details

```ruby
def ranges
  properties["graduated_ranges"]&.map(&:with_indifferent_access)
end

def compute_amount
  amount_details.fetch(:graduated_ranges).sum { |e| e[:total_with_flat_amount] }
end

def compute_projected_amount
  return BigDecimal("0") if projected_units.zero?
  
  remaining_units_to_price = projected_units
  total_amount = BigDecimal("0")
  priced_units_count = BigDecimal("0")
  
  ranges.each do |range|
    range_to = range[:to_value] ? BigDecimal(range[:to_value].to_s) : Float::INFINITY
    tier_capacity = range_to - priced_units_count
    units_in_this_tier = [remaining_units_to_price, tier_capacity].min
    
    if units_in_this_tier > 0
      range_per_unit = BigDecimal(range[:per_unit_amount] || 0)
      range_flat_amount = BigDecimal(range[:flat_amount] || 0)
      range_amount = (units_in_this_tier * range_per_unit) + range_flat_amount
      total_amount += range_amount
      remaining_units_to_price -= units_in_this_tier
      priced_units_count += units_in_this_tier
    end
    break if remaining_units_to_price <= 0
  end
  
  total_amount
end
```

## 3. Volume Charge Model

### 3.1 Range Matching Algorithm

```
Find matching range where:
  from_value <= number_of_units AND (to_value IS NULL OR number_of_units <= to_value)

If no matching range:
  Total Amount = 0

If matching range:
  Total Amount = (units × per_unit_amount) + flat_amount
```

### 3.2 Mathematical Formula

```
Total Amount = (Units × Per Unit Price) + Flat Fee
```

### 3.3 Unit Amount Calculation

```
Unit Amount = Total Amount ÷ Number of Units
```

### 3.4 Projected Amount Algorithm

```
Find range_for_projection where:
  from_value <= projected_units.ceil AND (to_value IS NULL OR projected_units <= to_value)

If range_for_projection exists:
  per_unit_price = per_unit_amount from range
  flat_fee = flat_amount from range
  Projected Amount = (projected_units × per_unit_price) + flat_fee
Else:
  Projected Amount = 0
```

### 3.5 Implementation Details

```ruby
def ranges
  properties["volume_ranges"]&.map(&:with_indifferent_access)&.sort_by { |h| h[:from_value] }
end

def compute_amount
  return 0 if units.zero?
  
  per_unit_total_amount + flat_unit_amount
end

def matching_range
  @matching_range ||= ranges.find do |range|
    range[:from_value] <= number_of_units&.ceil && (!range[:to_value] || number_of_units <= range[:to_value])
  end
end

def compute_projected_amount
  return BigDecimal("0") if projected_units.zero?
  
  range_for_projection = ranges.find do |range|
    range[:from_value] <= projected_units.ceil && (!range[:to_value] || projected_units <= range[:to_value])
  end
  
  return BigDecimal("0") unless range_for_projection
  
  per_unit_price = BigDecimal(range_for_projection[:per_unit_amount] || 0)
  flat_fee = BigDecimal(range_for_projection[:flat_amount] || 0)
  
  (projected_units * per_unit_price) + flat_fee
end
```

## 4. Package Charge Model

### 4.1 Package Counting Algorithm

```
Paid Units = max(0, Total Units - Free Units)
Package Count = ceil(Paid Units ÷ Package Size)
Total Amount = Package Count × Price per Package
```

### 4.2 Mathematical Formula

```
Total Amount = ceil(max(0, Units - Free Units) ÷ Package Size) × Package Price
```

### 4.3 Unit Amount Calculation

```
Unit Amount = Total Amount ÷ Paid Units
```

### 4.4 Projected Amount Algorithm

```
proj_paid_units = projected_units - free_units
if proj_paid_units <= 0:
  Projected Amount = 0
else:
  proj_package_count = ceil(proj_paid_units ÷ package_size)
  Projected Amount = proj_package_count × package_price
```

### 4.5 Implementation Details

```ruby
def compute_amount
  return 0 if paid_units.negative?
  
  package_count = paid_units.fdiv(per_package_size).ceil
  package_count * per_package_unit_amount
end

def compute_projected_amount
  return 0 if projected_units.zero?
  
  proj_paid_units = projected_units - BigDecimal(free_units.to_s)
  return 0 if proj_paid_units <= 0
  
  proj_package_count = (proj_paid_units / BigDecimal(per_package_size.to_s)).ceil
  proj_package_count * per_package_unit_amount
end

def paid_units
  @paid_units ||= units - free_units
end

def free_units
  @free_units ||= properties["free_units"] || 0
end
```

## 5. Percentage Charge Model

### 5.1 Basic Algorithm (without min/max)

```
Free Units Value = min(Last Running Total, Free Units per Total Aggregation)
Paid Units = max(0, Total Units - Free Units Value)
Percentage Amount = Paid Units × Rate ÷ 100
Fixed Amount = max(0, Event Count - Free Events) × Fixed Amount per Event
Total Amount = Percentage Amount + Fixed Amount
```

### 5.2 Free Units Calculation

```
If free_units_per_events > 0 AND free_units_per_events < running_total_count:
  Free Units Value = running_total[free_units_per_events - 1]
Else if free_units_per_total_aggregation > 0:
  Free Units Value = min(last_running_total, free_units_per_total_aggregation)
Else:
  Free Units Value = 0

Free Events = min(free_units_per_events, events_below_threshold)
```

### 5.3 Per-Transaction Min/Max Algorithm

```
For each event:
  1. Apply free units (events and amount)
  2. Calculate event amount: (value × rate ÷ 100) + fixed_amount
  3. Apply min/max constraints:
     if min_amount exists AND event_amount < min_amount:
       event_amount = min_amount
     if max_amount exists AND event_amount > max_amount:
       event_amount = max_amount
  4. Add to total amount
```

### 5.4 Mathematical Formulas

**Basic Calculation:**
```
Total Amount = Σ(event_amount[i])
```

**Event Amount Calculation:**
```
event_amount = max(min_amount, min(max_amount, (value × rate ÷ 100) + fixed_fee))
```

### 5.5 Implementation Details

```ruby
def compute_amount
  return compute_amount_with_transaction_min_max if should_apply_min_max?
  
  compute_percentage_amount + compute_fixed_amount
end

def compute_percentage_amount
  return 0 if free_units_value > units
  
  (units - free_units_value) * rate / 100
end

def compute_fixed_amount
  return 0.0 if units.zero?
  return 0.0 if fixed_amount.nil?
  return 0.0 if free_units_count >= aggregation_result.count
  
  (aggregation_result.count - free_units_count) * fixed_amount
end

def compute_amount_with_transaction_min_max
  remaining_free_events = free_units_per_events
  remaining_free_amount = free_units_per_total_aggregation
  
  events_values.reduce(0) do |total_amount, event_value|
    value = event_value
    
    # Apply free units
    if remaining_free_events.positive? || remaining_free_amount.positive?
      remaining_free_events -= 1
      
      next 0 unless remaining_free_amount.positive?
      
      if remaining_free_amount > value
        remaining_free_amount -= value
        next 0
      else
        value -= remaining_free_amount
        remaining_free_amount = 0
        remaining_free_events = 0
      end
    end
    
    # Apply rate and fixed amount
    event_amount = (value * rate) / 100
    event_amount += fixed_amount
    
    # Apply min and max constraints
    event_amount = apply_min_max(event_amount)
    
    total_amount + event_amount
  end
end

def apply_min_max(amount)
  return per_transaction_min_amount if per_transaction_min_amount? && amount < per_transaction_min_amount
  return per_transaction_max_amount if per_transaction_max_amount? && amount > per_transaction_max_amount
  
  amount
end
```

## 6. Graduated Percentage Charge Model

### 6.1 Tier Processing Algorithm

```
For each tier in graduated_percentage_ranges:
  1. Calculate units in tier based on from_value and to_value
  2. Calculate percentage amount: units_in_tier × rate ÷ 100
  3. Add flat amount: percentage_amount + flat_amount
  4. Add to total amount
  
  Break if all units processed
```

### 6.2 Mathematical Formula

```
Total Amount = Σ((tier_units[i] × rate[i] ÷ 100) + flat_amount[i])
```

### 6.3 Unit Amount Calculation

```
Unit Amount = Total Amount ÷ Total Units
```

### 6.4 Implementation Details

```ruby
def ranges
  properties["graduated_percentage_ranges"]&.map(&:with_indifferent_access)
end

def compute_amount
  amount_details.fetch(:graduated_percentage_ranges).sum { |e| e[:total_with_flat_amount] }
end

def unit_amount
  total_units = aggregation_result.full_units_number || units
  return 0 if total_units.zero?
  
  compute_amount / total_units
end
```

## 7. Prorated Graduated Charge Model

### 7.1 Core Algorithm

```
Initialize:
  full_units = per_event_aggregation
  prorated_units = per_event_prorated_aggregation
  index = 0, overflow = 0, full_sum = 0, prorated_sum = 0, result_amount = 0

While (events to process OR overflow exists):
  1. Determine current range based on full_sum and overflow
  2. Handle overflow from previous iteration
  3. Process current event (if exists)
  4. Calculate overflow for next tier (if applicable)
  5. Apply prorated calculation to result_amount

Add flat amounts based on tiers traversed
```

### 7.2 Proration Coefficient

```
prorated_coefficient = prorated_value ÷ full_value
```

### 7.3 Overflow Calculation

```
If to_value is NULL:
  overflow = full_sum - from_value + 1
Else if full_sum >= to_value:
  overflow = full_sum - to_value
Else:
  overflow = full_sum - from_value + 1
```

### 7.4 Implementation Details

```ruby
def compute_amount
  full_units = per_event_aggregation_result.event_aggregation
  prorated_units = per_event_aggregation_result.event_prorated_aggregation
  
  index = 0
  overflow = 0
  full_sum = 0
  max_full_sum = 0
  prorated_sum = 0
  result_amount = 0
  
  return 0 if units.zero?
  
  while (index < prorated_units.count) || !overflow.zero?
    range = range(full_sum, overflow, full_units[index])
    
    # Handle overflow from previous iteration
    unless overflow.zero?
      prorated_sum += overflow * prorated_coefficient(prorated_units[index - 1], full_units[index - 1])
      
      if range[:to_value] && full_sum >= range[:to_value]
        overflow = full_sum - range[:to_value]
        prorated_sum -= overflow * prorated_coefficient(prorated_units[index - 1], full_units[index - 1])
        result_amount += prorated_sum * BigDecimal(range[:per_unit_amount])
        prorated_sum = 0
        next
      end
      
      overflow = 0
    end
    
    # Break if no more events and overflow handled
    break if prorated_units[index].nil?
    
    full_sum += full_units[index]
    max_full_sum = full_sum if full_sum > max_full_sum
    prorated_sum += prorated_units[index]
    
    index += 1
    
    next if skip_overflow_calculation?(full_sum, range[:to_value], range[:from_value])
    
    # Calculate overflow for next tier
    overflow = calculate_overflow(full_sum, range[:to_value], range[:from_value])
    prorated_sum -= overflow * prorated_coefficient(prorated_units[index - 1], full_units[index - 1])
    
    result_amount += prorated_sum * BigDecimal(range[:per_unit_amount])
    prorated_sum = 0
  end
  
  result_amount += prorated_sum * BigDecimal(range[:per_unit_amount])
  result_with_flat_amount(result_amount, full_sum, max_full_sum)
end

def prorated_coefficient(prorated_value, full_value)
  prorated_value.fdiv(full_value)
end

def calculate_overflow(full_sum, to_value, from_value)
  return full_sum - from_value + 1 if to_value.nil?
  
  if full_sum >= to_value
    full_sum - to_value
  else
    full_sum - from_value + 1
  end
end
```

## 8. Period Ratio and Proration

### 8.1 Period Ratio Calculation

```ruby
def calculate_period_ratio
  from_date = boundaries.charges_from_datetime.to_date
  to_date = boundaries.charges_to_datetime.to_date
  current_date = Time.current.to_date
  
  total_days = (to_date - from_date).to_i + 1
  charges_duration = boundaries.charges_duration || total_days
  
  return 1.0 if current_date >= to_date
  return 0.0 if current_date < from_date
  
  days_passed = (current_date - from_date).to_i + 1
  ratio = days_passed.fdiv(charges_duration)
  ratio.clamp(0.0, 1.0)
end
```

### 8.2 Projected Units Calculation

```
If period_ratio > 0:
  Projected Units = units ÷ period_ratio
Else:
  Projected Units = 0
```

### 8.3 Projected Amount Calculation

```
If current_amount > 0 AND period_ratio > 0:
  Projected Amount = current_amount ÷ period_ratio
Else:
  Projected Amount = 0
```

## 9. Currency Conversion and Precision

### 9.1 Amount to Cents Conversion

```
rounded_amount = amount.round(currency.exponent)
amount_cents = rounded_amount × currency.subunit_to_unit
precise_amount_cents = amount × currency.subunit_to_unit
```

### 9.2 Currency Conversion with Pricing Units

```
adjusted_amount = amount_cents × conversion_rate ÷ pricing_unit.subunit_to_unit
adjusted_unit_amount = unit_amount_cents × conversion_rate ÷ pricing_unit.subunit_to_unit
```

### 9.3 Precision Management

```ruby
# Dual precision storage
amount_cents: rounded_amount * currency.subunit_to_unit
precise_amount_cents: amount * currency.subunit_to_unit.to_d

# Currency conversion
adjusted_amount = amount_cents.to_d * conversion_rate / pricing_unit.subunit_to_unit
{
  amount_cents: adjusted_amount.round(currency.exponent) * currency.subunit_to_unit,
  precise_amount_cents: adjusted_amount * currency.subunit_to_unit.to_d,
  unit_amount_cents: adjusted_unit_amount * currency.subunit_to_unit,
  precise_unit_amount: adjusted_unit_amount
}
```

## 10. Error Handling and Edge Cases

### 10.1 Division by Zero Protection

```ruby
def unit_amount
  total_units = aggregation_result.full_units_number || units
  return 0 if total_units.zero?
  
  compute_amount / total_units
end
```

### 10.2 Negative Unit Handling

```ruby
# In fee creation
if amount_result.units.negative? || amount_result.amount.negative?
  amount_result.amount = amount_result.unit_amount = BigDecimal(0)
  amount_result.full_units_number = amount_result.units = BigDecimal(0)
end
```

### 10.3 Overflow and Infinity Handling

```ruby
# In graduated calculations
range_to = range[:to_value] ? BigDecimal(range[:to_value].to_s) : Float::INFINITY
```

### 10.4 Precision Error Handling

```ruby
def projected_units
  return BigDecimal("0") if units.nil? || units.zero?
  
  begin
    (period_ratio > 0) ? (units / BigDecimal(period_ratio.to_s)).round(2) : BigDecimal("0")
  rescue => e
    Rails.logger.error "Error calculating projected_units in #{self.class}: #{e.message}"
    BigDecimal("0")
  end
end
```

## 11. Performance Optimizations

### 11.1 Memoization Patterns

```ruby
# Cache expensive calculations
@paid_units ||= units - free_units
@matching_range ||= ranges.find { |range| condition }
@compute_amount_with_transaction_min_max ||= calculate_expensive_operation
```

### 11.2 Early Exit Conditions

```ruby
# Return early for zero cases
return 0 if units.zero?
return 0 if paid_units.negative?
return 0 if free_units_value > units
break if remaining_units_to_price <= 0
```

### 11.3 Efficient Range Processing

```ruby
# Use each_with_object for accumulation
ranges.each_with_object([]) do |range, amounts|
  amounts << calculate_range_amount(range)
  break amounts if range[:to_value].nil? || range[:to_value] >= units
end
```

This comprehensive mathematical foundation ensures accurate, efficient, and reliable pricing calculations across all charge models while maintaining precision and handling edge cases appropriately.