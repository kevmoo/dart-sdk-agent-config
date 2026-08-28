# Dart VM C++ Best Practices & Architectural Reference

A concise reference guide detailing C++ best practices, object model invariants, memory management rules, and code review standards used by the Dart VM team.

---

## 1. Object Model & GC Invariants

The Dart VM relies on a **precise, moving generational garbage collector** consisting of a parallel semispace scavenger for new-space and concurrent mark-sweep/compact for old-space.

### 1.1 Pointer Tagging
* **Small Integers (`Smi`)**: Immediate values tagged with `0` in the lowest bit (e.g. `(val << 1)`).
* **Heap Objects**: Tagged with `1` in the lowest bit. Objects are aligned on double-word boundaries (new-space objects are kept offset by 1 word from double-word alignment, allowing fast generational age checks).

### 1.2 `ObjectPtr` vs. `Handle` Discipline
* **`ObjectPtr` / `UntaggedObject*` (Raw Pointers)**:
  * Direct pointers into the GC-managed heap.
  * **Unsafe across safepoints**: Any allocation, function call that can enter a safepoint, or thread transition can move objects and invalidate raw pointers.
  * Never store an `ObjectPtr` in a local variable across operations that may allocate or trigger GC.
* **`Handle` / `ZoneHandle` (Indirect GC Pointers)**:
  * Safe references managed by the VM handle pool (e.g., `const String& str = String::Handle(zone, ...)`).
  * Automatically traced and updated by the GC during scavenge, mark-compact, and `become` phases.
  * Managed via `HANDLESCOPE(thread)` or allocated within the current `Zone`.

### 1.3 Direct Interior Pointers & `NoSafepointScope`
* If an algorithm extracts direct interior pointers into heap memory (e.g. `TypedData::DataAddr`, string byte buffers):
  * The entire sequence from address extraction to consumption **must** be enclosed in a `NoSafepointScope` or `NoOOBMessageScope`.
  * No allocations or safepoint transitions are permitted while holding interior pointers.

### 1.4 Write Barriers
* Direct stores into heap fields (`StorePointer`) must execute generational and incremental write barriers unless provably safe (e.g. writing Smis, constants, or fresh unshared objects).

---

## 2. Memory Allocation Taxonomy

Memory allocation in the VM is explicitly structured through base classes defined in `runtime/platform/allocation.h` and `runtime/vm/allocation.h`:

| Base Class | Allocation Scheme | Deallocation / Lifecycle |
| :--- | :--- | :--- |
| **`ValueObject`** | Stack-only (`DISALLOW_ALLOCATION()`) | Normal C++ stack unwinding. |
| **`StackResource`** / **`ThreadStackResource`** | Stack-only | Linked into a per-thread list; destructors run safely even during `longjmp` / exception unwinds. |
| **`ZoneObject`** | Arena (`Zone`) allocated | `delete` is `UNREACHABLE()`; all objects freed en masse when the `Zone` destructs. |
| **`MallocAllocated`** | C Heap | Explicit `dart::malloc` / `::free`. |
| **`AllStatic`** | Static utility class | Cannot be instantiated on stack or heap (`DISALLOW_ALLOCATION()`). |

---

## 3. Thread Execution States & Safepoints

Mutator threads operate under explicit states defined in `runtime/vm/heap/safepoint.h`:

* **States**:
  * `kThreadInVM`: Executing C++ VM code (not at safepoint).
  * `kThreadInGenerated`: Executing compiled Dart code (not at safepoint).
  * `kThreadInNative`: Executing external C/native code (at safepoint).
  * `kThreadInBlockedState`: Blocked on locks (at safepoint).
* **RAII State Transitions**:
  * `TransitionNativeToVM` / `TransitionVMToNative`
  * `TransitionToGenerated` / `TransitionVMToGenerated`
  * `TransitionToVM` / `TransitionToNative`
* **Concurrency**: Background tasks (markers, sweepers, JIT compilers) run as `ThreadPool::Task` instances rather than unmanaged threads.

---

## 4. Dialect & Core VM Conventions

* **No Standard STL in Core VM**: Standard C++ containers (`std::vector`, `std::unordered_map`) are avoided in `runtime/vm` to eliminate hidden heap allocations and keep binary sizes minimal. Use VM containers instead:
  * `GrowableArray<T>` / `ZoneGrowableArray<T>` / `MallocGrowableArray<T>`
  * `IntMap<T>`, `HashTable`, `BitVector`
* **Native Method Bindings**:
  * Defined via `DEFINE_NATIVE_ENTRY(Name, type_args_len, args_len)`.
  * Arguments extracted with `GET_NON_NULL_NATIVE_ARGUMENT(Type, name, arguments->NativeArgAt(idx))` or `GET_NATIVE_ARGUMENT`.
  * Errors signaled via `Exceptions::ThrowArgumentError(...)`, `Exceptions::ThrowRangeError(...)`, etc.
* **Macros & Assertions**:
  * Invariant enforcement: `ASSERT(...)`, `ASSERT_UNREACHABLE()`, `DEBUG_ASSERT(...)`, `COMPILE_ASSERT(...)`.
  * Runtime flags: `DEFINE_FLAG(type, name, default_value, "doc")`.
  * Rule of zero/five enforcement: `DISALLOW_COPY_AND_ASSIGN(...)`, `DISALLOW_ALLOCATION()`, `DISALLOW_IMPLICIT_CONSTRUCTORS(...)`.
* **Formatting**: Chromium style (`.clang-format`), 2-space indentation, 80-column line limit.

---

## 5. Reviewer Priorities & Standards (Gerrit)

Common review themes from VM owners and Gerrit reviewers:

1. **Safepoint Integrity**: Strict verification that no raw pointers cross potential GC safepoints or allocation sites.
2. **Virtual Integration over Ad-hoc Branches**: Prefer extending existing polymorphic methods (e.g. `InitialValueForSlot` on `Definition`) rather than adding bespoke `if/else` checks across compiler passes.
3. **No Fluff in Comments**: Comments must be concise, accurate, and explain non-obvious hardware or GC invariants. Obvious or conversational comments are rejected.
4. **Cross-Platform Portability**: Zero warnings across X64, ARM, ARM64, RISC-V (32/64-bit), SIMARM/SIMARM64, and sanitizers (ASAN, MSAN, TSAN, UBSAN).
