# Proposal for adding `Finalizer` and `WeakReference` to core libraries.

Author: vegorov@google.com<br>Version: 0.1

## Background

This proposal includes two interconnected pieces of functionality:

- an ability to create _weak references_ (`WeakReference`);
- an ability to associate finalization actions with objects (`Finalizer`).

### Weak references

_Weak reference_ is an object which holds to its _target_ in a manner that does
not prevent runtime system from reclaiming the referenced object when no future
expression evaluation not relying on weak references or `Expando` instances
can end up evaluating to that object.

When an object referenced by a weak reference is reclaimed, the reference itself
is cleared.

### Finalization

_Finalization_ is a process of invoking a piece of user-defined code at some
point after a certain object has been deemed _unreachable_ by a garbage
collector. _Unreachable_ objects can no longer be interacted with by any
programmatic means and thus can be safely reclaimed in a manner otherwise
unobservable to the rest of the program.

Finalization provides a way to observe and react to the event of runtime system
discovering that some object is unreachable and can be used in a variety of
ways. Most commonly, finalization is used to cleanup some sort of manually
managed (native) resources associated with GC managed objects.

For example, finalization can be used to automatically free native resources
accessed through `dart:ffi` when Dart objects owning these resources become
unreachable.

```dart
/// Wrapper for a native resource (accessed through the [_resource] pointer).
class Wrapper {
  final ffi.Pointer<ffi.Void> _resource;

  /// Without automatic finalization [Wrapper] user needs to explicitly
  /// call [destroy] to free the underlying [_resource], when it is no
  /// longer needed, leading to a possibility of leaks.
  void destroy() {
    malloc.free(_resource);
  }
}
```

### Prior art

#### ECMA-262 `WeakRef` and `FinalizationRegistry`

APIs proposed below are directly modeled after [`WeakRef`][MDN WeakRef] and
[`FinalizationRegistry`][MDN FinalizationRegistry] described in ECMA-262
standard. The choice is made to allow minimum effort efficient implementation
of these APIs in those implementations of the Dart programming language that
target JavaScript.

Original TC-39 proposal resides [here][proposal-weakrefs], while normative
specification for processing model of weak references can be found
[here][weakref-processing-model].

### Dart VM Embedding API

Dart VM exposes an [Embedding API][dart_api.h] which allows the native
programmer to associate finalization action(s) with a particular object by
creating a [_weak persistent handle_][weak handle] or
a [_finalizable handle_][finalizable handle]: after the garbage collector has
reclaimed the object referenced by such a handle it would clear the handle and
invoke a callback function associated with it.

There are two important things to highlight about this API:

- callback is invoked post reclamation and can't resurrect the referenced
object;
- callback is not allowed to invoke any Dart code or call most of Embedding API
methods;

### Design constraints and requirements

Here are the main requirements to finalization API:

- must be efficiently implementable in JavaScript;
- finalization callbacks should not interrupt sequential execution of the
program in a way which is observable by pure Dart code;
- there should be a way to minimize the possibility of _premature finalization_,
that is finalization of the wrapper object while native resources owned by it
are still in use;
- when using finalization API for managing native resources, the developer
should be able to:
  - rely on them when writing fully synchronous code,
  - rely on finalization callbacks to be invoked even if an isolate group is
  shutting down.

The second requirement is self-explanatory: we would like to avoid departures
from run-to-completion model that Dart currently provides for synchronous code.

To explain the problem of premature finalization, consider the following code:

```dart
/// [Wrapper] holds native [resource] and registers its instances for
/// finalization. When finalization callback is called it will destroy
/// corresponding [resource].
class Wrapper {
  /// Native resource held by the wrapper. Lifetime
  /// of the wrapper determines lifetime of the [_resource].
  final ffi.Pointer<ffi.Void> _resource;

  void method() {
    // [this] might be reclaimed by GC after evaluating
    // [this.resource] because it is never used again
    // and not kept live
    useResource(this._resource, causeGC());
  }
}
```

In this example if finalization can occur synchronously when `causeGC` is called
`this._resource` might be released prematurely because no references to `this`
remain alive after `this._resource` is evaluated and saved as an outgoing
argument. This means that a garbage collection which occurs within `causeGC()`
might reclaim a `Wrapper` object which was previous pointed by `this` and
consequently invoke finalization callback associated with it, which in turn
would release the native resource.

