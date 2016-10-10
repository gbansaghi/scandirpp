# scandirpp
A C++ wrapper around [scandir(3)][scandir].

## About
This is a lightweight library providing wrapper functions around the Unix [`scandir`][scandir] function. It is resource-safe and compatible with modern C++ (in fact, it requires at least C++11).

## Usage
The entire library is contained in [scandirpp.h](scandirpp.h), so that is the only requirement. Once `#include`d, the library functions are accessible through the `scandirpp` namespace.

In the following paragraphs, the term "entry" will refer to the `struct dirent` objects returned by the call to `scandir`, while the term "value" will refer to a member of said objects. The [The Open Group Base Specifications][dirent] defines two such members: `d_name` (the filename) and `d_ino` (the file serial number). The library has built-in support for retrieving either entries or one of the two values mentioned. Accessing other values is also possible, see [Advanced usage][#advanced-usage].

### Basic usage
There are three high-level functions returning `std::vector`s of either entries or values:

Function | Member accessed | Return type
---------|-----------------|------------
`get_entries` | none (returns entire structure)   | `std::vector<struct dirent>`
`get_names`   | `d_name` | `std::vector<std::string>`
`get_inos`    | `d_ino`  | `std::vector<ino_t>`

The only required argument to these functions is an `std::string` specifying the path of the directory to be scanned (optional arguments are described in [Filtering](#filtering)).

Example:
```c++
// Print all filenames in the current working directory
for (const auto& name : scandirpp::get_names(".")) {
  std::cout << name << std::endl;
}
```

### Filtering
The `get_entries`, `get_names` and `get_inos` functions take two optional arguments: `value_filter` and `entry_filter`. These are callable objects (i.e. functions, function objects, lambdas, etc.) that take one argument and return true for entries/values that should be included in the results. Entry filters always take a `const struct dirent&` parameter, while value filters must take a `const T&` parameter where `T` is the type of the value being returned (see table in [Basic usage](#basic-usage)).

When retrieving results, the library functions apply the entry filter first. If it returns true, the value is accessed (see [Extractors](#extractors)) and the value filter is applied to it. If the value filter returns true, the value is added to the results. Both filters default to `scandirpp::default_filter`, defined as:

```c++
template<class T>
bool default_filter(const T&) { return true; }
```

## Advanced usage

### Extractors

[scandir]: http://man7.org/linux/man-pages/man3/scandir.3.html
[dirent]: http://pubs.opengroup.org/onlinepubs/009695399/basedefs/dirent.h.html
