# scandirpp
A C++ wrapper around [scandir(3)][scandir].

## About
This is a lightweight library providing wrapper functions around the Unix [`scandir`][scandir] function. It is resource-safe and compatible with modern C++ (in fact, it requires at least C++11).

## Usage
The entire library is contained in [scandirpp.h](scandirpp.h), so that is the only requirement. Once `#include`d, the library functions are accessible through the `scandirpp` namespace.

In the following paragraphs, the term "entry" will refer to the `struct dirent` objects returned by the call to `scandir`, while the term "value" will refer to a member of said objects. The [The Open Group Base Specifications][dirent] defines two such members: `d_name` (the filename) and `d_ino` (the file serial number). The library has built-in support for retrieving either entries or one of the two values mentioned. Accessing other values is also possible, see [Extractors](#extractors).

### Basic usage
There are three high-level functions returning `std::vector`s of either entries or values:

Function | Member accessed | Return type
---------|-----------------|------------
`get_entries` | none (returns entire structure)   | `std::vector<struct dirent>`
`get_names`   | `d_name` | `std::vector<std::string>`
`get_inos`    | `d_ino`  | `std::vector<ino_t>`

These functions share the same signature (default values omitted for clarity):

```c++
template<class ValueFilter, class EntryFilter>
/*...*/ (const std::string& dir, ValueFilter&& value_filter, EntryFilter&& entry_filter)
```

The only required argument is `dir`, specifying the path of the directory to be scanned (the other two arguments are described in [Filtering](#filtering)).

Example:
```c++
// Print all filenames in the current working directory
for (const auto& name : scandirpp::get_names(".")) {
  std::cout << name << std::endl;
}
```

### Filtering
The `get_entries`, `get_names` and `get_inos` functions take two optional arguments: `value_filter` and `entry_filter` (in that order). These are callable objects (i.e. functions, function objects, lambdas, etc.) that take one argument and return true for entries/values that should be included in the results. Entry filters always take a `const struct dirent&` parameter, while value filters must take a `const T&` parameter where `T` is the type of the value being returned (see table in [Basic usage](#basic-usage)).

Entry filters are more robust in the sense that they let you filter by any property of the `struct dirent` for a file, irrespective of the value that is retrieved from it (e.g. you could filter by name even if you access `d_ino`). Value filters can only operate on the retrieved value, but they can be written without any knowledge about the internals of `struct dirent`s.

When retrieving results, the library functions apply the entry filter first. If it returns true, the value is accessed (see [Extractors](#extractors)) and the value filter is applied to it. If the value filter returns true, the value is added to the results. Both filters default to `scandirpp::default_filter`, defined as:

```c++
template<class T>
bool default_filter(const T&) { return true; }
```

Example:
```c++
// Print all filenames except the "." and ".." entries.

auto filter = [](const std::string& name) {
  return (name != "." && name != "..");
};

for (const auto& name : scandirpp::get_names(".", filter)) {
  std::cout << name << std::endl;
}
```

Example:
```c++
// To specify an entry filter but no value filter, use default_filter
for (const auto& name : scandirpp::get_names(".", scandirpp::default_filter, entry_filter)) {
  std::cout << name << std::endl;
}
```

## Exceptions
The library defines the `scandirpp::ScandirException` class, which derives from `std::system_error`. It is thrown whenever `scandir` returns with a negative number (indicating an error), using the error number and description from `errno`.

## Advanced usage
### Extractors
As mentioned in [Basic usage](#basic-usage), the standard defines two data members for `struct dirent`: `d_name` and `d_ino`. However, your system may provide additional members. For example, under Xubuntu 16.04, there are three additional members: `d_off`, `d_reclen` and `d_type`. While it is certainly possible to access these extra members by using `get_entries`, that approach could lead to redundant and less maintainable code. Instead, extra members can be accessed by defining a custom extractor.

Exractors are callables taking a `const struct dirent&` and returning the desired value. For example, `get_names` uses the `scandirpp::extract_name` method, defined as:

```c++
inline std::string extract_name(const struct dirent& entry) {
    return entry.d_name;
}
```

Similarly, `get_entries` and `get_inos` use `extract_entry` and `extract_ino`, respectively.

Custom extractors can be used with `scandirpp::get_vector`. This function behaves exactly like `get_entries`, `get_names` etc. except it must be passed the extractor as its second argument (default values omitted for clarity):

```c++
template<class ValueExtractor, class ValueType, class ValueFilter, class EntryFilter>
std::vector<ValueType> get_vector(const std::string& dir,
                                  ValueExtractor&& value_extractor,
                                  ValueFilter&& value_filter,
                                  EntryFilter&& entry_filter)
```

Example:

```c++
// Define a custom extractor for d_reclen and print it using get_vector

auto extractor = [](struct dirent& entry) {
  return entry.d_reclen;
};

for (const auto& reclen : scandirpp::get_vector(".", extractor /*, value_filter, entry_filter*/)) {
  std::cout << reclen << std::endl;
}
```

Note that the actual type of `d_reclen` does not appear anywhere in the above example: it is implicitly deduced in the lambda, and it is declared `auto` in the for-loop.

### Custom containers
Collecting results in container types other than `std::vector` is possible with `scandirpp::get_values`. The signature is almost identical to `get_vector`, except for an additional required argument (`result`):

```c++
template<class OutputIterator, class ValueExtractor, class ValueFilter, class EntryFilter>
void get_values(const std::string& dir,
                OutputIterator result,
                ValueExtractor value_extractor,
                ValueFilter value_filter,
                EntryFilter entry_filter)
```

`result` must support being assigned an object of the type returned by `value_extractor`. Note that default extractors and filters can be used here (`value_filter` and `entry_filter` both default to `scandirpp::default_filter`).

```c++
// Just like get_names, but place results in a set instead of a vector

std::set<std::string> names;

scandirpp::get_values(".", std::inserter(names, names.begin()), scandirpp::extract_name);
```

### Low-level access
The immediate wrapper around `scandir` is the `scandirpp::get_entry_ptrs` function:

```c++
std::vector<std::unique_ptr<struct dirent>> get_entry_ptrs(const std::string& dir)
```

Whenever `scandir` returns successfully, it `malloc`s enough memory to hold _n_ `struct dirent` pointers and returns _n_. There is no mechanism in `scandir` itself to free this memory, the user is expected to do so, as demostrated by the example code on the [man page][scandir]:

```c++
#include <dirent.h>

int
main(void)
{
   struct dirent **namelist;
   int n;

   n = scandir(".", &namelist, NULL, alphasort);
   if (n < 0)
       perror("scandir");
   else {
       while (n--) {
           printf("%s\n", namelist[n]->d_name);
           free(namelist[n]);
       }
       free(namelist);
   }
}
```

This is obviously only suitable if the code within the `while (n--) {}` block never throws, otherwise memory will be leaked. However, by constructing a `std::unique_ptr` from each pointer, memory safety is assured: whenever a `unique_ptr` is destroyed, it calls `delete` on the underlying raw pointer, which in turn ensures that the associated memory is freed.

Because the pointers are in consecutive memory addresses, and because their number (_n_) is known, they can be used in the range constructor of `std::vector`. Using the variable names from the snippet above:

```c++
struct dirent **namelist;
int n = scandir(".", &namelist, NULL, alphasort);
if (n < 0) {
   /* throw exception */
} else {
   std::vector<std::unique_ptr<struct dirent>> result(namelist, namelist+n);
}
```

This is effectively how `get_entry_ptrs` retrieves the results of scandir.

[scandir]: http://man7.org/linux/man-pages/man3/scandir.3.html
[dirent]: http://pubs.opengroup.org/onlinepubs/009695399/basedefs/dirent.h.html
