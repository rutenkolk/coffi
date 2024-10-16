# coffi
[![Clojars Project](https://img.shields.io/clojars/v/org.suskalo/coffi.svg)](https://clojars.org/org.suskalo/coffi)

Coffi is a foreign function interface library for Clojure, using the [Foreign
Function & Memory API](https://openjdk.org/jeps/454) in JDK 22 and later. This
allows calling native code directly from Clojure without the need for either
Java or native code specific to the library, as e.g. the JNI does. Coffi focuses
on ease of use, including functions and macros for creating wrappers to allow
the resulting native functions to act just like Clojure ones, however this
doesn't remove the ability to write systems which minimize the cost of
marshaling data and optimize for performance, to make use of the low-level
access Panama gives us.

## Installation
This library is available on Clojars. Add one of the following entries to the
`:deps` key of your `deps.edn`:

```clojure
org.suskalo/coffi {:mvn/version "1.0.471"}
io.github.IGJoshua/coffi {:git/tag "v1.0.471" :git/sha "d3d2d25"}
```

If you use this library as a git dependency, you will need to prepare the
library.

```sh
$ clj -X:deps prep
```

Coffi requires usage of the package `java.lang.foreign`, and most of the
operations are considered unsafe by the JDK, and are therefore unavailable to
your code without passing some command line flags. In order to use coffi, add
the following JVM arguments to your application.

```sh
--enable-native-access=ALL-UNNAMED
```

You can specify JVM arguments in a particular invocation of the Clojure CLI with
the -J flag like so:

``` sh
clj -J--enable-native-access=ALL-UNNAMED
```

You can also specify them in an alias in your `deps.edn` file under the
`:jvm-opts` key (see the next example) and then invoking the CLI with that alias
using `-M`, `-A`, or `-X`.

``` clojure
{:aliases {:dev {:jvm-opts ["--enable-native-access=ALL-UNNAMED"]}}}
```

Other build tools should provide similar functionality if you check their
documentation.

When creating an executable jar file, you can avoid the need to pass this
argument by adding the manifest attribute `Enable-Native-Access: ALL-UNNAMED` to
your jar.

Coffi also includes support for the linter clj-kondo. If you use clj-kondo and
this library's macros are not linting correctly, you may need to install the
config bundled with the library. You can do so with the following shell command,
run from your project directory:

```sh
$ clj-kondo --copy-configs --dependencies --lint "$(clojure -Spath)"
```

## Usage
There are two major components to coffi and interacting with native code:
manipulating off-heap memory, and loading native code for use with Clojure.

In the simplest cases, the native functions you call will work exclusively with
built-in types, for example the function `strlen` from libc.

```clojure
(require '[coffi.mem :as mem :refer [defalias]])
(require '[coffi.ffi :as ffi :refer [defcfn]])

(defcfn strlen
  "Given a string, measures its length in bytes."
  strlen [::mem/c-string] ::mem/long)

(strlen "hello")
;; => 5
```

The first argument to `defcfn` is the name of the Clojure var that will hold the
native function reference, followed by an optional docstring and attribute map,
then the C function identifier, including the name of the native symbol, a
vector of argument types, and the return type.

If you wish to use a native function as an anonymous function, it can be done
with the `cfn` function.

```clojure
((ffi/cfn "strlen" [::mem/c-string] ::mem/long) "hello")
;; => 5
```

If you want to use functions from libraries other than libc, then you'll need to
load them. Two functions are provided for this, `load-system-library`, and
`load-library`. `load-system-library` takes a string which represents the name
of a library that should be loaded via system lookup.

```clojure
(ffi/load-system-library "z")
```

This will load libz from the appropriate place on the user's load path.

Alternatively, `load-library` takes a file path to a dynamically loaded library.

```clojure
(ffi/load-library "lib/libz.so")
```

This will load libz from the lib subdirectory of the current working directory.
As you can see this requires the entire filename, including platform-specific
file extensions.

If a library is attempted to be loaded but doesn't exist or otherwise can't be
loaded, an exception is thrown. This can be convenient as any namespace with a
`load-library` call at the top level cannot be required without the library
being able to be loaded.

### Primitive Types
Coffi defines a basic set of primitive types:
- byte
- short
- int
- long
- char
- float
- double
- pointer

Each of these types maps to their C counterpart. Values of any of these
primitive types except for `pointer` will be cast with their corresponding
Clojure function when they are passed as arguments to native functions.
Additionally, the `c-string` type is defined, although it is not primitive.

### Composite Types
In addition, some composite types are also defined in coffi, including struct
and union types (unions will be discussed with serialization and
deserialization). For an example C struct and function:

```c
typedef struct point {
    float x;
    float y;
} Point;

Point zero(void) {
    Point res = {};

    res.x = 0.0;
    res.y = 0.0;

    return res;
}
```

The corresponding coffi definition is like so:

```clojure
(defcfn zero-point
  "zero" [] [::mem/struct [[:x ::mem/float] [:y ::mem/float]]])

(zero-point)
;; => {:x 0.0,
;;     :y 0.0}
```

Writing out struct definitions like this every time would get tedious, so the
macro `defalias` is used to define a struct alias.

```clojure
(defalias ::point
  [::mem/struct
   [[:x ::mem/float]
    [:y ::mem/float]]])

(defcfn zero-point
  "zero" [] ::point)
```

Struct definitions do not include any padding by default. Functions for
transforming struct types to include padding conforming to various standards can
be found in `coffi.layout`.

``` clojure
(require '[coffi.layout :as layout])

(defalias ::needs-padding
  (layout/with-c-layout
   [::mem/struct
    [[:a ::mem/char]
     [:x ::mem/float]]]))

(mem/size-of ::needs-padding)
;; => 8

(mem/align-of ::needs-padding)
;; => 4
```

Values deserialized with types produced from layout functions may include an
extra `:coffi.layout/padding` key with a nil value.

A limitation of the `defcfn` macro in its current form is that types provided to
it must be provided in a literal form, not as an expression that evaluates to a
type. This means that if you wish to use a layout function on a struct you must
define an alias for it before the type can be used as a type in `defcfn`.

In cases where a pointer to some data is required to pass as an argument to a
native function, but doesn't need to be read back in, the `pointer` primitive
type can take a type argument.

```clojure
[::mem/pointer ::mem/int]
```

Arrays are also supported via a type argument. Keep in mind that they are the
array itself, and not a pointer to the array like you might see in certain cases
in C.

```clojure
[::mem/array ::mem/int 3]
```

### Callbacks
In addition to these composite types, there is also support for Clojure
functions.

```clojure
[::ffi/fn [::mem/c-string] ::mem/int]
```

Be aware though that if an exception is thrown out of a callback that is called
from C, the JVM will crash. The resulting crash log should include the exception
type and message in the registers section, but it's important to be aware of all
the same. Ideally you should test your callbacks before actually passing them to
native code.

When writing a wrapper library for a C library, it may be a good choice to wrap
all passed Clojure functions in an additional function which catches all
throwables, potentially notifies the user in some manner (e.g. logging), and
returns a default value. This is on the wrapper library's developer to decide
when and where this is appropriate, as in some cases no reasonable default
return value can be determined and it is most sensible to simply crash the JVM.
This is the reason that coffi defaults to this behavior, as in the author's
opinion it is better to fail hard and fast rather than to attempt to produce a
default and cause unexpected behavior later.

Another important thing to keep in mind is the expected lifetime of the function
that you pass to native code. For example it is perfectly fine to pass an
anonymous function to a native function if the callback will never be called
again once the native function returns. If however it saves the callback for
later use the JVM may collect it prematurely, causing a crash when the callback
is later called by native code.

### Variadic Functions
Some native functions can take any number of arguments, and in these cases coffi
provides `vacfn-factory` (for "varargs C function factory").

```clojure
(def printf-factory (ffi/vacfn-factory "printf" [::mem/c-string] ::mem/int))
```

This returns a function of the types of the rest of the arguments which itself
returns a native function wrapper.

```clojure
(def print-int (printf-factory ::mem/int))

(print-int "Some integer: %d\n" 5)
;; Some integer: 5
```

At the moment there is no equivalent to `defcfn` for varargs functions.

Some native functions that are variadic use the type `va_list` to make it easier
for other languages to call them in their FFI. At the time of writing, coffi
does not support va-list, however it is a planned feature.

### Global Variables
Some libraries include global variables or constants accessible through symbols.
To start with, constant values stored in symbols can be fetched with `const`, or
the parallel macro `defconst`

```clojure
(def some-const (ffi/const "some_const" ::mem/int))
(ffi/defconst some-const "some_const" ::mem/int)
```

This value is fetched once when you call `const` and is turned into a Clojure
value. If you need to refer to a global variable, then `static-variable` (or
parallel `defvar`) can be used to create a reference to the native value.

```clojure
(def some-var (ffi/static-variable "some_var" ::mem/int))
(ffi/defvar some-var "some_var" ::mem/int)
```

This variable is an `IDeref`. Each time you dereference it, the value will be
deserialized from the native memory and returned. Additional functions are
provided for mutating the variable.

```clojure
(ffi/freset! some-var 5)
;; => 5
@some-var
;; => 5
```

Be aware however that there is no synchronization on these types. The value
being read is not read atomically, so you may see an inconsistent state if the
value is being mutated on another thread.

A parallel function `fswap!` is also provided, but it does not provide any
atomic semantics either.

The memory that backs the static variable can be fetched with the function
`static-variable-segment`, which can be used to pass a pointer to the static
variable to native functions that require it.

### Complex Wrappers
Some functions require more complex code to map nicely to a Clojure function.
The `defcfn` macro provides facilities to wrap the native function with some
Clojure code to make this easier.

```clojure
(defcfn takes-array
  "takes_array_with_count" [::mem/pointer ::mem/long] ::mem/void
  native-fn
  [ints]
  (let [arr-len (count ints)
        int-array (mem/serialize ints [::mem/array ::mem/int arr-len])]
    (native-fn int-array arr-len)))
```

The symbol `native-fn` can be any unqualified symbol, and names the native
function being wrapped. It must be called in the function body below if you want
to call the native code.

This `serialize` function has a paired `deserialize`, and allows marshaling
Clojure data back and forth to native data structures.

This can be used to implement out variables often seen in native code.

```clojure
(defcfn out-int
  "out_int" [::mem/pointer] ::mem/void
  native-fn
  [i]
  (let [int-ptr (mem/serialize i [::mem/pointer ::mem/int])]
    (native-fn int-ptr)
    (mem/deserialize int-ptr [::mem/pointer ::mem/int])))
```

### Arenas
In order to serialize any non-primitive type (such as the previous
`[::mem/pointer ::mem/int]`), off-heap memory needs to be allocated. When memory
is allocated inside the JVM, the memory is associated with an arena. Because
none was provided here, the arena is an implicit arena, and the memory will be
freed when the serialized object is garbage collected.

In many cases this is not desirable, because the memory is not freed in a
deterministic manner, causing garbage collection pauses to become longer, as
well as changing allocation performance. Instead of an implicit arena, there
are other kinds of arenas as well. A `confined-arena` is a thread-local arena.
Confined arenas are `Closeable`, which means they should usually be used in a
`with-open` form. When a `confined-arena` is closed, it immediately frees all
the memory associated with it. The previous example, `out-int`, can be
implemented with a confined arena.

```clojure
(defcfn out-int
  "out_int" [::mem/pointer] ::mem/void
  native-fn
  [i]
  (with-open [arena (mem/confined-arena)]
    (let [int-ptr (mem/serialize i [::mem/pointer ::mem/int] arena)]
      (native-fn int-ptr)
      (mem/deserialize int-ptr [::mem/pointer ::mem/int]))))
```

This will free the pointer immediately upon leaving the function.

When memory needs to be accessible from multiple threads, there's
`shared-arena`. When a `shared-arena` is `.close`d, it will release all its
associated memory immediately, and so this should only be done once all other
threads are done accessing memory associated with it.

In addition, two non-`Closeable` arenas are `global-arena`, which never frees
the resources associated with it, and `auto-arena`, which is an arena that frees
its resources once all of them are unreachable during a garbage collection
cycle, like an implicit arena, but potentially for multiple allocations rather
than just one.

### Serialization and Deserialization
Custom serializers and deserializers may also be created. This is done using two
sets of three multimethods which can be extended by the user. For any given
type, only one set need be implemented.

Two examples of custom types are given here, one is a 3d vector, and the other
an example of a tagged union.

#### Vector3
For the vector type, it will serialize to a pointer to an array of three floats.

The multimethod `primitive-type` returns the primitive type that a given type
serializes to. For this example, it should be a pointer.

```clojure
(defmethod mem/primitive-type ::vector
  [_type]
  ::mem/pointer)
```

For any type which doesn't serialize to a primitive, it returns nil, and
therefore need not be overriden.

Next is `serialize*` and `deserialize*`, multimethods that work with types that
serialize to primitives.

```clojure
(defmethod mem/serialize* ::vector
  [obj _type arena]
  (mem/serialize obj [::mem/array ::mem/float 3] arena))

(defmethod mem/deserialize* ::vector
  [segment _type]
  (mem/deserialize (mem/reinterpret segment (mem/size-of [::mem/array ::mem/float 3]))
                   [::mem/array ::mem/float 3]))
```

The `reinterpret` function allows you to take a segment and decorate it with a
new size, and possibly associate it with an arena or add cleanup functions on
it.

In cases like this where we don't know the arena of the pointer, we could use
`reinterpret` to ensure it's freed. For example if a `free-vector!` function
that takes a pointer exists, we could use this:

```clojure
(defcfn returns-vector
  "returns_vector" [] ::mem/pointer
  native-fn
  [arena]
  (let [ret-ptr (native-fn)]
    (-> (reinterpret ret-ptr (mem/size-of ::vector) arena free-vector!)
        (deserialize ::vector))))
```

This function takes an arena and returns the deserialized vector, and it will
free the pointer when the arena closes.

#### Tagged Union
For the tagged union type, we will represent the value as a vector of a keyword
naming the tag and the value. The type itself will need to take arguments,
similar to `struct`. For example, if we were to represent a result type like in
Rust, we might have the following values:

```clojure
[:ok 5]
[:err "Invalid number format"]
```

To represent this, we can have a `tagged-union` type. For this instance of the
result type, it may look like this:

```clojure
[::tagged-union [:ok :err] {:ok ::mem/int :err ::mem/c-string}]
```

The native representation of these objects is a struct of the tag and a union of
the value. In order to correctly serialize the data and pass it to native code,
we need a representation of the native layout of the data. The `c-layout`
multimethod provides that.

```clojure
(defmethod mem/c-layout ::tagged-union
  [[_tagged-union tags type-map]]
  (mem/c-layout [::mem/struct
                 [[:tag ::mem/long]
                  [:value [::mem/union (vals type-map)]]]]))
```

Types with type arguments are represented as vectors of the type name and any
additional arguments. The type name is what is dispatched on for the
multimethods.

Now that we have a native layout, we need to be able to serialize and
deserialize the value into and out of memory segments. This is accomplished with
`serialize-into` and `deserialize-from`.

```clojure
(defn item-index
  "Gets the index of the first occurance of `item` in `coll`."
  [coll item]
  (first
   (->> coll
        (map-indexed vector)
        (filter (comp #{item} second))
        (map first))))

(defmethod mem/serialize-into ::tagged-union
  [obj [_tagged-union tags type-map] segment arena]
  (mem/serialize-into
   {:tag (item-index tags (first obj))
    :value (second obj)}
   [::mem/struct
    [[:tag ::mem/long]
     [:value (get type-map (first obj))]]]
   segment
   arena))
```

This serialization method is rather simple, it just turns the vector value into
a map, and serializes it as a struct, choosing the type of the value based on
the tag.

```clojure
(defmethod mem/deserialize-from ::tagged-union
  [segment [_tagged-union tags type-map]]
  (let [tag (mem/deserialize-from segment ::mem/long)]
    [(nth tags tag)
     (mem/deserialize-from
      (mem/slice segment (mem/size-of ::mem/long))
      (get type-map tag))]))
```

Deserialization is a little more complex. First the tag is retrieved from the
beginning of the segment, and then the type of the value is decided based on
that before it is deserialized.

### Unions
In the last section the custom serialization and deserialization of a tagged
union used a union from coffi in order to define the native layout, but not for
actual serialization or deserialization. This is intentional. A union in coffi
is rather limited. It can be serialized, but not deserialized without external
information.

```clojure
[::mem/union
 #{::mem/float ::mem/double}
 :dispatch #(cond
             (float? %) ::mem/float
             (double? %) ::mem/double)]
```

This is a minimal union in coffi. If the `:dispatch` keyword argument is not
passed, then the union cannot be serialized, as coffi would not know which type
to serialize the values as. In the example with a tagged union, a dispatch
function was not provided because the type was only used for the native layout.

In addition to a dispatch function, when serializing a union an extract function
may also be provided. In the case of the value in the tagged union from before,
it could be represented for serialization purposes like so:

```clojure
[::mem/union
 #{::mem/int ::mem/c-string}
 :dispatch #(case (first %)
              :ok ::mem/int
              :err ::mem/c-string)
 :extract second]
```

This union however would not include the tag when serialized.

If a union is deserialized, then all that coffi does is to allocate a new
segment of the appropriate size with an implicit arena so that it may later be
garbage collected, and copies the data from the source segment into it. It's up
to the user to call `deserialize-from` on that segment with the appropriate
type.

### Unwrapped Native Handles
Some native libraries work with handles to large amounts of data at once, making
it undesirable to marshal data back and forth from Clojure, both because it's
not necessary to work with the data in Clojure directly, or also because of the
high (de)serialization costs associated with marshaling. In cases like these,
unwrapped native handles are desirable.

The functions `make-downcall` and `make-varargs-factory` are also provided to
create raw function handles.

```clojure
(def raw-strlen (ffi/make-downcall "strlen" [::mem/c-string] ::mem/long))
(raw-strlen (mem/serialize "hello" ::mem/c-string))
;; => 5
```

With raw handles, the argument types are expected to exactly match the types
expected by the native function. For primitive types, those are primitives. For
pointers, that is `MemorySegment`, and for composite types like structs and
unions, that is also `MemorySegment`. `MemorySegment` comes from the
`java.lang.foreign` package.

In addition, when a raw handle returns a composite type represented with a
`MemorySegment`, it requires an additional first argument, a `SegmentAllocator`,
which can be acquired with `arena-allocator` to get one associated with a
specific arena. The returned value will live until that arena is released.

In addition, function types can be specified as being raw, in the following
manner:

```clojure
[::ffi/fn [::mem/int] ::mem/int :raw-fn? true]
```

Clojure functions serialized to this type will have their arguments and return
value exactly match the types specified and will not perform any serialization
or deserialization at their boundaries.

One important caveat to consider when writing wrappers for performance-sensitive
functions is that the convenience macro `defcfn` that coffi provides will
already perform no serialization or deserialization on primitive arguments and
return types, so for functions with only primitive argument and return types
there is no performance reason to choose unwrapped native handles over the
convenience macro.

### Manual (De)Serialization
Coffi uses multimethods to dispatch to (de)serialization functions to enable
code that's generic over the types it operates on. However, in cases where you
know the exact types that you will be (de)serializing and the multimethod
dispatch overhead is too high a cost, it may be appropriate to manually handle
(de)serializing data. This will often be done paired with [Unwrapped Native
Handles](#unwrapped-native-handles).

Convenience functions are provided to both read and write all primitive types
and addresses, including byte order.

As an example, when wrapping a function that returns an array of big-endian
floats, the following code might be used.

``` clojure
;; int returns_float_array(float **arr)
(def ^:private returns-float-array* (ffi/make-downcall "returns_float_array" [::mem/pointer] ::mem/int))
;; void releases_float_array(float *arr)
(def ^:private release-floats* (ffi/make-downcall "releases_float_array" [::mem/pointer] ::mem/void))

(defn returns-float-array
  []
  (with-open [arena (mem/confined-arena)]
    ;; float *out_floats;
    ;; int num_floats = returns_float_array(&out_floats);
    (let [out-floats (mem/alloc mem/pointer-size arena)
          num-floats (returns-float-array* out-floats)
          floats-addr (mem/read-address out-floats)
          floats-slice (mem/reinterpret floats-addr (unchecked-multiply-int mem/float-size num-floats))]
      ;; Using a try/finally to perform an operation when the stack frame exits,
      ;; but not to try to catch anything.
      (try
        (loop [floats (transient [])
               index 0]
          (if (>= index num-floats)
            (persistent! floats)
            (recur (conj! floats (mem/read-float floats-slice
                                                 (unchecked-multiply-int index mem/float-size)
                                                 mem/big-endian))
                   (unchecked-inc-int index))))
        (finally
          (release-floats* floats-addr))))))
```

The above code manually performs all memory operations rather than relying on
coffi's dispatch. This will be more performant, but because multimethod overhead
is usually relatively low, it's recommended to use the multimethod variants for
convenience in colder functions.

### Data Model
In addition to the macros and functions provided to build a Clojure API for
native libraries, facilities are provided for taking data and loading all the
symbols specified by it. This can be useful if a library provides (or an
external provider maintains) a data representation of their API, as Clojure data
to represent it may be programmatically generated from these sources.

The data to represent an API is a map with the following form:

```clojure
(def strlen-libspec
  {:strlen {:type :function
            :symbol "strlen"
            :function/args [::mem/c-string]
            :function/ret ::mem/long}})
```

Each key in this map represents a single symbol to be loaded. The value is a map
with at least the keys `:type` and `:symbol`. These are the currently recognized
types:

- function
- varargs-factory
- const
- static-var

Each one has its own set of additional keys which can be added to the map. Both
`function` and `varargs-factory` have the three keys `:function/args`,
`:function/ret`, and `:function/raw-fn?`. The `const` type has `:const/type` and
`static-var` has `:static-var/type`.

This data can be passed to the function `reify-libspec`, which will take the
data and return a map from the same keys as the input map to whatever value is
appropriate for a given symbol type (e.g. a Clojure function for `function`, a
value for `const`, etc.).

```clojure
(ffi/reify-libspec strlen-libspec)
;; => {:strlen #function[...]}
```

This functionality can be extended by specifying new types as implementations of
the multimethod `reify-symbolspec`, although it's recommended that for any
library authors who do so, namespaced keywords be used to name types.

## Alternatives
**ALTERNATIVES INFORMATION IS OUT OF DATE. THE LINKS ARE FINE, BUT DESCRIPTIONS WILL BE UPDATED AT A LATER DATE.**

This library is not the only Clojure library providing access to native code. In
addition the following libraries exist:

- [dtype-next](https://github.com/cnuernber/dtype-next)
- [tech.jna](https://github.com/techascent/tech.jna)
- [clojure-jna](https://github.com/Chouser/clojure-jna)

Dtype-next has support for Java versions 8-16 and GraalVM, but is focused
strongly on array-based programming, as well as being focused on keeping memory
in the native side rather than marshaling data to and from Clojure-native
structures. In Java 16, this uses the first iteration of Panama, while in other
Java versions it uses JNA.

Tech.jna and clojure-jna both use the JNA library in all cases, and neither
provide support for dealing with struct types or callbacks.

An additional alternative to coffi is to directly use the JNI, which is the
longest-standing method of wrapping native code in the JVM, but comes with the
downside that it requires you to write both native and Java code to use, even if
you only intend to use it from Clojure.

If your application needs to be able to run in earlier versions of the JVM than
17, or you don't want to use incubator functionality, you should consider these
other options. Dtype-next provides the most robust support for native code, but
if you are wrapping a simple library then the other libraries may be more
appealing, as they have a smaller API surface area and it's easier to wrap
functions.

### Benchmarks
**BENCHMARKS FOR COFFI AND DTYPE-NEXT ARE BASED ON AN OLD VERSION. NEW BENCHMARKS WILL BE CREATED SOON.**

An additional consideration when thinking about alternatives is the performance
of each available option. It's an established fact that JNA (used by all three
alternative libraries on JDK <16) introduces more overhead when calling native
code than JNI does.

In order to provide a benchmark to see how much of a difference the different
native interfaces make, we can use
[criterium](https://github.com/hugoduncan/criterium) to benchmark each.
[GLFW](https://www.glfw.org)'s
[`glfwGetTime`](https://www.glfw.org/docs/latest/group__input.html#gaa6cf4e7a77158a3b8fd00328b1720a4a)
function will be used for the test as it performs a simple operation, and is
conveniently already wrapped in JNI by the excellent
[LWJGL](https://www.lwjgl.org/) library.

The following benchmarks were run on a Lenovo Thinkpad with an Intel i7-10610U
running Manjaro Linux, using Clojure 1.10.3 on Java 17.

#### JNI
The baseline for performance is the JNI. Using LWJGL it's relatively simple to
benchmark. The following Clojure CLI command will start a repl with LWJGL and
criterium loaded.

```sh
$ clj -Sdeps '{:deps {org.lwjgl/lwjgl {:mvn/version "3.2.3"}
                      org.lwjgl/lwjgl-glfw {:mvn/version "3.2.3"}
                      org.lwjgl/lwjgl$natives-linux {:mvn/version "3.2.3"}
                      org.lwjgl/lwjgl-glfw$natives-linux {:mvn/version "3.2.3"}
                      criterium/criterium {:mvn/version "0.4.6"}}}'
```

Then from the repl

```clojure
user=> (import 'org.lwjgl.glfw.GLFW)
org.lwjgl.glfw.GLFW
user=> (require '[criterium.core :as bench])
nil
user=> (GLFW/glfwInit)
true
user=> (bench/bench (GLFW/glfwGetTime) :verbose)
amd64 Linux 5.10.68-1-MANJARO 8 cpu(s)
OpenJDK 64-Bit Server VM 17+35-2724
Runtime arguments: -Dclojure.basis=/home/jsusk/.clojure/.cpcache/2667074721.basis
Evaluation count : 1613349900 in 60 samples of 26889165 calls.
      Execution time sample mean : 32.698446 ns
             Execution time mean : 32.697811 ns
Execution time sample std-deviation : 1.274600 ns
    Execution time std-deviation : 1.276437 ns
   Execution time lower quantile : 30.750813 ns ( 2.5%)
   Execution time upper quantile : 33.757662 ns (97.5%)
                   Overhead used : 6.400704 ns
nil
```

GLFW requires that we initialize it before calling the `glfwGetTime` function.
Besides that this is a simple interop call which directly maps to the native
function.

This gives us a basis of 32.7 ns +/-1.3 ns. All other libraries will be
evaluated relative to this result.

To ensure fairness, we'll also get that overhead value to be used in further
tests.

```clojure
user=> bench/estimated-overhead-cache
6.400703613065185E-9
```

#### Coffi
The dependencies when using coffi are simpler, but it also requires some JVM
options to support the foreign access api.

```sh
$ clj -Sdeps '{:deps {org.suskalo/coffi {:mvn/version "0.1.205"}
                      criterium/criterium {:mvn/version "0.4.6"}}}' \
      -J--add-modules=jdk.incubator.foreign \
      -J--enable-native-access=ALL-UNNAMED
```

In order to ensure fair comparisons, we're going to use the same overhead value
on each run, so before we do the benchmark we'll set it to the observed value
from last time.

```clojure
user=> (require '[criterium.core :as bench])
nil
user=> (alter-var-root #'bench/estimated-overhead-cache (constantly 6.400703613065185E-9))
6.400703613065185E-9
user=> (require '[coffi.ffi :as ffi])
nil
user=> (require '[coffi.mem :as mem])
nil
user=> (ffi/load-system-library "glfw")
nil
user=> ((ffi/cfn "glfwInit" [] ::mem/int))
1
user=> (let [f (ffi/cfn "glfwGetTime" [] ::mem/double)]
         (bench/bench (f) :verbose))
amd64 Linux 5.10.68-1-MANJARO 8 cpu(s)
OpenJDK 64-Bit Server VM 17+35-2724
Runtime arguments: --add-modules=jdk.incubator.foreign --enable-native-access=ALL-UNNAMED -Dclojure.basis=/home/jsusk/.clojure/.cpcache/72793624.basis
Evaluation count : 1657995600 in 60 samples of 27633260 calls.
      Execution time sample mean : 31.382665 ns
             Execution time mean : 31.386493 ns
Execution time sample std-deviation : 1.598571 ns
    Execution time std-deviation : 1.608818 ns
   Execution time lower quantile : 29.761194 ns ( 2.5%)
   Execution time upper quantile : 33.228276 ns (97.5%)
                   Overhead used : 6.400704 ns
nil
```

This result is about 1.3 ns faster, and while that is less than the standard
deviation of 1.6, it's quite close to it.

#### Clojure-JNA
Clojure-JNA uses the JNA library, which was designed to provide Java with an
easy way to access native libraries, but which is known for not having the
greatest performance. Since this is an older project, I'm also including the
clojure dependency to ensure the correct version is used.

```sh
$ clj -Sdeps '{:deps {org.clojure/clojure {:mvn/version "1.10.3"}
                      net.n01se/clojure-jna {:mvn/version "1.0.0"}
                      criterium/criterium {:mvn/version "0.4.6"}}}'
```

The naive way to call the function using Clojure-JNA is to use `jna/invoke`.

```clojure
user=> (require '[criterium.core :as bench])
nil
user=> (alter-var-root #'bench/estimated-overhead-cache (constantly 6.400703613065185E-9))
6.400703613065185E-9
user=> (require '[net.n01se.clojure-jna :as jna])
nil
user=> (jna/invoke Integer glfw/glfwInit)
1
user=> (bench/bench (jna/invoke Double glfw/glfwGetTime) :verbose)
amd64 Linux 5.10.68-1-MANJARO 8 cpu(s)
OpenJDK 64-Bit Server VM 17+35-2724
Runtime arguments: -Dclojure.basis=/home/jsusk/.clojure/.cpcache/3229486237.basis
Evaluation count : 195948720 in 60 samples of 3265812 calls.
      Execution time sample mean : 350.335614 ns
             Execution time mean : 350.373520 ns
Execution time sample std-deviation : 24.833070 ns
    Execution time std-deviation : 24.755929 ns
   Execution time lower quantile : 300.000019 ns ( 2.5%)
   Execution time upper quantile : 365.759273 ns (97.5%)
                   Overhead used : 6.400704 ns

Found 13 outliers in 60 samples (21.6667 %)
	low-severe	 12 (20.0000 %)
	low-mild	 1 (1.6667 %)
 Variance from outliers : 53.4220 % Variance is severely inflated by outliers
nil
```

As you can see, this method of calling functions is very bad for performance,
with call overhead dominating function runtime by an order of magnitude. That
said, this isn't a completely fair comparison, nor the most realistic, because
this way of calling functions looks the function up on each invocation.

To adjust for this, we'll use the `jna/to-fn` function to give a persistent
handle to the function that we can call.

```clojure
user=> (let [f (jna/to-fn Double glfw/glfwGetTime)]
         (bench/bench (f) :verbose))
amd64 Linux 5.10.68-1-MANJARO 8 cpu(s)
OpenJDK 64-Bit Server VM 17+35-2724
Runtime arguments: -Dclojure.basis=/home/jsusk/.clojure/.cpcache/3229486237.basis
Evaluation count : 611095020 in 60 samples of 10184917 calls.
      Execution time sample mean : 104.623634 ns
             Execution time mean : 104.638406 ns
Execution time sample std-deviation : 7.649296 ns
    Execution time std-deviation : 7.638963 ns
   Execution time lower quantile : 92.446016 ns ( 2.5%)
   Execution time upper quantile : 110.258832 ns (97.5%)
                   Overhead used : 6.400704 ns
nil
```

This is much better, but is still about 3x slower than JNI, meaning the overhead
from using JNA is still bigger than the function runtime.

This performance penalty is still small in the scope of longer-running
functions, and so may not be a concern for your application, but it is something
to be aware of.

#### tech.jna
The tech.jna library is similar in scope to Clojure-JNA, however was written to
fit into an ecosystem of libraries meant for array-based programming for machine
learning and data science.

```sh
$ clj -Sdeps '{:deps {techascent/tech.jna {:mvn/version "4.05"}
                      criterium/criterium {:mvn/version "0.4.6"}}}'
```

This library is also quite simple to use, the only slightly odd thing I'm doing
here is to dereference the var outside the benchmark in order to ensure it's an
apples-to-apples comparison. We don't want var dereference time mucking up our
benchmark.

```clojure
user=> (require '[criterium.core :as bench])
nil
user=> (alter-var-root #'bench/estimated-overhead-cache (constantly 6.400703613065185E-9))
6.400703613065185E-9
user=> (require '[tech.v3.jna :as jna])
nil
user=> (jna/def-jna-fn "glfw" glfwInit "initialize glfw" Integer)
#'user/glfwInit
user=> (glfwInit)
Oct 09, 2021 10:30:50 AM clojure.tools.logging$eval1122$fn__1125 invoke
INFO: Library glfw found at [:system "glfw"]
1
user=> (jna/def-jna-fn "glfw" glfwGetTime "gets the time as a double since init" Double)
#'user/glfwGetTime
user=> (let [f @#'glfwGetTime]
         (bench/bench (f) :verbose))
amd64 Linux 5.10.68-1-MANJARO 8 cpu(s)
OpenJDK 64-Bit Server VM 17+35-2724
Runtime arguments: -Dclojure.basis=/home/jsusk/.clojure/.cpcache/2910209237.basis
Evaluation count : 323281680 in 60 samples of 5388028 calls.
      Execution time sample mean : 203.976803 ns
             Execution time mean : 203.818712 ns
Execution time sample std-deviation : 14.557312 ns
    Execution time std-deviation : 14.614080 ns
   Execution time lower quantile : 179.732593 ns ( 2.5%)
   Execution time upper quantile : 213.929374 ns (97.5%)
                   Overhead used : 6.400704 ns
nil
```

This version is even slower than Clojure-JNA. I'm unsure where this overhead is
coming from, but I'll admit that I haven't looked at their implementations very
closely.

#### dtype-next
The library dtype-next replaced tech.jna in the toolkit of the group working on
machine learning and array-based programming, and it includes support for
composite data types including structs, as well as primitive functions and
callbacks.

In addition, dtype-next has two different ffi backends. First is JNA, which is
usable on any JDK version, and is what we'll use for the first benchmark. Second
is the Java 16 version of Project Panama, which will be shown next.

In order to use the dtype-next ffi with the JNA backend, the JNA library has to
be included in the dependencies.

```sh
$ clj -Sdeps '{:deps {cnuernber/dtype-next {:mvn/version "8.032"}
                      net.java.dev.jna/jna {:mvn/version "5.8.0"}
                      criterium/criterium {:mvn/version "0.4.6"}}}'
```

The dtype-next library also requires some more ceremony around declaring native
functions. One advantage this has is that multiple symbols with the same name
can be loaded from different shared libraries, but it also does increase
friction when defining native wrappers.

Some easier ways to define native wrappers are provided than what is seen here,
but they share some disadvantages in documentation over the core methods
provided in coffi, although they are comparable to the data model provided in
coffi.

```clojure
user=> (require '[criterium.core :as bench])
nil
user=> (alter-var-root #'bench/estimated-overhead-cache (constantly 6.400703613065185E-9))
6.400703613065185E-9
user=> (require '[tech.v3.datatype.ffi :as dt-ffi])
nil
user=> (def fn-defs {:glfwInit {:rettype :int32} :glfwGetTime {:rettype :float64}})
#'user/fn-defs
user=> (def library-def (dt-ffi/define-library fn-defs))
#'user/library-def
user=> (def library-instance (dt-ffi/instantiate-library library-def "/usr/lib/libglfw.so"))
#'user/library-instance
user=> (def init (:glfwInit @library-instance))
#'user/init
user=> (init)
1
user=> (let [f (:glfwGetTime @library-instance)]
         (bench/bench (f) :verbose))
amd64 Linux 5.10.68-1-MANJARO 8 cpu(s)
OpenJDK 64-Bit Server VM 17+35-2724
Runtime arguments: -Dclojure.basis=/home/jsusk/.clojure/.cpcache/643862289.basis
Evaluation count : 710822100 in 60 samples of 11847035 calls.
      Execution time sample mean : 90.900112 ns
             Execution time mean : 90.919917 ns
Execution time sample std-deviation : 6.463312 ns
    Execution time std-deviation : 6.470108 ns
   Execution time lower quantile : 79.817126 ns ( 2.5%)
   Execution time upper quantile : 95.454652 ns (97.5%)
                   Overhead used : 6.400704 ns
nil
```

This version of JNA usage is significantly faster than either of the other JNA
libraries, but is still substantially slower than using JNI or coffi.

In addition to the JNA backend, dtype-next has a Java 16-specific backend that
uses an older version of Panama. This version requires similar setup to coffi in
order to run.

```sh
$ clj -Sdeps '{:deps {cnuernber/dtype-next {:mvn/version "8.032"}
                      criterium/criterium {:mvn/version "0.4.6"}}}' \
      -J--add-modules=jdk.incubator.foreign \
      -J-Dforeign.restricted=permit \
      -J--add-opens=java.base/java.lang=ALL-UNNAMED \
      -J-Djava.library.path=/usr/lib/x86_64-linux-gnu
```

The actual code to run the benchmark is identical to the last example, but is
reproduced here for completeness.

```clojure
user=> (require '[criterium.core :as bench])
nil
user=> (alter-var-root #'bench/estimated-overhead-cache (constantly 6.400703613065185E-9))
6.400703613065185E-9
user=> (require '[tech.v3.datatype.ffi :as dt-ffi])
nil
user=> (def fn-defs {:glfwInit {:rettype :int32} :glfwGetTime {:rettype :float64}})
#'user/fn-defs
user=> (def library-def (dt-ffi/define-library fn-defs))
#'user/library-def
user=> (def library-instance (dt-ffi/instantiate-library library-def "/usr/lib/libglfw.so"))
#'user/library-instance
user=> (def init (:glfwInit @library-instance))
#'user/init
user=> (init)
1
user=> (let [f (:glfwGetTime @library-instance)]
         (bench/bench (f) :verbose))
amd64 Linux 5.10.68-1-MANJARO 8 cpu(s)
OpenJDK 64-Bit Server VM 16.0.2+7
Runtime arguments: --add-modules=jdk.incubator.foreign -Dforeign.restricted=permit --add-opens=java.base/java.lang=ALL-UNNAMED -Djava.library.path=/usr/lib/x86_64-linux-gnu -Dclojure.basis=/home/jsusk/.clojure/.cpcache/2337051659.basis
Evaluation count : 1588513080 in 60 samples of 26475218 calls.
      Execution time sample mean : 58.732468 ns
             Execution time mean : 58.647361 ns
Execution time sample std-deviation : 9.732389 ns
    Execution time std-deviation : 9.791738 ns
   Execution time lower quantile : 31.318115 ns ( 2.5%)
   Execution time upper quantile : 65.449222 ns (97.5%)
                   Overhead used : 6.400704 ns

Found 14 outliers in 60 samples (23.3333 %)
	low-severe	 8 (13.3333 %)
	low-mild	 4 (6.6667 %)
	high-mild	 2 (3.3333 %)
 Variance from outliers : 87.6044 % Variance is severely inflated by outliers
nil
```

Not reproduced here, but notable for comparison, in my testing Java 16's version
of the JNI version performed about the same.

This is significantly faster than the JNA version of dtype-next, but it is still
slower than modern Panama. This is likely to simply be a result of optimizations
and changes to the Panama API, and when dtype-next is updated to use the Java 17
version of Panama I expect it will perform in line with coffi, but this
benchmark will be reproduced when this happens. Still, this shows that as it
stands, coffi is the fastest FFI available to Clojure developers.

## Known Issues
The project author is aware of these issues and plans to fix them in a future
release:

- When generating docs with codox in a library that depends on coffi, the below error will be produced. A temporary workaround is to add an explicit dependency in your codox build on insn at version 0.2.1
  ```
  Unable to find static field: ACC_OPEN in interface org.objectweb.asm.Opcodes
  ```
- Pointer wrapper types like `[::mem/pointer ::mem/int]` currently use one too many layers of indirection. This is fixed on develop.

## Future Plans
These features are planned for future releases.

- Support for va_args type
- Header parsing tool for generating a data model?
- Generic type aliases
- Unsigned integer types
- Record-based struct types
- Helper macro for out arguments
- Improve error messages from defcfn macro
- Mapped memory
- Helper macros for custom serde implementations for composite data types

## License

Copyright © 2023 Joshua Suskalo

Distributed under the Eclipse Public License version 1.0.