## Proposal

We split the proposed finalization API into two parts:

- `Finalizer` which:
  - does not provide strong guarantees around promptness of finalization,
  - does not impose any restrictions on objects you could associate a
    finalization action with,
  - does not impose any restrictions on finalization actions and invokes them
    asynchronously,
  - is implementable on the Web.
- `NativeFinalizer` which:
  - provides stronger guarantees around promptness of finalization,
  - guarantees that finalization actions are invoked when isolate is shutting
  down,
  - is limited to objects which implement `Finalizable` interface,
  - imposes restrictions on finalization actions to allow calling these actions
  outside of Dart universe.

The following classes are added to the `dart:core` library

```dart
/// A finalizer which can be attached to Dart objects.
///
/// When [attach]ed to a Dart object, this finalizer's callback *may* be called
/// some time after the Dart object becomes inaccessible to the program.
///
/// No promises are made that the callback will ever be called,
/// only that *if* it is called with a finalization token as argument,
/// at least one object registered in the registry with that finalization token
/// is no longer accessible to the program.
///
/// If multiple finalizers has been attached to an object or the same
/// finalizer has been attached multiple times to an object and that object
/// becomes inaccessible to the program, then any number of those finalizers
/// may trigger their associated callback. It will not necessarily be all or
/// none of them.
///
/// Finalization callbacks will happen as *events*, not during execution of
/// other code and not as a microtask, but as high-level events similar to
/// timer events.
///
/// Finalization callbacks should not throw.
abstract class Finalizer<T> {
  /// Creates a finalizer with the given finalization callback.
  ///
  /// The [callback] is bound to the [current zone](Zone.current)
  /// when the [Finalizer] is created, and will run in that zone when called.
  external factory Finalizer(void Function(T) callback);

  /// Attaches this finalizer to the given [value].
  ///
  /// When [value] is no longer accessible to the program,
  /// the registry *may* call its callback function with [token]
  /// as argument.
  ///
  /// The [value] and [detachKey] arguments do not count towards those
  /// objects being accessible to the program. Both must be objects supported
  /// as an [Expando] key.
  ///
  /// Multiple objects may be using the same finalization token,
  /// and the finalizer can be attached multiple times to the same object
  /// with different, or the same, finalization token.
  ///
  /// The callback may be called at most once per attachment, and not
  /// for registrations which have been detached since they were attached.
  void attach(Object value, T token, {Object? detachKey});

  /// Detaches this finalizer from any objects that used the identical [detachKey] when
  /// attaching this finalizer to them.
  ///
  /// If this finalizer was attached multiple times to the same object with different
  /// detachment keys, only those attachments which used [detachKey] are
  /// removed.
  ///
  /// After detaching, an attachment won't cause any callbacks to happen if the
  /// object become inaccessible.
  void detach(Object detachKey);
}

/// A weak reference to another object.
///
/// A _weak_ reference to the [target] object which may be cleared
/// (set to reference `null` instead) at any time
/// when there is no other ways for the program to access the target object.
///
/// _The referenced object may be garbage collected when the only reachable
/// references to it are weak._
///
/// Not all objects are supported as targets for weak references.
/// The [WeakReference] constructor will reject any object that is not
/// supported as an [Expando] key.
abstract class WeakReference<T extends Object> {
  /// Create a [WeakReference] pointing to the given [target].
  ///
  /// The [target] must be an object supported as an [Expando] key.
  /// Which means [target] can not be a number, a string, a boolean, or
  /// the `null` value.
  external factory WeakReference(T target);

  /// The current object weakly referenced by [this], if any.
  ///
  /// The value os either the object supplied in the constructor,
  /// or `null` if the weak reference has been cleared.
  T? get target;
}
```

The following classes are added to `dart:ffi` library:

```dart
/// Any variable which has a static type that is a subtype of a [Finalizable]
/// is guaranteed to be alive until execution exits the code block where
/// the variable would be in scope.
///
/// In other words if an object is referenced by such a variable it is
/// guaranteed to *not* be considered unreachable for the duration of the scope.
abstract class Finalizable {
  factory Finalizable._() => throw UnsupportedError("");
}

typedef NativeFinalizerSignature = Void Function(Pointer<Void> token);
typedef NativeFinalizerPtr = Pointer<NativeFunction<NativeFinalizerSignature>>;

/// A native finalizer which can be attached to Dart objects.
///
/// When [attach]ed to a Dart object, this finalizer's native callback is called
/// after the Dart object is garbage collected or becomes inaccessible for other
/// reasons.
///
/// Callbacks will happen as early as possible, when the object becomes
/// inaccessible to the program, and may happen at any moment during execution
/// of the program. At the latest, when an isolate group shuts down,
/// this callback is guaranteed to be called for each object in that isolate
/// group that the finalizer is still attached to.
///
/// Compared to the [Finalizer] from `dart:core`, which makes no promises to
/// ever call an attached callback, this native finalizer promises that all
/// attached finalizers are definitely called at least once before the program
/// ends, and the callbacks are called as soon as possible after an object
/// is recognized as inaccessible.
abstract class NativeFinalizer {
  /// Creates a finalizer with the given finalization callback.
  ///
  /// Note: the [callback] is expected to be a native function which can be
  /// executed outside of a Dart isolate. This means that passing an FFI
  /// trampoline (a function pointer obtained via [Pointer.fromFunction]) is
  /// not supported for arbitrary Dart functions. This constructor will throw
  /// if an unsupported [callback] is passed to it.
  ///
  /// The [callback] might be invoked on an arbitrary thread and not necessary
  /// on the same thread that created [FinalizationRegistry].
  external factory NativeFinalizer(NativeFinalizerPtr callback);

  /// Attaches this finalizer to the given [value].
  ///
  /// When [value] is no longer accessible to the program,
  /// the registry will call its callback function with [token]
  /// as argument.
  ///
  /// The [value] and [detachKey] arguments do not count towards those
  /// objects being accessible to the program. Both must be objects supported
  /// as an [Expando] key.
  ///
  /// Multiple objects may be using the same finalization token,
  /// and the finalizer can be attached multiple times to the same object
  /// with different, or the same, finalization token.
  ///
  /// The callback may be called at most once per attachment, and not
  /// for registrations which have been detached since they were attached.
  ///
  /// [externalSize] is an amount of native (non-Dart) memory owned by the
  /// given [value]. This information is used to drive garbage collection
  /// scheduling heuristics.
  void attach(Finalizable value, Pointer<Void> token,
      {Object? detachKey, int externalSize});

  /// Detaches this finalizer from any objects that used the identical
  /// [detachKey] when attaching this finalizer to them.
  ///
  /// If this finalizer was attached multiple times to the same object with
  /// different detachment keys, only those attachments which used [detachKey]
  /// are removed.
  ///
  /// After detaching, an attachment won't cause any callbacks to happen if the
  /// object become inaccessible.
  void detach(Object detachKey);
}
```

`Finalizer` was directly modeled after its
[JavaScript counterpart][MDN FinalizationRegistry] and only supports
asynchronous finalization, while `NativeFinalizer` is added to
`dart:ffi` to allow eager synchronous finalization.

Note differences in API between `Finalizer` and
`NativeFinalizer`:

- `NativeFinalizer` requires objects which are registered with it
to implement `Finalizable` interface, which serves as a marker instructing
optimizing compiler to provide stronger liveness guarantees for an object. This
interface is our solution to the problem of _premature finalization_.
- `NativeFinalizer` is constructed with a _native function_ as a
callback rather than a Dart function. This is done to guarantee that eager
synchronous execution of a finalization callback is not going to produce any
side-effects observable from the pure Dart code.

Unfortunately the second restriction has far reaching implications: in many
commonly used native APIs destruction method does not adhere to a single
argument signature that we expect from a finalization callback. This makes
`NativeFinalizer` API unusable without writing additional trampoline
code in native programming language (e.g. C), which we consider highly
undesirable: as we want `dart:ffi` to be expressive enough to enable developers
to create bindings in pure Dart, without requiring them to write and compile
any additional native glue code. We will discuss this limitation more in the
next section.

### Isolate-independent native functions

