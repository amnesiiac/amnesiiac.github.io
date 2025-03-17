---
layout: post
title: "rust vs cpp (rust intro)"
author: "melon"
date: 2025-01-05 15:47
categories: "2025"
tags:
  - cpp
  - rust
---

### # common type equivalents (rust vs cpp)

rust                                 | c++
---                                  | ---
bool                                 | ≈ bool, assuming sizeof(bool)==1
abibool::*                           | other boolean types
char                                 | ≈ char32_t + unicode scalar value guarantee
u8                                   | char as used for typical byte buffers
u8, u16, u32, u64                    | uint8_t, uint16_t, uint32_t, uint64_t
i8, i16, i32, i64                    | int8_t, int16_t, int32_t, int64_t
u128, i128                           | __uint128, __int128, but with different alignment
f32, f64                             | float, double, on most implementations
usize, isize (pointer sized)         | uintptr_t, intptr_t (pointer sized)
usize, isize (pointer sized)         | ≈ size_t, ptrdiff_t (could be smaller)
() – read as "unit"                  | void
! – read as "never"                  | [[noreturn]] void ?
&Struct – "(shareable) reference"    | const Struct &
&DynamicallySizedType                | std::tuple<const DynamicallySizedType *, size_t>
&dyn Trait                           | std::tuple<const TraitVtable *, const Struct &>
&Trait – Pre 1.27 version            | std::tuple<const TraitVtable *, const Struct &>
&mut T – read as "mutable reference" | Like !mut, but exclusive access and writable
std::option::Option\<T\>             | std::optional<T> (C++17)
std::result::Result<T, E>            | std::expected<T, E> (C++23)
std::result::Result<_, E>::Err       | std::unexpected<E> (C++23)
std::result::Result<T, _>::Ok        | T (implicitly converts to std::expected<T, _>)
std::boxed::Box<T>                   | unique_ptr<const T>, !null, value semantics
std::rc::Rc<T>                       | shared_ptr<const T>, !null, single threaded, value semantics
std::rc::Weak<T>                     | weak_ptr<const T>, single threaded
std::sync::Arc<T>                    | shared_ptr<const T>, !null, atomic refcount, value semantics
std::sync::Weak<T>                   | weak_ptr<const T>, atomic refcounts
std::option::Option<Box<T>>          | unique_ptr<const T>
std::option::Option<Arc<T>>          | shared_ptr<const T>
Option<Box<UnsafeCell<T>>>           | unique_ptr<T>
Option<Arc<UnsafeCell<T>>>           | shared_ptr<T>
Weak<UnsafeCell<T>>                  | weak_ptr<T> – Weak is nullable, so no Option necessary
RefCell<Weak<Self>> [example]        | std::enable_shared_from_this
std::cell::UnsafeCell<T>             | mutable specifier (unchecked access, avoid)
std::cell::Cell<T>                   | mutable specifier + value/copy semantics
std::cell::RefCell<T>                | mutable specifier + run time checked borrows
std::sync::RwLock<T>                 | mutable specifier + thread safe checked borrows
std::sync::Mutex<T>                  | mutable specifier + thread safe checked borrows (exclusive)
std::sync::atomic::AtomicBool        | std::atomic<bool>
std::sync::atomic::AtomicUsize       | std::atomic<size_t>
std::sync::Mutex<Struct>             | std::atomic<Struct>, note C++ may not be atomic!
std::sync::Mutex<Struct>             | std::mutex m; Struct s GUARDED_BY(m);
std::string::String                  | std::string, except guaranteed UTF8, no '\0'
&str                                 | std::string_view, except guaranteed UTF8
std::ffi::CString                    | std::string
std::ffi::CStr                       | std::string_view
std::ffi::OsString                   | ≈std::wstring on windows, but converted to WTF8(!)
std::path::PathBuf                   | ≈std::wstring on windows (just wraps OsString)
std::ffi::OsStr                      | ≈std::wstring_view, but wchar_t* isn't WTF8
std::path::Path                      | ≈std::wstring_view (just wraps OsStr)

<hr>

### # container equivalents (rust vs cpp)

rust container                   | cpp container
---                              | ---
std::vec::Vec\<T\>               | std::vector<T>
[T; 12]                          | std::array<T, 12> (C++11), T[12]
&[T]                             | std::span<T> (C++20)
std::collections::VecDeque<T>    | std::deque<T>
std::collections::LinkedList<T>  | std::list<T>
std::collections::BTreeSet<T>    | std::set<T>
std::collections::BTreeMap<K, V> | std::map<K, V>
std::collections::HashSet<T>     | std::unordered_set<T> (C++11)
std::collections::HashMap<K, V>  | std::unordered_map<K, V> (C++11)
std::collections::BinaryHeap<T>  | std::priority_queue<T>
                                 |
