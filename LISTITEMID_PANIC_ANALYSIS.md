# Complex Panic Vulnerability in listItemID Function

## Location
`iter.go`, line 277-279, in the `listItemID` function and its call sites on lines 64, 169, and 172

## Issue Description
The `listItemID` function uses reflection to extract the ID field from a generic type, but it doesn't handle the case where the value could be nil or a zero value. This can cause a **runtime panic** under specific pagination scenarios.

## The Vulnerable Code

```go
func listItemID[T any](x T) string {
	return reflect.ValueOf(x).Elem().FieldByName("ID").String()
}
```

Called from `v1List.next()`:
```go
func (it *v1List[T]) next() bool {
	if len(it.Data) == 0 && it.HasMore && !it.listParams.Single {
		// determine if we're moving forward or backwards in paging
		if it.listParams.EndingBefore != nil {
			it.listParams.EndingBefore = String(listItemID(it.cur))  // LINE 169 - PANIC HERE
			it.formValues.Set(EndingBefore, *it.listParams.EndingBefore)
		} else {
			it.listParams.StartingAfter = String(listItemID(it.cur))   // LINE 172 - PANIC HERE
			it.formValues.Set(StartingAfter, *it.listParams.StartingAfter)
		}
		it.getPage()
	}
	// ...
}
```

## Multiple Panic Scenarios

### Scenario 1: Nil Pointer Panic
If `x` is a nil pointer (e.g., `T` is `*Customer` and `x` is `nil`):

```go
reflect.ValueOf(nil).Elem()  // PANIC: reflect: call of reflect.Value.Elem on zero Value
```

### Scenario 2: Non-Pointer Type Panic
If `x` is not a pointer type:

```go
reflect.ValueOf(Customer{}).Elem()  // PANIC: reflect: call of reflect.Value.Elem on struct Value
```

### Scenario 3: Missing ID Field
If the type doesn't have an "ID" field:
```go
reflect.ValueOf(x).Elem().FieldByName("ID").String()
// Returns "<invalid Value>" instead of panicking, but this is still incorrect
```

## When Can This Occur?

### Critical Path: Empty First Page with HasMore=true

This is rare but possible if the Stripe API returns:
```json
{
  "data": [],
  "has_more": true
}
```

**Step-by-step breakdown:**

1. User calls `newV1List()` which creates iterator
2. `iter.getPage()` is called, fetches first page with 0 items
3. `it.Data` is empty slice `[]`
4. `it.cur` remains as zero value of type `T`
5. User calls `next()` which checks `len(it.Data) == 0 && it.HasMore`
6. Since both are true, it tries to paginate using `listItemID(it.cur)`
7. **PANIC:** If `T` is `*Customer`, then `it.cur` is `nil`, and `Elem()` panics

### Additional Vulnerable Case: Backwards Pagination

When using `EndingBefore` for backwards pagination:
```go
params := &stripe.CustomerListParams{}
params.EndingBefore = stripe.String("cus_xyz")  // Start backwards pagination
```

If the user's first `EndingBefore` param points to an item where everything before it has been deleted, you might get an empty page with `has_more: true`, triggering the same panic.

## Real-World Trigger Conditions

1. **Race condition with deletions:** 
   - User starts pagination
   - All items on first page get deleted between setting up params and making request
   - API returns empty page but `has_more: true` because more pages exist

2. **Filtering edge case:**
   - Using complex filters that temporarily return 0 results on first page
   - But `has_more: true` because later pages match

3. **API quirk/bug:**
   - Stripe API returning `has_more: true` with empty data (shouldn't happen but defensive coding is important)

## Impact

### Severity: **HIGH**
- **Crash:** Causes immediate program panic/crash
- **Unrecoverable:** No way to catch this in user code since it's internal to the iterator
- **Data loss risk:** If this happens mid-transaction, could lose work
- **Production impact:** Could take down services unexpectedly

### Affected Code Paths
1. Any `List()` operation that returns empty first page with `has_more: true`
2. Any `List()` operation with backwards pagination (`EndingBefore`)
3. Both `v1List[T]` and old `Iter` implementations are affected

## Reproduction Test Case

```go
func TestListItemIDPanicOnNil(t *testing.T) {
	// This would panic with the current implementation
	defer func() {
		if r := recover(); r != nil {
			t.Logf("Caught expected panic: %v", r)
		} else {
			t.Error("Expected panic but didn't get one")
		}
	}()

	var nilCustomer *stripe.Customer
	_ = listItemID(nilCustomer)  // This should panic
}

func TestListItemIDPanicOnZeroValue(t *testing.T) {
	defer func() {
		if r := recover(); r != nil {
			t.Logf("Caught expected panic: %v", r)
		}
	}()

	// If T were not a pointer, this would also panic
	customer := stripe.Customer{}
	_ = listItemID(customer)  // Would panic: Elem() on non-pointer
}
```

## Recommended Fix

### Solution 1: Check for nil before calling Elem()

```go
func listItemID[T any](x T) string {
	v := reflect.ValueOf(x)
	
	// Check if x is nil
	if !v.IsValid() || (v.Kind() == reflect.Ptr && v.IsNil()) {
		return ""  // or panic with a better error message
	}
	
	// Check if we can call Elem()
	if v.Kind() != reflect.Ptr {
		// Get the ID field directly for non-pointer types
		if idField := v.FieldByName("ID"); idField.IsValid() {
			return idField.String()
		}
		return ""
	}
	
	elem := v.Elem()
	if !elem.IsValid() {
		return ""
	}
	
	idField := elem.FieldByName("ID")
	if !idField.IsValid() {
		return ""
	}
	
	return idField.String()
}
```

### Solution 2: Better guard in next() function

```go
func (it *v1List[T]) next() bool {
	if len(it.Data) == 0 && it.HasMore && !it.listParams.Single {
		// Only attempt pagination if we have a valid current item
		// This prevents panic when it.cur is nil/zero value
		v := reflect.ValueOf(it.cur)
		if !v.IsValid() || (v.Kind() == reflect.Ptr && v.IsNil()) {
			// Can't paginate without a cursor, fetch first page
			it.getPage()
		} else {
			// Normal pagination logic
			if it.listParams.EndingBefore != nil {
				it.listParams.EndingBefore = String(listItemID(it.cur))
				it.formValues.Set(EndingBefore, *it.listParams.EndingBefore)
			} else {
				it.listParams.StartingAfter = String(listItemID(it.cur))
				it.formValues.Set(StartingAfter, *it.listParams.StartingAfter)
			}
			it.getPage()
		}
	}
	// ... rest of function
}
```

### Solution 3: Use type constraints

```go
// Define a constraint that requires an ID field
type HasID interface {
	GetID() string
}

// Update function signature
func listItemID[T HasID](x T) string {
	if x == nil {
		return ""
	}
	return x.GetID()
}
```

## Additional Notes

The same issue exists in the older `Iter` type's `Next()` method (line 64), which uses interface{} instead of generics:

```go
func (it *Iter) Next() bool {
	if len(it.values) == 0 && it.meta.HasMore && !it.listParams.Single {
		if it.listParams.EndingBefore != nil {
			it.listParams.EndingBefore = String(listItemID(it.cur))  // SAME ISSUE
		}
		// ...
	}
	// ...
}
```

## Risk Assessment

- **Likelihood:** Low to Medium (requires specific API response patterns)
- **Impact:** High (application crash)
- **Detectability:** High (immediate panic with stack trace)
- **Overall Risk:** Medium-High

## Recommended Priority
**P1/Critical** - This should be fixed in the next release as it can cause production outages in edge cases.