`dart:ffi` allows developers to construct a native function pointer to a
static function defined in Dart via [`Pointer.fromFunction`] constructor.
This constructor essentially returns a pointer to a _trampoline_ which
follows C ABI and can be called from native code. When invoked such trampoline
will perform transition from native code into Dart code, marshal arguments,
call the target static function, marshaling the result it returns and then
transition back into native code. There is an implicit expectation baked into
this process: calling a native-to-Dart trampoline will only succeed if
there is a Dart isolate associated with the thread on which we perform the call
and this isolate is in the state which allows reentrancy.

Such isolate-dependent function can't be used as a finalization callback because
finalization callbacks should be callable in contexts when there is no current
isolate at all or isolates are not allowing entering into Dart code.

It seems thus reasonable to restrict `NativeFinalizer` constructor
in a way that would reject function pointers which are pointing to
native-to-Dart FFI trampolines.

This restriction however means that users might be required to write native code
to implement their finalizers, which we consider undesirable.

Consider for example [`mmap`/`munmap`][mmap API] POSIX APIs.

```cpp
// mmap -- allocate memory, or map files or devices into memory
void *mmap(void *addr, size_t len, int prot, int flags, int fd, off_t offset);

// munmap -- remove a mapping
int munmap(void *addr, size_t len);
```

A Dart developer will be able to call `mmap` and `munmap` through FFI, but they
will not be able to use `munmap` directly as a finalizer, because it does not
conform to `Void Function(Pointer<Void>)` signature and expects the size of
the mapping as well. A developer would have to implement a helper in a native
language instead:

```cpp
// C++
struct Mapping {
  void* addr;
  uint64_t size;
};

void FinalizeMapping(Mapping* mapping) {
  munmap(mapping->addr, mapping->size);
  free(mapping);
}
```

```dart
// Dart

final NativeFinalizerPtr finalizeMapping = lib.lookup('FinalizeMapping');
final mmap = DynamicLibrary.process().lookupFunction<...>('mmap');

class _Mapping extends Struct {
  external Pointer<Void> addr;

  @Uint64()
  external int size;
}

class Mapping extends Finalizable {
  final Pointer<_Mapping> mapping;

  factory Mapping(int len) {
    final mapping = malloc.alloc<_Mapping>(sizeOf<_Mapping>());
    mapping.addr = mmap(len, ...);
    mapping.len = len;
    final wrapper = Mapping._(mapping);
    _registry.register(wrapper, mapping, externalSize: len);
    return wrapper;
  }

  static final _registry = NativeFinalizer(finalizeMapping);
}
```

It would be more convenient if Dart developer did not need to write any native
code to implement this.

It is fathomable that in future releases `dart:ffi` library could allow
some Dart functions to be invoked outside of Dart isolates, as long as this
functions can be compiled into an isolate-independent native code.

Consider for example the following piece of Dart code:

```dart
void FinalizeMapping(Pointer<Mapping> mapping) {
  munmap(mapping.ref.addr, mapping.ref.size);
  malloc.free(mapping);
}
```

This function does not really need Dart isolate for execution because it only
interacts with native world through FFI. Thus it could be compiled in a special
way and consequently used as a finalization callback.

This functionality though is outside of scope for this proposal and can be
implemented independently at a later date.

[dart_api.h]: https://github.com/dart-lang/sdk/blob/master/runtime/include/dart_api.h
[weak handle]: https://github.com/dart-lang/sdk/blob/39a165647a7f2cf1ca8e81e696c552d25365c0c5/runtime/include/dart_api.h#L460-L494
[finalizable handle]: https://github.com/dart-lang/sdk/blob/39a165647a7f2cf1ca8e81e696c552d25365c0c5/runtime/include/dart_api.h#L512-L550
[MDN FinalizationRegistry]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/FinalizationRegistry
[MDN WeakRef]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakRef
[`Pointer.fromFunction`]: https://api.dart.dev/dev/2.15.0-95.0.dev/dart-ffi/Pointer/fromFunction.html
[mmap API]: https://man7.org/linux/man-pages/man2/mmap.2.html
[proposal-weakrefs]: https://github.com/tc39/proposal-weakrefs
[weakref-processing-model]: https://tc39.es/ecma262/#sec-weakref-processing-model
