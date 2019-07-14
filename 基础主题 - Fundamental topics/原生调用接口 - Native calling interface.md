# Native calling interface

Call into dynamic libraries that follow the C calling convention

# Getting started

The simplest imaginable use of `NativeCall` would look something like this:

```
use NativeCall;
sub some_argless_function() is native('something') { * }
some_argless_function();
```

The first line imports various traits and types. The next line looks like a relatively ordinary Perl 6 sub declaration—with a twist. We use the "native" trait in order to specify that the sub is actually defined in a native library. The platform-specific extension (e.g., `.so` or `.dll`), as well as any customary prefixes (e.g., 'lib') will be added for you.

The first time you call "some_argless_function", the "libsomething" will be loaded and the "some_argless_function" will be located in it. A call will then be made. Subsequent calls will be faster, since the symbol handle is retained.

Of course, most functions take arguments or return values—but everything else that you can do is just adding to this simple pattern of declaring a Perl 6 sub, naming it after the symbol you want to call and marking it with the "native" trait.

You will also need to declare and use native types. Please check [the native types page](https://docs.perl6.org/language/nativetypes) for more information.

# Changing names

Sometimes you want the name of your Perl subroutine to be different from the name used in the library you're loading. Maybe the name is long or has different casing or is otherwise cumbersome within the context of the module you are trying to create.

NativeCall provides a `symbol` trait for you to specify the name of the native routine in your library that may be different from your Perl subroutine name.

```
unit module Foo;
use NativeCall;
our sub init() is native('foo') is symbol('FOO_INIT') { * }
```

Inside of `libfoo` there is a routine called `FOO_INIT` but, since we're creating a module called Foo and we'd rather call the routine as `Foo::init`, we use the `symbol` trait to specify the name of the symbol in `libfoo` and call the subroutine whatever we want ("init" in this case).

# Passing and returning values

Normal Perl 6 signatures and the `returns` trait are used in order to convey the type of arguments a native function expects and what it returns. Here is an example.

```
use NativeCall;
sub add(int32, int32) returns int32 is native("calculator") { * }
```

Here, we have declared that the function takes two 32-bit integers and returns a 32-bit integer. You can find the other types that you may pass in the [native types](https://docs.perl6.org/language/nativetypes) page. Note that the lack of a `returns` trait is used to indicate `void` return type. Do *not*use the `void` type anywhere except in the Pointer parameterization.

For strings, there is an additional `encoded` trait to give some extra hints on how to do the marshaling.

```
use NativeCall;
sub message_box(Str is encoded('utf8')) is native('gui') { * }
```

To specify how to marshal string return types, just apply this trait to the routine itself.

```
use NativeCall;
sub input_box() returns Str is encoded('utf8') is native('gui') { * }
```

Note that a `NULL` string pointer can be passed by passing the Str type object; a `NULL` return will also be represented by the type object.

If the C function requires the lifetime of a string to exceed the function call, the argument must be manually encoded and passed as `CArray[uint8]`:

```
use NativeCall;
# C prototype is void set_foo(const char *) 
sub set_foo(CArray[uint8]) is native('foo') { * }
# C prototype is void use_foo(void) 
sub use_foo() is native('foo') { * } # will use pointer stored by set_foo() 
 
my $string = "FOO";
# The lifetime of this variable must be equal to the required lifetime of 
# the data passed to the C function. 
my $array = CArray[uint8].new($string.encode.list);
 
set_foo($array);
# ... 
use_foo();
# It's fine if $array goes out of scope starting from here. 
```

# Specifying the native representation

When working with native functions, sometimes you need to specify what kind of native data structure is going to be used. `is repr` is the term employed for that.

```
use NativeCall;
 
class timespec is repr('CStruct') {
    has uint32 $.tv_sec;
    has long $.tv_nanosecs;
}
 
sub clock_gettime(uint32 $clock-id, timespec $tspec --> uint32) is native { * };
 
my timespec $this-time .=new;
 
my $result = clock_gettime( 0, $this-time);
 
say "$result, $this-time"; # OUTPUT: «0, timespec<65385480>␤» 
```

The original function we are calling, [clock_gettime](https://linux.die.net/man/3/clock_gettime), uses a pointer to the `timespec` struct as second argument. We declare it as a [class](https://docs.perl6.org/syntax/class) here, but specify its representation as `is repr('CStruct')`, to indicate it corresponds to a C data structure. When we create an object of that class, we are creating exactly the kind of pointer `clock_gettime` expects. This way, data can be transferred seamlessly to and from the native interface.

# Basic use of pointers

When the signature of your native function needs a pointer to some native type (`int32`, `uint32`, etc.) all you need to do is declare the argument `is rw` :

```
use NativeCall;
# C prototype is void my_version(int *major, int *minor) 
sub my_version(int32 is rw, int32 is rw) is native('foo') { * }
my_version(my int32 $major, my int32 $minor); # Pass a pointer to 
```

Sometimes you need to get a pointer (for example, a library handle) back from a C library. You don't care about what it points to - you just need to keep hold of it. The Pointer type provides for this.

```
use NativeCall;
sub Foo_init() returns Pointer is native("foo") { * }
sub Foo_free(Pointer) is native("foo") { * }
```

This works out OK, but you may fancy working with a type named something better than Pointer. It turns out that any class with the representation "CPointer" can serve this role. This means you can expose libraries that work on handles by writing a class like this:

```
use NativeCall;
 
class FooHandle is repr('CPointer') {
    # Here are the actual NativeCall functions. 
    sub Foo_init() returns FooHandle is native("foo") { * }
    sub Foo_free(FooHandle) is native("foo") { * }
    sub Foo_query(FooHandle, Str) returns int8 is native("foo") { * }
    sub Foo_close(FooHandle) returns int8 is native("foo") { * }
 
    # Here are the methods we use to expose it to the outside world. 
    method new {
        Foo_init();
    }
 
    method query(Str $stmt) {
        Foo_query(self, $stmt);
    }
 
    method close {
        Foo_close(self);
    }
 
    # Free data when the object is garbage collected. 
    submethod DESTROY {
        Foo_free(self);
    }
}
```

Note that the CPointer representation can do nothing more than hold a C pointer. This means that your class cannot have extra attributes. However, for simple libraries this may be a neat way to expose an object oriented interface to it.

Of course, you can always have an empty class:

```
class DoorHandle is repr('CPointer') { }
```

And just use the class as you would use Pointer, but with potential for better type safety and more readable code.

Once again, type objects are used to represent NULL pointers.

# Function pointers

C libraries can expose pointers to C functions as return values of functions and as members of Structures like, e.g., structs and unions.

Example of invoking a function pointer "$fptr" returned by a function "f", using a signature defining the desired function parameters and return value:

```
sub f() returns Pointer is native('mylib') { * }
 
my $fptr    = f();
my &newfunc = nativecast(:(Str, size_t --> int32), $fptr);
 
say newfunc("test", 4);
```

# Arrays

NativeCall has some support for arrays. It is constrained to work with machine-size integers, doubles and strings, sized numeric types, arrays of pointers, arrays of structs, and arrays of arrays.

Perl 6 arrays, which support amongst other things laziness, are laid out in memory in a radically different way to C arrays. Therefore, the NativeCall library offers a much more primitive CArray type, which you must use if working with C arrays.

Here is an example of passing a C array.

```
sub RenderBarChart(Str, int32, CArray[Str], CArray[num64]) is native("chart") { * }
my @titles := CArray[Str].new;
@titles[0]  = 'Me';
@titles[1]  = 'You';
@titles[2]  = 'Hagrid';
my @values := CArray[num64].new;
@values[0]  = 59.5e0;
@values[1]  = 61.2e0;
@values[2]  = 180.7e0;
RenderBarChart('Weights (kg)', 3, @titles, @values);
```

Note that binding was used to `@titles`, *not* assignment! If you assign, you are putting the values into a Perl 6 array, and it will not work out. If this all freaks you out, forget you ever knew anything about the `@` sigil and just use `$` all the way when using NativeCall.

```
use NativeCall;
my $titles = CArray[Str].new;
$titles[0] = 'Me';
$titles[1] = 'You';
$titles[2] = 'Hagrid';
```

Getting return values for arrays works out just the same.

Some library APIs may take an array as a buffer that will be populated by the C function and, for instance, return the actual number of items populated:

```
use NativeCall;
sub get_n_ints(CArray[int32], int32) returns int32 is native('ints') { * }
```

In these cases it is important that the CArray has at least the number of elements that are going to be populated before passing it to the native subroutine, otherwise the C function may stomp all over Perl's memory leading to possibly unpredictable behavior:

```
my $number_of_ints = 10;
my $ints = CArray[int32].allocate($number_of_ints); # instantiates an array with 10 elements 
my $n = get_n_ints($ints, $number_of_ints);
```

*Note*: `allocate` was introduced in Rakudo 2018.05. Before that, you had to use this mechanism to extend an array to a number of elements:

```
my $ints = CArray[int32].new;
my $number_of_ints = 10;
$ints[$number_of_ints - 1] = 0; # extend the array to 10 items 
```

The memory management of arrays is important to understand. When you create an array yourself, then you can add elements to it as you wish and it will be expanded for you as required. However, this may result in it being moved in memory (assignments to existing elements will never cause this, however). This means you'd best know what you're doing if you twiddle with an array after passing it to a C library.

By contrast, when a C library returns an array to you, then the memory can not be managed by NativeCall, and it doesn't know where the array ends. Presumably, something in the library API tells you this (for example, you know that when you see a null element, you should read no further). Note that NativeCall can offer you no protection whatsoever here - do the wrong thing, and you will get a segfault or cause memory corruption. This isn't a shortcoming of NativeCall, it's the way the big bad native world works. Scared? Here, have a hug. Good luck!



## CArray methods

Besides the usual methods available on every Perl 6 instance, `CArray` provides the following methods that can be used to interact with the it from the Perl 6 point of view:

- `elems` provides the number of elements within the array;
- `AT-POS` provides a specific element at the given position (starting from zero);
- `list` provides the [List](https://docs.perl6.org/type/List) of elements within the array building it from the native array iterator.

As an example, consider the following simple piece of code:

```
use NativeCall;
 
my $native-array = CArray[int32].new( 1, 2, 3, 4, 5 );
say 'Number of elements: ' ~ $native-array.elems;
 
# walk the array 
for $native-array.list -> $elem {
    say "Current element is: $elem";
}
 
# get every element by its index-based position 
for 0..$native-array.elems - 1 -> $position {
    say "Element at position $position is "
          ~ $native-array.AT-POS( $position );
}
 
```

that produces the following output

```
Number of elements: 5
Current element is: 1
Current element is: 2
Current element is: 3
Current element is: 4
Current element is: 5
Element at position 0 is 1
Element at position 1 is 2
Element at position 2 is 3
Element at position 3 is 4
Element at position 4 is 5
```

# Structs

Thanks to representation polymorphism, it's possible to declare a normal looking Perl 6 class that, under the hood, stores its attributes in the same way a C compiler would lay them out in a similar struct definition. All it takes is a quick use of the "repr" trait:

```
class Point is repr('CStruct') {
    has num64 $.x;
    has num64 $.y;
}
```

The attributes can only be of the types that NativeCall knows how to marshal into struct fields. Currently, structs can contain machine-sized integers, doubles, strings, and other NativeCall objects (CArrays, and those using the CPointer and CStruct reprs). Other than that, you can do the usual set of things you would with a class; you could even have some of the attributes come from roles or have them inherited from another class. Of course, methods are completely fine too. Go wild!

CStruct objects are passed to native functions by reference and native functions must also return CStruct objects by reference. The memory management rules for these references are very much like the rules for arrays, though simpler since a struct is never resized. When you create a struct, the memory is managed for you and when the variable(s) pointing to the instance of a CStruct go away, the memory will be freed when the GC gets to it. When a CStruct-based type is used as the return type of a native function, the memory is not managed for you by the GC.

NativeCall currently doesn't put object members in containers, so assigning new values to them (with =) doesn't work. Instead, you have to bind new values to the private members:

```
class MyStruct is repr('CStruct') {
    has CArray[num64] $!arr;
    has Str $!str;
    has Point $!point; # Point is a user-defined class 
 
    submethod TWEAK {
        my $arr := CArray[num64].new;
        $arr[0] = 0.9e0;
        $arr[1] = 0.2e0;
        $!arr := $arr;
        $!str := 'Perl 6 is fun';
        $!point := Point.new;
    }
}
```

As you may have predicted by now, a NULL pointer is represented by the type object of the struct type.

## CUnions

Likewise, it is possible to declare a Perl 6 class that stores its attributes the same way a C compiler would lay them out in a similar `union` definition; using the `CUnion` representation:

```
use NativeCall;
 
class MyUnion is repr('CUnion') {
    has int32 $.flags32;
    has int64 $.flags64;
}
 
say nativesizeof(MyUnion.new);  # OUTPUT: «8␤» 
                                # ie. max(sizeof(MyUnion.flags32), sizeof(MyUnion.flags64)) 
```

## Embedding CStructs and CUnions

CStructs and CUnions can be in turn referenced by—or embedded into—a surrounding CStruct and CUnion. To say the former we use `has` as usual, and to do the latter we use the `HAS` declarator instead:

```
class MyStruct is repr('CStruct') {
    has Point $.point;  # referenced 
    has int32 $.flags;
}
 
say nativesizeof(MyStruct.new);  # OUTPUT: «16␤» 
                                 # ie. sizeof(struct Point *) + sizeof(int32_t) 
 
class MyStruct2 is repr('CStruct') {
    HAS Point $.point;  # embedded 
    has int32 $.flags;
}
 
say nativesizeof(MyStruct2.new);  # OUTPUT: «24␤» 
                                  # ie. sizeof(struct Point) + sizeof(int32_t) 
```

## Notes on memory management

When allocating a struct for use as a struct, make sure that you allocate your own memory in your C functions. If you're passing a struct into a C function which needs a `Str`/`char*` allocated ahead of time, be sure to assign a container for a variable of type `Str` prior to passing your struct into the function.

### In your Perl 6 code...

```
class AStringAndAnInt is repr("CStruct") {
  has Str $.a_string;
  has int32 $.an_int32;
 
  sub init_struct(AStringAndAnInt is rw, Str, int32) is native('simple-struct') { * }
 
  submethod BUILD(:$a_string, :$an_int) {
    init_struct(self, $a_string, $an_int);
  }
}
 
```

In this code we first set up our members, `$.a_string` and `$.an_int32`. After that we declare our `init_struct()` function for the `init()` method to wrap around; this function is then called from `BUILD` to effectively assign the values before returning the created object.

### In your C code...

```
typedef struct a_string_and_an_int32_t_ {
  char *a_string;
  int32_t an_int32;
} a_string_and_an_int32_t;
 
```

Here's the structure. Notice how we've got a `char *` there.

```
void init_struct(a_string_and_an_int32_t *target, char *str, int32_t int32) {
  target->an_int32 = int32;
  target->a_string = strdup(str);
 
  return;
}
 
```

In this function we initialize the C structure by assigning an integer by value, and passing the string by reference. The function allocates memory that it points <char *a_string> to within the structure as it copies the string. (Note you will also have to manage deallocation of the memory as well to avoid memory leaks.)

```
# A long time ago in a galaxy far, far away... 
my $foo = AStringAndAnInt.new(a_string => "str", an_int => 123);
say "foo is {$foo.a_string} and {$foo.an_int32}";
# OUTPUT: «foo is str and 123␤» 
```

# Typed pointers

You can type your `Pointer` by passing the type as a parameter. It works with the native type but also with `CArray` and `CStruct`defined types. NativeCall will not implicitly allocate the memory for it even when calling `new` on them. It's mostly useful in the case of a C routine returning a pointer, or if it's a pointer embedded in a `CStruct`.

```
use NativeCall;
sub strdup(Str $s --> Pointer[Str]) is native {*}
my Pointer[Str] $p = strdup("Success!");
say $p.deref;
```

You have to call `.deref` on `Pointer`s to access the embedded type. In the example above, declaring the type of the pointer avoids typecasting error when dereferenced. Please note that the original [`strdup`](https://en.cppreference.com/w/c/experimental/dynamic/strdup) returns a pointer to `char`; we are using `Pointer<Str>`.

```
my Pointer[int32] $p; #For a pointer on int32; 
my Pointer[MyCstruct] $p2 = some_c_routine();
my MyCstruct $mc = $p2.deref;
say $mc.field1;
```

It's quite common for a native function to return a pointer to an array of elements. Typed pointers can be dereferenced as an array to obtain individual elements.

```
my $n = 5;
# returns a pointer to an array of length $n 
my Pointer[Point] $plot = some_other_c_routine($n);
# display the 5 elements in the array 
for 1 .. $n -> $i {
    my $x = $plot[$i - 1].x;
    my $y = $plot[$i - 1].y;
    say "$i: ($x, $y)";
}
```

Pointers can also be updated to reference successive elements in the array:

```
my Pointer[Point] $elem = $plot;
# show differences between successive points 
for 1 ..^ $n {
    my Point $lo = $elem.deref;
    ++$elem; # equivalent to $elem = $elem.add(1); 
    my Point $hi = (++$elem).deref;
    my $dx = $hi.x = $lo.x;
    my $dy = $hi.y = $lo.y;
    say "$_: delta ($dx, $dy)";
}
```

Void pointers can also be used by declaring them `Pointer[void]`. Please consult [the native types documentation](https://docs.perl6.org/language/nativetypes#The_void_type) for more information on the subject.

# Strings

## Explicit memory management

Let's say there is some C code that caches strings passed, like so:

```
#include <stdlib.h> 
 
static char *__VERSION;
 
char *
get_version()
{
    return __VERSION;
}
 
char *
set_version(char *version)
{
    if (__VERSION != NULL) free(__VERSION);
    __VERSION = version;
    return __VERSION;
}
```

If you were to write bindings for `get_version` and `set_version`, they would initially look like this, but will not work as intended:

```
sub get_version(--> Str)     is native('./version') { * }
sub set_version(Str --> Str) is native('./version') { * }
 
say set_version('1.0.0'); # 1.0.0 
say get_version;          # Differs on each run 
say set_version('1.0.1'); # Double free; segfaults 
```

This code segfaults on the second `set_version` call because it tries to free the string passed on the first call after the garbage collector had already done so. If the garbage collector shouldn't free a string passed to a native function, use `explicitly-manage` with it:

```
say set_version(explicitly-manage('1.0.0')); # 1.0.0 
say get_version;                             # 1.0.0 
say set_version(explicitly-manage('1.0.1')); # 1.0.1 
say get_version;                             # 1.0.1 
```

Bear in mind all memory management for explicitly managed strings must be handled by the C library itself or through the NativeCall API to prevent memory leaks.

## Buffers and blobs

[Blob](https://docs.perl6.org/type/Blob)s and [Buf](https://docs.perl6.org/type/Buf)s are the Perl 6 way of storing binary data. We can use them for interchange of data with native functions and data structures, although not directly. We will have to use [`nativecast`](https://docs.perl6.org/routine/nativecast).

```
my $blob = Blob.new(0x22, 0x33);
my $src = nativecast(Pointer, $blob);
```

This `$src` can then be used as an argument for any native function that takes a Pointer. The opposite, putting values pointed to by a `Pointer` into a `Buf` or using it to initialize a `Blob` is not directly supported. You might want to use [`NativeHelpers::Blob`](https://github.com/salortiz/NativeHelpers-Blob)to do this kind of operations.

```
my $esponja = blob-from-pointer( $inter, :2elems, :type(Blob[int8]));
say $esponja;
```

# Function arguments

NativeCall also supports native functions that take functions as arguments. One example of this is using function pointers as callbacks in an event-driven system. When binding these functions via NativeCall, one need only provide the equivalent signature as [a constraint on the code parameter](https://docs.perl6.org/type/Signature#Constraining_signatures_of_Callables):

```
use NativeCall;
# void SetCallback(int (*callback)(const char *)) 
my sub SetCallback(&callback (Str --> int32)) is native('mylib') { * }
```

Note: the native code is responsible for memory management of values passed to Perl 6 callbacks this way. In other words, NativeCall will not free() strings passed to callbacks.

# Library paths and names

The `native` trait accepts the library name, the full path, or a subroutine returning either of the two. When using the library name, the name is assumed to be prepended with "lib" and appended with ".so" (or just appended with ".dll" on Windows), and will be searched for in the paths in the LD_LIBRARY_PATH (PATH on Windows) environment variable.

```
use NativeCall;
constant LIBMYSQL = 'mysqlclient';
constant LIBFOO = '/usr/lib/libfoo.so.1';
sub LIBBAR {
    my $path = qx/pkg-config --libs libbar/.chomp;
    $path ~~ s/\/[[\w+]+ % \/]/\0\/bar/;
    $path
}
# and later 
 
sub mysql_affected_rows returns int32 is native(LIBMYSQL) {*};
sub bar is native(LIBFOO) {*}
sub baz is native(LIBBAR) {*}
```

You can also put an incomplete path like './foo' and NativeCall will automatically put the right extension according to the platform specification.

BE CAREFUL: the `native` trait and `constant` are evaluated at compile time. Don't write a constant that depends on a dynamic variable like:

```
# WRONG: 
constant LIBMYSQL = %*ENV<P6LIB_MYSQLCLIENT> || 'mysqlclient';
```

This will keep the value given at compile time. A module will be precompiled and `LIBMYSQL` will keep the value it acquires when the module gets precompiled.

## ABI/API version

If you write `native('foo')` NativeCall will search libfoo.so under Unix like system (libfoo.dynlib on OS X, foo.dll on win32). In most modern system it will require you or the user of your module to install the development package because it's recommended to always provide an API/ABI version to a shared library, so libfoo.so ends often being a symbolic link provided only by a development package.

To avoid that, the `native` trait allows you to specify the API/ABI version. It can be a full version or just a part of it. (Try to stick to Major version, some BSD code does not care for Minor.)

```
use NativeCall;
sub foo1 is native('foo', v1) {*} # Will try to load libfoo.so.1 
sub foo2 is native('foo', v1.2.3) {*} # Will try to load libfoo.so.1.2.3 
 
my List $lib = ('foo', 'v1');
sub foo3 is native($lib) {*}
```

## Routine

The `native` trait also accepts a `Callable` as argument, allowing you to provide your own way to handle the way it will find the library file to load.

```
use NativeCall;
sub foo is native(sub {'libfoo.so.42'}) {*}
```

It will only be called at the first invocation of the sub.

## Calling into the standard library

If you want to call a C function that's already loaded, either from the standard library or from your own program, you can omit the value, so `is native`.

For example on a UNIX-like operating system, you could use the following code to print the home directory of the current user:

```
use NativeCall;
my class PwStruct is repr('CStruct') {
    has Str $.pw_name;
    has Str $.pw_passwd;
    has uint32 $.pw_uid;
    has uint32 $.pw_gid;
    has Str $.pw_gecos;
    has Str $.pw_dir;
    has Str $.pw_shell;
}
sub getuid()              returns uint32   is native { * };
sub getpwuid(uint32 $uid) returns PwStruct is native { * };
 
say getpwuid(getuid()).pw_dir;
```

Though of course `$*HOME` is a much easier way :-)

# Exported variables

Variables exported by a library – also named "global" or "extern" variables – can be accessed using `cglobal`. For example:

```
my $var := cglobal('libc.so.6', 'errno', int32)
```

This code binds to `$var` a new [Proxy](https://docs.perl6.org/type/Proxy) object that redirects all its accesses to the integer variable named "errno" as exported by the "libc.so.6" library.

# C++ support

NativeCall offers support to use classes and methods from C++ as shown in <https://github.com/rakudo/rakudo/blob/master/t/04-nativecall/13-cpp-mangling.t> (and its associated C++ file). Note that at the moment it's not as tested and developed as C support.

# Helper functions

The `NativeCall` library exports several subroutines to help you work with data from native libraries.

## sub nativecast

```
sub nativecast($target-type, $source) is export(:DEFAULT)
```

This will *cast* the Pointer `$source` to an object of `$target-type`. The source pointer will typically have been obtained from a call to a native subroutine that returns a pointer or as a member of a `struct`, this may be specified as `void *` in the `C` library definition for instance, but you may also cast from a pointer to a less specific type to a more specific one.

As a special case, if a [Signature](https://docs.perl6.org/type/Signature) is supplied as `$target-type` then a `subroutine` will be returned which will call the native function pointed to by `$source` in the same way as a subroutine declared with the `native` trait. This is described in [Function Pointers](https://docs.perl6.org/language/nativecall#Function_pointers).

## sub cglobal

```
sub cglobal($libname, $symbol, $target-type) is export is rw
```

This returns a [Proxy](https://docs.perl6.org/type/Proxy) object that provides access to the `extern` named `$symbol` that is exposed by the specified library. The library can be specified in the same ways that they can be to the `native` trait.

## sub nativesizeof

```
sub nativesizeof($obj) is export(:DEFAULT)
```

This returns the size in bytes of the supplied object, it can be thought of as being equivalent to `sizeof` in **C**. The object can be a builtin native type such as `int64` or `num64`, a `CArray` or a class with the `repr` `CStruct`, `CUnion` or `CPointer`.

## sub explicitly-manage

```
sub explicitly-manage($str) is export(:DEFAULT)
```

This returns a `CStr` object for the given `Str`. If the string returned is passed to a NativeCall subroutine, it will not be freed by the runtime's garbage collector.

# Examples

Some specific examples, and instructions to use examples above in particular platforms.

## PostgreSQL

The PostgreSQL examples in [DBIish](https://github.com/perl6/DBIish/blob/master/examples/pg.p6) make use of the NativeCall library and `is native` to use the native `_putenv` function call in Windows.

## MySQL

**NOTE:** Please bear in mind that, under the hood, Debian has substituted MySQL with MariaDB since the Stretch version, so if you want to install MySQL, use [MySQL APT repository](https://dev.mysql.com/downloads/repo/apt/) instead of the default repository.

To use the MySQL example in [DBIish](https://github.com/perl6/DBIish/blob/master/examples/mysql.p6), you'll need to install MySQL server locally; on Debian-esque systems it can be installed with something like:

```
wget https://dev.mysql.com/get/mysql-apt-config_0.8.10-1_all.deb
sudo dpkg -i mysql-apt-config_0.8.10-1_all.deb # Don't forget to select 5.6.x 
sudo apt-get update
sudo apt-get install mysql-community-server -y
sudo apt-get install libmysqlclient18 -y
```

Prepare your system along these lines before trying out the examples:

```
$ mysql -u root -p
SET PASSWORD = PASSWORD('sa');
DROP DATABASE test;
CREATE DATABASE test;
```

## Microsoft Windows

Here is an example of a Windows API call:

```
use NativeCall;
 
sub MessageBoxA(int32, Str, Str, int32)
    returns int32
    is native('user32')
    { * }
 
MessageBoxA(0, "We have NativeCall", "ohai", 64);
```

## Short tutorial on calling a C function

This is an example for calling a standard function and using the returned information in a Perl 6 program.

`getaddrinfo` is a POSIX standard function for obtaining network information about a network node, e.g., `google.com`. It is an interesting function to look at because it illustrates a number of the elements of NativeCall.

The Linux manual provides the following information about the C callable function:

```
int getaddrinfo(const char *node, const char *service,
       const struct addrinfo *hints,
       struct addrinfo **res);
```

The function returns a response code 0 = error, 1 = success. The data are extracted from a linked list of `addrinfo` elements, with the first element pointed to by `res`.

From the table of NativeCall Types we know that an `int` is `int32`. We also know that a `char *` is one of the forms C for a C `Str`, which maps simply to Str. But `addrinfo` is a structure, which means we will need to write our own Type class. However, the function declaration is straightforward:

```
sub getaddrinfo( Str $node, Str $service, Addrinfo $hints, Pointer $res is rw )
    returns int32
    is native
    { * }
```

Note that $res is to be written by the function, so it must be labeled as rw. Since the library is standard POSIX, the library name can be the Type definition or null.

We now have to handle structure Addrinfo. The Linux Manual provides this information:

```
struct addrinfo {
               int              ai_flags;
               int              ai_family;
               int              ai_socktype;
               int              ai_protocol;
               socklen_t        ai_addrlen;
               struct sockaddr *ai_addr;
               char            *ai_canonname;
               struct addrinfo *ai_next;
           };
```

The `int, char*` parts are straightforward. Some research indicates that `socklen_t` can be architecture dependent, but is an unsigned integer of at least 32 bits. So `socklen_t` can be mapped to the `uint32` type.

The complication is `sockaddr` which differs depending on whether `ai_socktype` is undefined, INET, or INET6 (a standard v4 IP address or a v6 address).

So we create a Perl 6 `class` to map to the C `struct addrinfo`; while we're at it, we also create another class for `SockAddr` which is needed for it.

```
class SockAddr is repr('CStruct') {
    has int32    $.sa_family;
    has Str      $.sa_data;
}
 
class Addrinfo is repr('CStruct') {
    has int32     $.ai_flags;
    has int32     $.ai_family;
    has int32     $.ai_socktype;
    has int32     $.ai_protocol;
    has int32     $.ai_addrlen;
    has SockAddr  $.ai_addr       is rw;
    has Str       $.ai_cannonname is rw;
    has Addrinfo  $.ai_next       is rw;
 
}
```

The `is rw` on the last three attributes reflects that these were defined in C to be pointers.

The important thing here for mapping to a C `Struct` is the structure of the state part of the class, that is the attributes. However, a class can have methods and `NativeCall` does not 'touch' them for mapping to C. This means that we can add extra methods to the class to unpack the attributes in a more readable manner, e.g.,

```
method flags {
    do for AddrInfo-Flags.enums { .key if $!ai_flags +& .value }
}
```

By defining an appropriate `enum`, `flags` will return a string of keys rather than a bit packed integer.

The most useful information in the `sockaddr` structure is the address of node, which depends on the family of the Socket. So we can add method `address` to the Perl 6 class that interprets the address depending on the family.

In order to get a human readable IP address, there is the C function `inet_ntop` which returns a `char *` given a buffer with the `addrinfo`.

Putting all these together, leads to the following program:

```
#!/usr/bin/env perl6 
 
use v6;
use NativeCall;
 
constant \INET_ADDRSTRLEN = 16;
constant \INET6_ADDRSTRLEN = 46;
 
enum AddrInfo-Family (
    AF_UNSPEC                   => 0;
    AF_INET                     => 2;
    AF_INET6                    => 10;
);
 
enum AddrInfo-Socktype (
    SOCK_STREAM                 => 1;
    SOCK_DGRAM                  => 2;
    SOCK_RAW                    => 3;
    SOCK_RDM                    => 4;
    SOCK_SEQPACKET              => 5;
    SOCK_DCCP                   => 6;
    SOCK_PACKET                 => 10;
);
 
enum AddrInfo-Flags (
    AI_PASSIVE                  => 0x0001;
    AI_CANONNAME                => 0x0002;
    AI_NUMERICHOST              => 0x0004;
    AI_V4MAPPED                 => 0x0008;
    AI_ALL                      => 0x0010;
    AI_ADDRCONFIG               => 0x0020;
    AI_IDN                      => 0x0040;
    AI_CANONIDN                 => 0x0080;
    AI_IDN_ALLOW_UNASSIGNED     => 0x0100;
    AI_IDN_USE_STD3_ASCII_RULES => 0x0200;
    AI_NUMERICSERV              => 0x0400;
);
 
sub inet_ntop(int32, Pointer, Blob, int32 --> Str)
    is native {}
 
class SockAddr is repr('CStruct') {
    has uint16 $.sa_family;
}
 
class SockAddr-in is repr('CStruct') {
    has int16 $.sin_family;
    has uint16 $.sin_port;
    has uint32 $.sin_addr;
 
    method address {
        my $buf = buf8.allocate(INET_ADDRSTRLEN);
        inet_ntop(AF_INET, Pointer.new(nativecast(Pointer,self)+4),
            $buf, INET_ADDRSTRLEN)
    }
}
 
class SockAddr-in6 is repr('CStruct') {
    has uint16 $.sin6_family;
    has uint16 $.sin6_port;
    has uint32 $.sin6_flowinfo;
    has uint64 $.sin6_addr0;
    has uint64 $.sin6_addr1;
    has uint32 $.sin6_scope_id;
 
    method address {
        my $buf = buf8.allocate(INET6_ADDRSTRLEN);
        inet_ntop(AF_INET6, Pointer.new(nativecast(Pointer,self)+8),
            $buf, INET6_ADDRSTRLEN)
    }
}
 
class Addrinfo is repr('CStruct') {
    has int32 $.ai_flags;
    has int32 $.ai_family;
    has int32 $.ai_socktype;
    has int32 $.ai_protocol;
    has uint32 $.ai_addrNativeCalllen;
    has SockAddr $.ai_addr is rw;
    has Str $.ai_cannonname is rw;
    has Addrinfo $.ai_next is rw;
 
    method flags {
        do for AddrInfo-Flags.enums { .key if $!ai_flags +& .value }
    }
 
    method family {
        AddrInfo-Family($!ai_family)
    }
 
    method socktype {
        AddrInfo-Socktype($!ai_socktype)
    }
 
    method address {
        given $.family {
            when AF_INET {
                nativecast(SockAddr-in, $!ai_addr).address
            }
            when AF_INET6 {
                nativecast(SockAddr-in6, $!ai_addr).address
            }
        }
    }
}
 
sub getaddrinfo(Str $node, Str $service, Addrinfo $hints,
                Pointer $res is rw --> int32)
    is native {};
 
sub freeaddrinfo(Pointer)
    is native {}
 
sub MAIN() {
    my Addrinfo $hint .= new(:ai_flags(AI_CANONNAME));
    my Pointer $res .= new;
    my $rv = getaddrinfo("google.com", Str, $hint, $res);
    say "return val: $rv";
    if ( ! $rv ) {
        my $addr = nativecast(Addrinfo, $res);
        while $addr {
            with $addr {
                say "Name: ", $_ with .ai_cannonname;
                say .family, ' ', .socktype;
                say .address;
                $addr = .ai_next;
            }
        }
    }
    freeaddrinfo($res);
}
```

This produces the following output:

```
return val: 0
Name: google.com
AF_INET SOCK_STREAM
216.58.219.206
AF_INET SOCK_DGRAM
216.58.219.206
AF_INET SOCK_RAW
216.58.219.206
AF_INET6 SOCK_STREAM
2607:f8b0:4006:800::200e
AF_INET6 SOCK_DGRAM
2607:f8b0:4006:800::200e
AF_INET6 SOCK_RAW
2607:f8b0:4006:800::200e
```