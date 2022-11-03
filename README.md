## ziglang-learn

- string literals
string literals are const literals, pointer to array of u8. `[]const u8`
pointers to arrays coerce to slices, but constness is a one-way road, meaning that you can pass a mutable pointer/slice to a function that expects a const pointer/slice, but not vice versa.
```zig
const s="hello";
```

How to make a mutable string from a string literal then? Simple: dereference the pointer and you get an array, which can be assigned to local memory, which can then be used freely.
```zig
pub fn main() void {
   var foo = "hello".*;
   foo[0] = 'j'; // foo now equals "jello"
}
```
