
# Nxxxx — Proposal for unnamed function expressions in C
**Author:** Thiago R Adams  
**Date:** 2025-07-01  
**Project:** ISO/IEC JTC 1/SC 22/WG 14  
**Title:** Unnamed function expression


## History
- draft-2 grammar fixes

## Abstract

This proposal introduces *unnamed function expression* into the C language, providing a syntax for defining unnamed functions where a function pointer is expected. *Unnamed function expression* allow programmers to define small, local functions without requiring separate file-scope declarations. 

---

## 1. Background

Modern programming increasingly favors *local reasoning*: keeping logic close to its point of use improves clarity, maintainability, and reduces cognitive load. C currently lacks a mechanism to define functions at the point of use. Programmers must declare static or external functions at file scope, which leads to:
- Code duplication for similar small callbacks.
- Reduced code locality, forcing readers to jump between unrelated parts of a source file.
- Cluttering of file/global scope with functions and structs used only once.

In other languages, notably C++ (lambdas) the ability to define small, unnamed functions at the point of use is well-established. However, those features often rely on capturing variables, which increases complexity. Since C prioritizes simplicity and performance, introducing unnamed functions **without captures** offers most of the benefits with minimal cost and no hidden memory management.

---

## 2. Motivation

Many standard C library functions (e.g., `qsort`, `bsearch`) or common APIs rely on callbacks. Today, using them requires extra boilerplate. For example:

```c
static int cmp(const void *a, const void *b) {
    return (*(const int*)a - *(const int*)b);
}
qsort(arr, n, sizeof(int), cmp);
````

With unnamed function expressions, the programmer could write:

```c
  qsort(arr, 20, sizeof(int), (int (const void *a, const void *b) {
   return (*(const int*)a - *(const int*)b);
  });
```

This reduces friction for developers, aligns C with modern programming practices, and improves code clarity without changing the fundamental simplicity of the language.

---

## 3. Syntax and Semantics

### 3.1 Syntax

An unnamed function expression has a similar syntax of compound literal, except, that the type in the *unnamed-function-expression*/ must be a function type, while this was never allowed in compount literal.

``` 
postfix-expression: 
   ...      
   unnamed-function-expression     
  
unnamed-function-expression:
   ( attribute-specifier-sequence opt declaration-specifiers abstract-declarator ) function-body  
 ```


### 3.2 Semantics

* The address of the unnamed function expression remains valid for the lifetime of the program.

* The resulting function pointer is compatible with existing function pointer types.

* Re-evaluating the same unnamed function expression expression may produce the same or different addresses at the compiler's discretion, but the behavior of the code must remain consistent.

* Evaluating an unnamed functions expression yields a pointer to a unique, unnamed internal function with the specified signature and body.

  
```c
int main() { 
  //error not lvalue
  auto p = &(void (void} *p)){};
}
```

* The unnamed function expression behaves as if it were defined as a `static` function at file scope, but it is unnamed and only accessible through the returned pointer.

* The rules for the scope of the return and parameters follow the existing rules. For instance
  
```c
int main()
{
  (struct X {int i;}  (struct Y {int i;})){  }();
   struct X x; //OK
   struct Y y; //ERROR
}
```

* The function-body of the unnamed functions creates a block scope with some particularities:
 
* Labels declared inside of the body of the unnamed function have function scope relative to the unnamed function.

* Labels from outer scope are not accessible inside unnamed function and vice versa. This makes impossible to jump into or out of the unnamed function.

* Unnamed functions body have access to the identifier of the out scope (except labels)

* Any usage of memory of the objects of the out scope are forbidden.

```c
int main() {
  int i;
  (void (void)){
    i = 1; //error
    int k =i; //error    
  };
}
```

* Using object identifiers in expressions that are not evaluated are fine.
```c
int main() {
  int i;
  (void (void)){
    int j = sizeof(i); //ok
    typeof(i) k; //ok
  };
}
```

* Object living on the file scope can be used normally.

* constexpr objects could be used by creating a clone (capture) inside the function body scope. However, not sure if this is useful. file scope constants can be used normally.




## 5. Examples

### 5.1 Sorting integers

```c
int arr[] = {3, 1, 2};
  qsort(arr, 3, sizeof(int), (int (const void *a, const void *b)) {
    return (*(const int*)a - *(const int*)b);
  });
```


## 6. Rationale

This proposal explicitly avoids variable capture to:

* Stay true to C’s philosophy of explicit and minimal runtime mechanisms.
* Avoid introducing closure allocation or hidden context structures, which would require significant compiler and ABI changes.
* Keep unnamed function expressions a simple syntactic convenience rather than a semantic extension requiring runtime support.

By restricting unnamed function expressions to pure parameter- and global-only access, they map directly to existing function pointer mechanisms, making them feasible to implement in current compilers without affecting performance or portability.

---

## 7. Compatibility and Impact

* This feature does not break any existing valid C programs, since compound literal objects of type function cannot be created in the current C version. 
* Compilers supporting unnamed function expressions must ensure that internally generated symbols do not clash with user-defined symbols.
* No changes are needed to existing runtime environments or library ABIs.

---

## 8. Implementation Considerations

unnamed function expressions can be implemented by the compiler as:

* Anonymous `static` functions with unique internal names.
* Emission of a unique function symbol in the object file.

Since there is no capture, unnamed function expressions do not require closures or heap allocations, and the calling convention is identical to regular functions.
  
There is an implementation at Cake transpiler that is not 100% complete yet and scope rules are missing.
http://thradams.com/cake/playground.html  

---

## 9. References

* ISO/IEC 9899:2024 (C23), Programming Languages - C.

---

## 10. Acknowledgements