(A, B)                           | std::tuple<A, B>
struct T(A, B);                  | struct T : std::tuple<A, B> {};
struct T { a: A, b: B, }         | struct T { A a; B b; };
type Foo = u64;                  | typedef uint64_t Foo;
enum types                       | std::variant + names and language support
#[repr(u32)] enum E { A, B, C }  | enum class E : uint32_t { A, B, C }; [not ffi safe]
bitflags! { struct Flags ... }   | enum class Flags { A=0x1, B=0x2, C=0x4 }; [not ffi safe]
                                 |
#[repr(transparent)]             | // required for FFI safety
struct E(pub u32);               | enum class E : uint32_t {
impl E {                         |     A = 0
    const A : E = E(0);          | };
}                                |

<hr>

### # FFI type equivalents (rust vs cpp)

exact rust ffi type                | cpp
---                                | ---
()                                 | void (return type)
!                                  | [[noreturn]] void (return type) ?
std::os::raw::c_void*              | void*
...::raw::c_float, c_double        | float, double
...::raw::c_char, c_schar, c_uchar | char, signed char, unsigned char
...::raw::c_ushort, c_short        | unsigned short, short
...::raw::c_uint, c_int            | unsigned int, int
...::raw::c_ulong, c_long          | unsigned long, long
...::raw::c_ulonglong, c_longlong  | unsigned long long, long long

<hr>

### # syntax equivalents (rust vs cpp)

rust                                   | c++
---                                    | ---
assert!(condition);                    | assert(condition);
print!("Hello {}\n", world);           | printf("Hello %s\n", world);
if a == b { ... }                      | if (a == b) { ... }
else if a == b { ... }                 | else if (a == b) { ... }
else { ... }                           | else { ... }
                                       |
let foo = 42;                          | const auto foo = 42;
let foo = 42.0f32;                     | const auto foo = 42.0f;
let foo = 42u32;                       | const auto foo = 42u;
let foo = 42u64;                       | const auto foo = 42ull;
let mut foo = 42;                      | auto foo = 42;
let mut foo : i32 = 42;                | int32_t foo = 42;
let mut (x,y) = some_tuple;            | float x,y; std::tie(x,y) = some_tuple;
let some_tuple = (1,2);                | const auto some_tuple=std::make_tuple(1,2);
let a : [i32; 3] = [1, 2, 3];          | int32_t a[3] = { 1, 2, 3 };
let v : Vec\<i32\> = vec![1, 2, 3];    | std::vector<int32_t> v = { 1, 2, 3 };
                                       |
for i in v { ... }                     | for (auto i : v) { ... }
for i in 1..9 { ... }                  | for (size_t i = 1; i < 9; ++i) { ... }
for i in 1..=9 { ... }                 | for (size_t i = 1; i <= 9; ++i) { ... }
while i < 9 { ... }                    | while (i < 9) { ... }
                                       |
loop {                                 | while (true) {
    break;                             |     break;
    continue;                          |     continue;
}                                      | }
label: loop {                          | while (true) {
    break label;                       |     goto label_break;
    continue label;                    |     goto label_continue;
}                                      | label_continue: } label_break:
                                       |
other_var as f32                       | (float)other_var
                                       |
match var { ... }                      | switch (var) { ... }
42 => foo(),                           | case 42: foo(); break;
42 => { foo(); bar(); },               | case 42: foo(); bar(); break;
_ => ...,                              | default: ... break;
                                       |
mod foo { ... }                        | namespace foo { ... }
use foo as bar;                        | using namespace bar = foo;
use foo::*;                            | using namespace foo;
use foo::some_func;                    | using foo::some_func;
                                       |
/// Doc comment                        | // \brief Doc comment
fn add (a: i32, b: i32) -> i32 {       | int32_t add (int32_t a, int32_t b) {
    return a + b;                      |     return a + b;
    a + b // final statement, no ;     |     return a + b; // regular comment
}                                      | }
                                       |
trait Interface {                      | struct Interface {
    fn foo(&self, _: u32) -> u64;      |         virtual uint64_t foo(uint32_t) = 0;
    fn bar(&self, param: u32) -> u64 { |         virtual uint64_t bar(uint32_t param) {
        self.foo(param)                |             return this->foo(param);
    }                                  |         }
}                                      | };

<hr>

### # reference
https://maulingmonkey.com/guide/cpp-vs-rust/  
https://locka99.gitbooks.io/a-guide-to-porting-c-to-rust/content/ (todo)
