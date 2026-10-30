# Python Interview Quick Reference

| Question | Answer |
|---|---|
| 1. What is Binary Search? | Binary search finds an element in a sorted list by repeatedly dividing the search range in half. Time complexity: `O(log n)`. |
| 2. Binary Search — Python code | `def binary_search(arr, target):`<br>`    low, high = 0, len(arr) - 1`<br>`    while low <= high:`<br>`        mid = (low + high) // 2`<br>`        if arr[mid] == target:`<br>`            return mid`<br>`        elif arr[mid] < target:`<br>`            low = mid + 1`<br>`        else:`<br>`            high = mid - 1`<br>`    return -1` |
| 3. What is the Fibonacci series? | A sequence where each number is the sum of the previous two numbers: `0, 1, 1, 2, 3, 5, 8, 13...` |
| 4. Fibonacci — Python code | `a, b = 0, 1`<br>`for _ in range(10):`<br>`    print(a, end=" ")`<br>`    a, b = b, a + b` |
| 5. What is a decorator? | A decorator wraps a function to add or modify behavior without changing the original function's code. |
| 6. Decorator — Python code | `def my_decorator(func):`<br>`    def wrapper():`<br>`        print("Before function")`<br>`        func()`<br>`        print("After function")`<br>`    return wrapper`<br><br>`@my_decorator`<br>`def hello():`<br>`    print("Hello!")`<br><br>`hello()` |
| 7. Where are decorators useful in DevOps? | Logging, execution-time measurement, retry logic, authentication and monitoring function calls. |
| 8. What does `@my_decorator` mean? | It is equivalent to `hello = my_decorator(hello)` when applied to the `hello()` function. |