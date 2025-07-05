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
            this.value = BigDecimal.ZERO // 或者 null，根据需求决定是否允许 value 为 null
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
                case "<=>": // Groovy 的太空船操作符对应的方法
                    return false
            }
        } else {
            // 特殊处理比较方法的参数
            def transformedArgs = args.collect { arg ->
                if (arg instanceof NaNBigDecimal) {
                    return arg.isNaN ? BigDecimal.ZERO : arg.value
                }
                return arg
            }

            // 委托给 BigDecimal 的方法实现
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