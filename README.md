## ziglang-learn

read file 
https://ravendb.net/articles/implementing-a-file-pager-in-zig-reading-writing-from-the-disk
https://github.com/thomastay/phone-numbers-words/blob/master/zig/phone_number_words.zig#L84
https://gist.github.com/komuw/98111edb30262f0c34d93e3aa7f537ed

Learning Zig 
Today I’ve played around with Zig, the new, hip (is it hip?) programming language. I find it pretty neat. I’m going to walk you (and myself) through my first, very short, piece of code.

Below you can see the entirety of it. It basically allocates a 2MB buffer and reads a file into it… Yep, not particularly impressive, but this is a judgment free, learning zone, ok?!
```zig
const std = @import("std");
const warn = @import("std").debug.warn;
const Allocator = std.mem.Allocator;

fn readFile(allocator: *Allocator, filename: []const []const u8) ![]u8 {
    // Get the full path to the file.
    const path = try std.fs.path.resolve(allocator, filename);
    defer allocator.free(path);

    // Open file, allocate memory of the same size as the file, read file contents into that memory.
    const f: std.fs.File =
        try std.fs.openFileAbsolute(path, .{ .read = true });
    const f_stat = try f.stat();
    const result = try allocator.alloc(u8, f_stat.size);
    errdefer allocator.free(result);

    const read_result = try f.read(result);

    return result;
}
```
```zig
pub fn main() !void {
    var memory: [2 * 1024 * 1024]u8 = undefined;
    const allocator = &std.heap.FixedBufferAllocator.init(&memory).allocator;

    warn("Lets open a file...\n", .{});

    const file_contents = try readFile(allocator, &[_][]const u8{".", "Box.gltf"});
    warn("{x}\n", .{file_contents});
}
```
It starts with adding files to the build process with @import built-in function. What’s interesting is that source files are implicitly structs. That’s why a typical syntax for declaring variables applies. In Zig you can define methods inside a structure definition. That basically puts the function in a namespace with the name of that struct. So if std is a struct, the methods accessed via std are just functions in that structure namespace.

The const Allocator = std.mem.Allocator is a declaration. It declares an alias to std.mem.Allocator. I find this a bit confusing since it looks like a variable declaration. I think it makes sense though. It’s just a declaration. Declaration is an information passed to the compiler which says a thing, called like so, exists (scientific definition!). In Zig declaring a struct uses a very similar syntax: const Foo = struct {...};. Anyway, this line doesn’t create and instance of the Allocator. It just declares a new name for it.

Let’s jump to the main function. First of all the type it returns is !void meaning it can return a void or an error. That means that main can fail. If it can’t you can just change it to void. I think this feature comes down to having a valid Error Return Trace - that’s a very Zig thing. Not sure what’s going on here and I won’t go into it right now (why? because boring).

var memory: [2 * 1024 * 1024]u8 = undefined;
const allocator = &std.heap.FixedBufferAllocator.init(&memory).allocator;
In Zig memory allocations are very explicit. Zig doesn’t provide any default memory allocator. You can access the standard libc allocating functions like malloc, realloc, etc. via std.heap.c_allocator. In my case I’ve used the std.heap.FixedBufferAllocator, which gets inited with a stack buffer. It’s not the best choice for what I’m going for since stack very limited. It will do for now. I very much appreciate this idea of not providing a default memory allocator. That works well for C savages like me. We don’t like throwing mallocs and frees around. If you want to learn more read Memory and Choosing an Allocator sections of the documentation.

warn function outputs a message to stderr. Not much here, except maybe for the fact that you can’t skip the args argument, meaning that the .{} has to dangle there.

Time to talk about the function that reads the file, starting with its signature.

fn readFile(allocator: *Allocator, filename: []const []const u8) ![]u8 {
Firstly it takes the Allocator instance as the first argument. In Zig functions that allocate memory take an allocator as an argument. That way there are no hidden allocations. The second argument, []const []const u8 is an const array of const arrays. The return type is an error or a slice which in this case is a string. Arrays and slices are two very similar concepts. Arrays have a length known during compile time, while slices length is changing during run time. Let’s look at an example from the Zig’s documentation.

```zig
// Zig has no concept of strings. String literals are arrays of u8, and
// in general the string type is []u8 (slice of u8).
// Here we implicitly cast [5]u8 to []const u8
const hello: []const u8 = "hello";
```
This code creates an array holding string "hello". The hello variable is a const slice which points to that array.

Going back to my code…
```zig
const path = try std.fs.path.resolve(allocator, filename);
defer allocator.free(path);
```
First line uses the array of arrays as a filename. That’s because resolve function either resolves path with Windows or Posix standard (using / or \, etc.). It also takes an allocator. Seems like resolve uses some dynamic memory allocations and thus needs some cleanup. In Zig you can cleanup in a nice manner by using defer. You call something that allocates memory and you immediately call free prepended with defer. That means that when the scope ends, the memory will be freed. It’s a nice quality of life thing.

You can also see the try statement before calling the std.fs.path.resolve(). Most of the functions in the standard library return either an error or a proper result. try makes it so if the function returns an error, the entire scope returns with the same error.

The rest of the code in this function:
```zig
const f: std.fs.File =
    try std.fs.openFileAbsolute(path, .{ .read = true });
const f_stat = try f.stat();
const result = try allocator.alloc(u8, f_stat.size);
errdefer allocator.free(result);

const read_result = try f.read(result);
```
return result;
File gets opened via the std.fs function. Stats about the file get fetched. A memory chunk of the same size as the file size gets allocated, from the stack buffer. This time I’ve used the errdefer. That way, if the function errors out anywhere else, the memory gets freed. If it succeeds the function returns the pointer to the file contents. The last, few lines of code read the data into the allocated memory. read_result holds the amount of bytes read from the file. Function returns the pointer to the contents of the file.

As soon as the function returns…
```zig
const file_contents = try readFile(allocator, &[_][]const u8{".", "Box.gltf"});
```
The pointer to the file contents gets passed to a main scope variable. What’s worth pointing out is that I call the readFile function with a &[_][]const u8{".", "Box.gltf"} argument. [_] means that the compiler should infer the length of the array, which is easy enough. There are two elements in the array. The second [] is a slice… So in the end this is an array of slices. I don’t feel comfortable to explain how slices and arrays implicitly map one to another. I hope I’ll be able to explain it better in the future. Also, the & symbol is used to pass a address… even though it’s an array… Zig ain’t C.

Ending with warn… on a pointer? Well, more precisely on a slice. Yes, you can easily print any variable since they have a default implementation of format. You can implement a format for your own structs too. Nice!

That was my first hour with Zig. Writing this article has been very helpful to understand the more subtle parts. Let’s do it again soon!


https://zig.news/

https://zig.news/kristoff, https://zig.news/david_vanderson



integer numbers and float numbers
```zig
const a: i32 = 1;
const b: f32 = 1 / 3;
```
```zig

```
```zig

```
```zig

```
```zig

```
```zig

```
```zig

```
```zig

```
```zig

```


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
