# How to use cutlass
## 1. Clone the Cutlass Library
```
git clone https://github.com/NVIDIA/cutlass.git
```

## 2. Modify the Source Code
open cutlass/include/cutlass/array.h, add a negative sign at line 1548, and remove a negative sign at line 1549.
```python
# origin
1547    if constexpr (N % 2) {
1548      half_t x = lhs[N - 1];
1549      __half lhs_val = -reinterpret_cast<__half const &>(x);
1550      result[N - 1] = reinterpret_cast<half_t const &>(lhs_val);
1551    }
# modified
1547    if constexpr (N % 2) {
1548      half_t x = -lhs[N - 1];
1549      __half lhs_val = reinterpret_cast<__half const &>(x);
1550      result[N - 1] = reinterpret_cast<half_t const &>(lhs_val);
1551    }

```    
> This modification is to avoid the "array.h(1549): error: no operator "-" matches these operands" error encountered during compilation, an error encountered when using the `cutlass::epilogue::thread::LinearCombinationSigmoid` method. This issue has already been submitted as a pull request to the Cutlass library: https://github.com/NVIDIA/cutlass/pull/916 but has not yet been fixed.