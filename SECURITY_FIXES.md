# Recommended Security Fixes

This document provides specific code fixes for the critical and high-priority issues identified in the diagnostic report.

## Critical Issue #1: Reentrancy in Balance Updates

### Problem
Balance is updated before external calls, creating potential for reentrancy attacks.

**Location**: `contracts/wallet/msgs/any.fc:24`

### Current Code
```func
storage::balance += ton_amount; 
storage::save();

reserves::exact((ctx.at(BALANCE) - ctx.at(MSG_VALUE)) + storage_fees() + ton_amount);
msgs::send_simple(
    0, 
    storage::owner_address, 
    any::transfer_notification(
        ton_amount, 
        ctx.at(SENDER), 
        either_forward_payload
    ),
    CARRY_ALL_BALANCE | BOUNCE_IF_FAIL
);
```

### Recommended Fix
```func
// Reserve tokens first, then update balance after successful external calls
reserves::exact((ctx.at(BALANCE) - ctx.at(MSG_VALUE)) + storage_fees() + ton_amount);

// Send transfer notification first
msgs::send_simple(
    0, 
    storage::owner_address, 
    any::transfer_notification(
        ton_amount, 
        ctx.at(SENDER), 
        either_forward_payload
    ),
    CARRY_ALL_BALANCE | BOUNCE_IF_FAIL
);

// Update balance only after successful external interactions
storage::balance += ton_amount; 
storage::save();
```

## High Priority Issue #1: Code Duplication

### Problem
Duplicate validation logic in `ton_transfer` and `ft::transfer` operations.

### Recommended Solution
Create shared validation function:

```func
;; Add to common/helpers.fc
(int) validate_transfer_params(int amount, slice refund_address, slice payload) inline {
    throw_unless(error::low_amount, amount > 0);
    throw_unless(error::invalid_address, refund_address.preload_uint(2) == params::addr_std_type);
    throw_unless(error::insufficient_gas, ctx.at(MSG_VALUE) > amount);
    throw_unless(error::invalid_body, payload.slice_bits() >= 1);
    return (true);
}
```

## High Priority Issue #2: Safe Math for Balance Operations

### Problem
Potential for integer underflow in balance operations.

**Location**: `contracts/wallet/msgs/owner.fc:15`

### Current Code
```func
storage::balance -= jetton_amount;
throw_unless(error::low_balance, storage::balance >= 0);
```

### Recommended Fix
```func
throw_unless(error::low_balance, storage::balance >= jetton_amount);
storage::balance -= jetton_amount;
```

## Dependency Security Updates

### Required Updates
Run the following commands to fix dependency vulnerabilities:

```bash
npm audit fix
npm audit fix --force  # For breaking changes if needed
```

### Manual Updates Required
1. Update `@ton/blueprint` to latest version
2. Replace deprecated `axios` usage
3. Update `form-data` to secure version

## Additional Security Enhancements

### 1. Enhanced Address Validation
```func
;; Add to common/helpers.fc
(int) validate_address_extended(slice addr) inline {
    int addr_type = addr.preload_uint(2);
    return (addr_type == params::addr_std_type) | (addr_type == 3); ;; Support addr_var$11 too
}
```

### 2. Dynamic Gas Calculation
```func
;; Add to common/gas.fc
(int) calculate_gas_for_operation(int base_gas, int current_price) inline {
    ;; Adjust gas based on current network conditions
    return base_gas + (current_price / 1000000); ;; Simple adjustment
}
```

### 3. Atomic Balance Operations
```func
;; Add mutex-like protection for balance operations
global int balance_lock;

() acquire_balance_lock() impure inline {
    throw_unless(error::concurrent_operation, balance_lock == 0);
    balance_lock = 1;
}

() release_balance_lock() impure inline {
    balance_lock = 0;
}
```

## Testing for Security Fixes

Add these test cases after implementing fixes:

```typescript
it('should prevent reentrancy attacks', async () => {
    // Test implementation needed
});

it('should handle concurrent balance operations safely', async () => {
    // Test implementation needed  
});

it('should validate extended address formats', async () => {
    // Test implementation needed
});
```

## Deployment Checklist

Before deploying with security fixes:

- [ ] All security fixes implemented
- [ ] Dependencies updated to secure versions
- [ ] New test cases added and passing
- [ ] Code review completed
- [ ] Professional security audit completed
- [ ] Documentation updated

---

*Apply these fixes incrementally and test thoroughly before proceeding to the next fix.*