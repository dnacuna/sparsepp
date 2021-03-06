[![Build Status](https://travis-ci.org/greg7mdp/sparsepp.svg?branch=master)](https://travis-ci.org/greg7mdp/sparsepp)

# Sparsepp: A fast, memory efficient hash map for C++

Sparsepp is derived from Google's excellent [sparsehash](https://github.com/sparsehash/sparsehash) implementation. It aims to achieve the following objectives:

- A drop-in alternative for unordered_map and unordered_set.
- **Extremely low memory usage** (typically about one byte overhead per entry).
- **Very efficient**, typically faster than your compiler's unordered map/set or Boost's.
- **C++11 support** (if supported by compiler).
- **Single header** implementation - just copy `sparsepp.h` to your project and include it.
- **Tested** on Windows (vs2010-2015, g++), linux (g++, clang++) and MacOS (clang++).

We believe Sparsepp provides an unparalleled combination of performance and memory usage, and will outperform your compiler's unordered_map on both counts. Only Google's `dense_hash_map` is consistently faster, at the cost of much greater memory usage (especially when the final size of the map is not known in advance). 

For a detailed comparison of various hash implementations, including Sparsepp, please see our [write-up](bench.md).

## Example

```c++
#include <iostream>
#include <string>
#include <sparsepp.h>

using spp::sparse_hash_map;
 
int main()
{
    // Create an unordered_map of three strings (that map to strings)
    sparse_hash_map<std::string, std::string> email = 
    {
        { "tom",  "tom@gmail.com"},
        { "jeff", "jk@gmail.com"},
        { "jim",  "jimg@microsoft.com"}
    };
 
    // Iterate and print keys and values 
    for (const auto& n : email) 
        std::cout << n.first << "'s email is: " << n.second << "\n";
 
    // Add a new entry
    email["bill"] = "bg@whatever.com";
 
    // and print it
    std::cout << "bill's email is: " << email["bill"] << "\n";
 
    return 0;
}
```

## Installation

Since the full Sparsepp implementation is contained in a single header file `sparsepp.h`, the installation consist in copying this header file wherever it will be convenient to include in your project(s). 

Optionally, a second header file `spp_utils.h` is provided, which implements only the spp::hash_combine() functionality. This is useful when we want to specify a hash function for a user-defined class in an header file, without including the full `sparsepp.h` header (this is demonstrated in [example 2](#example-2---providing-a-hash-function-for-a-user-defined-class) below).

## Usage

As shown in the example above, you need to include the header file: `#include <sparsepp.h>`

This provides the implementation for the following classes:

```c++
namespace spp
{
    template <class Key, 
              class T,
              class HashFcn  = spp_hash<Key>,  
              class EqualKey = std::equal_to<Key>,
              class Alloc    = libc_allocator_with_realloc<std::pair<const Key, T>>>
    class sparse_hash_map;

    template <class Value,
              class HashFcn  = spp_hash<Value>,
              class EqualKey = std::equal_to<Value>,
              class Alloc    = libc_allocator_with_realloc<Value>>
    class sparse_hash_set;
};
```

These classes provide the same interface as std::unordered_map and std::unordered_set, with the following differences:

- Calls to erase() may invalidate iterators. However, conformant to the C++11 standard, the position and range erase functions return an iterator pointing to the position immediately following the last of the elements erased. This makes it easy to traverse a sparse hash table and delete elements matching a condition. For example to delete odd values:

```c++
        for (auto it = c.begin(); it != c.end(); )
        if (it->first % 2 == 1)
            it = c.erase(it);
        else
            ++it;
```

- Since items are not grouped into buckets, Bucket APIs have been adapted: `max_bucket_count` is equivalent to `max_size`, and `bucket_count` returns the sparsetable size, which is normally at least twice the number of items inserted into the hash_map.

## Example 2 - providing a hash function for a user-defined class

In order to use a sparse_hash_set or sparse_hash_map, a hash function should be provided. Even though a the hash function can be provided via the HashFcn template parameter, we recommend injecting a specialization of `std::hash` for the class into the "std" namespace. For example:

```c++
#include <iostream>
#include <functional>
#include <string>
#include "sparsepp.h"

using std::string;

struct Person 
{
    bool operator==(const Person &o) const 
    { return _first == o._first && _last == o._last; }

    string _first;
    string _last;
};

namespace std
{
    // inject specialization of std::hash for Person into namespace std
    // ----------------------------------------------------------------
    template<> 
    struct hash<Person>
    {
        std::size_t operator()(Person const &p) const
        {
            std::size_t seed = 0;
            spp::hash_combine(seed, p._first);
            spp::hash_combine(seed, p._last);
            return seed;
        }
    };
}
 
int main()
{
    // As we have defined a specialization of std::hash() for Person, 
    // we can now create sparse_hash_set or sparse_hash_map of Persons
    // ----------------------------------------------------------------
    spp::sparse_hash_set<Person> persons = { { "John", "Galt" }, 
                                             { "Jane", "Doe" } };
    for (auto& p: persons)
        std::cout << p._first << ' ' << p._last << '\n';
}
```

The `std::hash` specialization for `Person` combines the hash values for both first and last name using the convenient spp::hash_combine function, and returns the combined hash value. 

spp::hash_combine is provided by the header `sparsepp.h`. However, class definitions often appear in header files, and it is desirable to limit the size of headers included in such header files, so we provide the very small header `spp_utils.h` for that purpose:

```c++
#include <string>
#include "spp_utils.h"

using std::string;
 
struct Person 
{
    bool operator==(const Person &o) const 
    { 
        return _first == o._first && _last == o._last && _age == o._age; 
    }

    string _first;
    string _last;
    int    _age;
};

namespace std
{
    // inject specialization of std::hash for Person into namespace std
    // ----------------------------------------------------------------
    template<> 
    struct hash<Person>
    {
        std::size_t operator()(Person const &p) const
        {
            std::size_t seed = 0;
            spp::hash_combine(seed, p._first);
            spp::hash_combine(seed, p._last);
            spp::hash_combine(seed, p._age);
            return seed;
        }
    };
}
```

## Example 3 - serialization

sparse_hash_set and sparse_hash_map can easily be serialized/unserialized to a file or network connection.
This support is implemented in the following APIs:

```c++
    template <typename Serializer, typename OUTPUT>
    bool serialize(Serializer serializer, OUTPUT *stream);

    template <typename Serializer, typename INPUT>
    bool unserialize(Serializer serializer, INPUT *stream);
```

The following example demontrates how a simple sparse_hash_map can be written to a file, and then read back. The serializer we use read and writes to a file using the stdio APIs, but it would be equally simple to write a serialized using the stream APIS:

```c++
#include <cstdio>

#include "sparsepp.h"

using spp::sparse_hash_map;
using namespace std;

class FileSerializer 
{
public:
    // serialize basic types to FILE
    // -----------------------------
    template <class T>
    bool operator()(FILE *fp, const T& value) 
    {
        return fwrite((const void *)&value, sizeof(value), 1, fp) == 1;
    }

    template <class T>
    bool operator()(FILE *fp, T* value) 
    {
        return fread((void *)value, sizeof(*value), 1, fp) == 1;
    }

    // serialize std::string to FILE
    // -----------------------------
    bool operator()(FILE *fp, const string& value) 
    {
        const size_t size = value.size();
        return (*this)(fp, size) && fwrite(value.c_str(), size, 1, fp) == 1;
    }

    bool operator()(FILE *fp, string* value) 
    {
        size_t size;
        if (!(*this)(fp, &size)) 
            return false;
        char* buf = new char[size];
        if (fread(buf, size, 1, fp) != 1) 
        {
            delete [] buf;
            return false;
        }
        new (value) string(buf, (size_t)size);
        delete[] buf;
        return true;
    }

    // serialize std::pair<const A, B> to FILE - needed for maps
    // ---------------------------------------------------------
    template <class A, class B>
    bool operator()(FILE *fp, const std::pair<const A, B>& value)
    {
        return (*this)(fp, value.first) && (*this)(fp, value.second);
    }

    template <class A, class B>
    bool operator()(FILE *fp, std::pair<const A, B> *value) 
    {
        return (*this)(fp, (A *)&value->first) && (*this)(fp, &value->second);
    }
};

int main(int argc, char* argv[]) 
{
    sparse_hash_map<string, int> age{ { "John", 12 }, {"Jane", 13 }, { "Fred", 8 } };

    // serialize age hash_map to "ages.dmp" file
    FILE *out = fopen("ages.dmp", "wb");
    age.serialize(FileSerializer(), out);
    fclose(out);

    sparse_hash_map<string, int> age_read;

    // read from "ages.dmp" file into age_read hash_map 
    FILE *input = fopen("ages.dmp", "rb");
    age_read.unserialize(FileSerializer(), input);
    fclose(input);

    // print out contents of age_read to verify correct serialization
    for (auto& v : age_read)
        printf("age_read: %s -> %d\n", v.first.c_str(), v.second);
}
```



