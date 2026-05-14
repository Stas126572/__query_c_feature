# ISO/IEC JTC1/SC22/WG14 Preliminary Proposal

**Document number:** Nxxxx (To be assigned by Convener)  
**Date:** 2026-05-14  
**Author:** Stas126572 (Independent Researcher)  
**Project Link:** https://github.com/Stas126572/__query_c_feature
**Email:** c_query_feature@mail.ru
**Target:** C2Y  

---

## Title: Introducing `__query_c_feature` Macro to Supersede Legacy Feature-Test Macros and Enable Hardware-Aware Code Inspection

### 1. Introduction and Rationale
The current mechanism for querying optional language features in ISO C relies heavily on negative conditional macros introduced in C11, such as `__STDC_NO_THREADS__`, `__STDC_NO_COMPLEX__`, and `__STDC_NO_ATOMICS__`. This approach introduces significant architectural drawbacks:

1. **Inverted Logic:** Developers are forced to write clumsy `#ifndef` chains to check for the *absence of absence*, reducing code readability.
2. **Poor Scalability:** As modern standards introduce heavy-weight features, continuing to add `__STDC_NO_...` macros for every new feature will severely pollute the preprocessor namespace and complicate cross-platform header design.
3. **Lack of Implementation Transparency:** Microcontroller and embedded systems developers frequently need to know *how* a feature is implemented. A binary "yes/no" macro cannot distinguish between a slow runtime software emulation and native instruction set architecture (ISA) support.

To resolve this technical debt, we propose introducing a new preprocessor query macro: `__query_c_feature`. This macro shifts the paradigm from binary negation to an explicit, tier-based capability query, allowing compilers to transparently signal hardware backing while providing a clean deprecation path for legacy macros.

---

### 2. Proposed Wording

**X.Y Intellectual code inspection macro `__query_c_feature`**

**Syntax**
```c
__query_c_feature ( identifier )
```

**Description**
The `__query_c_feature` macro is a functional-like built-in preprocessor operator that allows querying the availability, implementation characteristics, and hardware backing of specific language features or optional components. 

The expression `__query_c_feature(identifier)` shall expand to an integer constant expression.

**Constraints**
The identifier shall be one of the feature tokens defined in table [Features] or an implementation-defined token. If the identifier is not recognized by the translation phase, the macro expands to `0`.

**Semantics**
The integer value returned by `__query_c_feature` represents the support and execution tier of the queried feature:

* **`0`**: The feature is **not supported** by the implementation.
* **Values `>= 1`**: The feature is supported. The execution characteristics are defined below.

**Table [Features]: Initial Standard Tokens**


| Token | Meaning | Legacy Macro Superseded |
| :--- | :--- | :--- |
| `c_threads` | Support for ISO C execution threads (`<threads.h>`) | `__STDC_NO_THREADS__` |
| `c_complex` | Support for complex types and `<complex.h>` | `__STDC_NO_COMPLEX__` |
| `c_atomics` | Support for atomic operations (`<stdatomic.h>`) | `__STDC_NO_ATOMICS__` |

#### 2.1 Parameterized Token Rules for Bit-Precise Integers
If the queried identifier is related to bit-precise integer types (`_BitInt(N)`), the token format shall be `c_bitint_<N>`, where `N` is the literal decimal integer width:
* Example: `__query_c_feature(c_bitint_64)`

#### 2.2 Recommended Practice for Execution Tiers

*Recommended Practice*

Implementations are encouraged to map the returned integer values to the following approximate hardware and resource boundaries of the target abstract machine:

* **Tier 4 (Register Native):** The feature is executed entirely within the **general-purpose or vector registers (such as ZMM/YMM/XMM)** of the target processor. It requires no function calls, utilizes no local stack storage, and induces no environment traps.
* **Tier 3 (Stack/Inline Bounded):** The feature executes inline without any function calls or environment traps, but requires allocation of **local stack storage / multiple memory objects** to complete the operation.
* **Tier 2 (Library Software Emulation):** The feature is implemented via implicit or explicit **function calls to a runtime helper library**, executing within the standard software layer without inducing environment traps.
* **Tier 1 (Environment Trap Emulation):** The feature requires **hardware-level exception vectors, software traps, or operating system interrupts** to emulate missing processor capabilities.

---

### 3. Obsolescent Features

**6.11 Legacy feature-test macros**

1. The use of the conditional predefined macros `__STDC_NO_THREADS__`, `__STDC_NO_COMPLEX__`, and `__STDC_NO_ATOMICS__` is an obsolescent feature.

---

### 4. Implementation Feasibility and Proof of Concept
The proposed `__query_c_feature` macro introduces zero syntactic overhead for existing preprocessor backends. Compiler vendors can implement it similarly to existing built-in traits (like `__has_builtin`).

An example implementation logic inside an OO compiler preprocessor frontend (C++ code):
```cpp
int handle_query_c_feature(const std::string& token, const TargetInfo& target) {
    if (token.rfind("c_bitint_", 0) == 0) { 
        std::string width_str = token.substr(9); 
        int bit_width = std::stoi(width_str); 
        
        if (bit_width > target.maxSupportedBitIntWidth()) {
            return 0; 
        }
        
        if (target.hasNativeBitWidth(bit_width)) return 4;
        if (target.canInlineBitWidth(bit_width)) return 3;
        return 2; 
    }
    
    if (token == "c_complex") {
        if (!target.hasFPU()) return 1; 
        if (target.hasNativeComplexInstructions()) return 4;
        return 3; 
    }
    
    return 0; 
}
```

---

### 5. Backward Compatibility and Preprocessor Fallback Examples

Due to the fundamental evaluation rules of the ISO C preprocessor (6.10.1), any unrecognized identifier inside a preprocessor conditional expression evaluates to 0. This ensures that legacy compilers will naturally treat `__query_c_feature(...)` as 0 if it is not supported.

Developers can adopt the new macro while maintaining backward compatibility with older C11/C23 compilers using the following pattern:

```c
#ifndef __query_c_feature
  /* Fallback for legacy compilers */
  #if defined(__STDC_NO_ATOMICS__)
    #define MY_HAS_ATOMICS 0
  #else
    #define MY_HAS_ATOMICS 1 
  #endif
#else
  /* Modern approach enabling hardware-aware path */
  #if __query_c_feature(c_atomics) >= 4
    #define MY_HAS_ATOMICS 2 /* Hardware native path */
  #elif __query_c_feature(c_atomics) > 0
    #define MY_HAS_ATOMICS 1 /* Emulated path */
  #else
    #define MY_HAS_ATOMICS 0
  #endif
#endif
```
