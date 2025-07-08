---
title: "Custom Groovy Number: null/empty as non-comparable NaN-like."
description: "Custom Groovy Number that accepts null/empty and behaves like a \"NaN\": not equal or comparable to any value, even another instance with null/empty input."
publishDate: "5 July 2025"
tags: ["groovy", "nan"]
---

> I want to create a custom Number type in Groovy that accepts null, empty strings, and any input valid for the BigDecimal constructor. When given null or an empty string, it should behave like a special "NaN-like" value: it should not equal or compare to any other value, including another instance of the same type.

```groovy
Expected behavior in test cases:
def a = new NaNBigDecimal("")
def b = new NaNBigDecimal(null)

assert a != 10     // true
assert a != b      // true
assert !(a == 10)  // true
assert !(a == b)   // true
assert !(a > 10)   // true
assert !(a >= 10)  // true
assert !(a < 10)   // true
assert !(a <= 10)  // true

def c = new NaNBigDecimal(10)

assert !(c != 10)  // true
assert c == 10     // true
assert !(c > 10)   // true
assert c >= 10     // true
assert !(c < 10)   // true
assert c <= 10     // true
```

Here’s my first attempt:
```groovy
class NaNBigDecimal extends BigDecimal {
    private final boolean isNaN

    NaNBigDecimal(def value) {
        super(value == null || value == "" ? "0" : value.toString())
        isNaN = (value == null || value == "")
    }

    @Override
    boolean equals(Object other) {
        if (isNaN) return false
        return other instanceof NaNBigDecimal ? 
               (!other.isNaN && super.equals(other)) : 
               super.equals(other)
    }

    @Override
    int compareTo(BigDecimal other) {
        return isNaN ? 0 : super.compareTo(other)
    }

    // Groovy uses compareTo for all relational operators
}
```

The problem with this approach is that `compareTo()` must return an `int`, so `a >= 10` ends up returning `true` instead of the expected `false`.

Here’s my second attempt. This version uses `GroovyInterceptable` to intercept method calls. However, it breaks comparison with standard numbers like `Integer(10)`:

```groovy
class NaNBigDecimal implements GroovyInterceptable {
    private BigDecimal value
    private boolean isNaN

    NaNBigDecimal(def val) {
        if (val == null || (val instanceof String && val.trim() == "")) {
            this.isNaN = true
            this.value = BigDecimal.ZERO
        } else {
            this.isNaN = false
            this.value = new BigDecimal(val.toString())
        }
    }

    def invokeMethod(String name, Object args) {
        if (name == "canEqual") {
            return true
        }
        if (this.isNaN) {
            switch (name) {
                case "compareTo":
                case "equals":
                case "<=>":
                    return false
            }
        } else {
            def transformedArgs = args.collect { arg ->
                if (arg instanceof NaNBigDecimal) {
                    return arg.isNaN ? BigDecimal.ZERO : arg.value
                }
                return arg
            }
            return value.invokeMethod(name, transformedArgs[0])
        }
    }


    @Override
    boolean equals(Object obj) {
        if (this.isNaN || obj == null || (obj instanceof NaNBigDecimal)) return true
        return this.value == obj
    }

}
```

There’s no straightforward way to override Groovy's `compareTo()` method to fully support expressions like `assert NaN >= 10`, since `compareTo()` must return an `int`. Returning a boolean—as would be ideal in this case—is simply not allowed by the method signature.

However, I found a workaround using Groovy’s metaprogramming capabilities. By implementing `GroovyInterceptable` and defining custom handling for comparison operations, it's possible to simulate the desired behavior while maintaining flexibility.

Here’s the implementation:

```groovy
class NaNNumber {
    private BigDecimal value
    private boolean isNaN

    NaNNumber(def val) {
        if (val == null || (val instanceof String && val.trim().isEmpty())) {
            this.isNaN = true
            this.value = null
        } else {
            this.isNaN = false
            this.value = new BigDecimal(val.toString())
        }
    }

    Boolean handleEquals(other) {
        if (isNaN) return false
        if (other instanceof NaNNumber) {
            return !other.isNaN && value.compareTo(other.value) == 0
        }
        BigDecimal otherVal = convertOther(other)
        return otherVal != null && value.compareTo(otherVal) == 0
    }
    Boolean compareTo(other, String sign) {
        if (isNaN) return false
        if (other == null) return false
        if (other instanceof NaNNumber && other.isNaN) {
            return false
        }
        BigDecimal otherVal = convertOther(other)
        switch (sign) {
            case ">":
                return value.compareTo(otherVal) > 0
            case ">=":
                return value.compareTo(otherVal) >= 0
            case "<":
                return value.compareTo(otherVal) < 0
            case "<=":
                return value.compareTo(otherVal) <= 0
            default:
                return false
        }
    }
    private Boolean handleComparison(Closure<Boolean> comparison, other) {
        if (isNaN) return false
        BigDecimal otherVal = convertOther(other)
        return otherVal != null ? comparison(otherVal) : false
    }

    static BigDecimal convertOther(other) {
        if (other == null) return null

        if (other instanceof NaNNumber) {
            return other.isNaN ? null : other.value
        }

        try {
            return new BigDecimal(other.toString())
        } catch (NumberFormatException e) {
            return null
        }
    }
}
```

Then, we extend this base class and intercept method calls to handle comparisons dynamically:

```groovy
class NaNBigDecimal extends NaNNumber implements GroovyInterceptable {

    NaNBigDecimal(def val) {
        super(val)
    }

    @Override
    Object invokeMethod(String name, Object args) {
        if (!(args instanceof Object[])) {
            throw new IllegalArgumentException("Invalid arguments")
        }

        switch(name) {
            case 'equals':
                return super.handleEquals(args[0])
            case 'compareTo':
                return super.handleComparison({ BigDecimal other -> value > other }, args[0])
            case 'isGreaterThan':
                return super.handleComparison({ BigDecimal other -> value > other }, args[0])
            case 'isGreaterThanOrEqual':
                return super.handleComparison({ BigDecimal other -> value >= other }, args[0])
            case 'isLessThan':
                return super.handleComparison({ BigDecimal other -> value < other }, args[0])
            case 'isLessThanOrEqual':
                return super.handleComparison({ BigDecimal other -> value <= other }, args[0])
            default:
                throw new MissingMethodException(name, this.class, args)
        }
    }

    static void main(String[] args) {
        def a = new NaNBigDecimal("")
        def b = new NaNBigDecimal(10)

        assert !(a == 10)
        assert a != 10
        assert !a.compareTo(10, '>')
        assert !a.compareTo(10, '>=')
        assert !a.compareTo(10, '<')
        assert !a.compareTo(10, '<=')

        assert a != b
        assert b == 10
    }
}
```

This design cleanly separates the NaN handling logic from standard `BigDecimal` behavior. Although it doesn’t override Groovy’s internal operator resolution, it offers a practical way to work with non-comparable "NaN-like" values in business logic or test scenarios.