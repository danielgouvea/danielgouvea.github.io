---
layout: post
title:  "C++ Simple Binary Serialization"
date:   2020-03-22 17:00:45 -0400
categories: c++ serialization
---

Data serialization is a very present matter whether the objective is to transmit information over a network or simply save it any kind of persistent memory. While serialization to some text formats are very popular, like JSON and XML, binary serialization has its advantages, for example, smaller size without requiring compression and smaller processing overhead.

Implementing a simple binary serialization can be fairly easy in C++ because the language offers access to memory through pointers which represent addresses to the memory regions that holds the variables data as bytes.

## Data Types

Everything in C++ boils down to its [fundamental types](https://en.cppreference.com/w/cpp/language/types) with each one having a specific size in bytes:

```cpp
#include <iostream>

using std::cout;

int main() {
    cout << "Type              Size (bytes)\n";
    cout << "bool              " << sizeof(bool) << '\n';
    cout << "char              " << sizeof(char) << '\n';
    cout << "short int         " << sizeof(short int) << '\n';
    cout << "int               " << sizeof(int) << '\n';
    cout << "long int          " << sizeof(long int) << '\n';
    cout << "long long int     " << sizeof(long long int) << '\n';
    cout << "float             " << sizeof(float) << '\n';
    cout << "double            " << sizeof(double) << '\n';
    cout << "long double       " << sizeof(long double) << '\n';
    cout << "any pointer       " << sizeof(void *) << '\n';
}
```

```text
Type              Size (bytes)
bool              1
char              1
short int         2
int               4
long int          8
long long int     8
float             4
double            8
long double       16
any pointer       8
```

This means that a class is nothing more than a set of fundamental type data members, and the size required to represent it in bytes is just the sum of all of its data members, or is it? See the following example:

```cpp
#include <iostream>

using std::cout;

class Foo {
    bool a;   // 1 byte
    double b; // 8 bytes
    int c;    // 4 bytes
    char d;   // 1 byte
};

class Bar {
    Foo *fooPointer; // 8 bytes
    Foo foo;         // sizeof(Foo) bytes
};

int main() {
    cout << "Foo size in bytes: " << sizeof(Foo) << '\n';
    cout << "Bar size in bytes: " << sizeof(Bar) << '\n';
}
```

```text
Foo size in bytes: 24
Bar size in bytes: 32
```

Shouldn't `Foo` size be 14 bytes instead of 24 bytes? What is happening here is that the compiler is aligning the data to the memory word size, which in the case of a 64-bit architecture is 8 bytes, by padding it. Therefore, `Foo::a` which theoretically should only occupy 1 byte is using the last 1 byte of a 8 byte word, `Foo::b` is taking a full word, and `Foo::c` and `Foo::d` are sharing the last word, hence `Foo` is using 3 words or 24 bytes.

When serializing a class we may want to be more mindful about the space it requires, and for that we may want to pack the data to occupy only the space which is the exact sum of the size of each of its data members. In order to achieve that we can make use of the `__attribute__((packed))` compiler feature:

```cpp
class __attribute__((packed)) Foo {
    bool a;   // 1 byte
    double c; // 8 bytes
    int b;    // 4 bytes
    char d;   // 1 byte
};
```
```text
Foo size in bytes: 14
```

Now we have the size we were expecting for `Foo`, only 14 bytes.

### Representing Bytes

When we serialize an entity to bytes, we need a data structure to hold this array of bytes. As C++ lacks a byte type, we are going to use `char` type, which has 1 byte in size, but particularly its variant `unsigned char` because we are not interested in sign when we are talking about bytes. And then, to represent an array of bytes we are going to use `std::vector<unsigned char>`.

### A Note on Printing Bytes

As we are representing bytes as `unsigned char` type, it may be very annoying printing them out because a text output stream will try to render it as an ASCII character instead of a number, and the character correspondent to that number may not be a printable one. One simple way to solve this issue is by expressing to the compiler that the `byte` variable should treated as a number by casting it to one: `static_cast<unsigned>(byte)`. Below it is a straightforward example on how to format and print a byte array:

```cpp
#include <iomanip>
#include <iostream>
#include <vector>

using std::cout;
using std::dec;     // Print numbers in decimal base
using std::hex;     // Print numbers in hexadecimal base
using std::setw;    // Set width in digits to print numbers
using std::setfill; // Set fill character to complete width
using std::vector;

void printBytes(const vector<unsigned char> &bytes) {
    for (const auto byte : bytes) {
        cout << "0x" << hex << setw(2) << setfill('0')
             << static_cast<unsigned>(byte) << ' ';
    }
    cout << dec << '\n';
}
```

For example:

```cpp
printBytes({1, 10, 100, 200})
```

```text
0x01 0x0a 0x64 0xc8 
```

## Serializing Fundamental Types

There are two main ways of how to implement data serialization in C++: using memory copy function or casting pointers. We are going to see how each of these works.

### Memory Copy

Using [`std::memcpy`](https://en.cppreference.com/w/cpp/string/byte/memcpy) function, we can copy a desired amount of bytes directly from one memory location to another:

```cpp
#include <cstring>
#include <iostream>
#include <vector>

using std::cout;
using std::memcpy;
using std::vector;

int main() {
    const int value = 42;

    // Serialization
    vector<unsigned char> bytes(sizeof(value));
    memcpy(bytes.data(), &value, sizeof(value));
    printBytes(bytes);

    // Deserialization
    int deserializedValue;
    memcpy(&deserializedValue, bytes.data(), sizeof(deserializedValue));
    cout << deserializedValue << '\n';
}
```

```text
0x2a 0x00 0x00 0x00 
42
```

The serialization implementation is really very straightforward, we allocate a vector named `bytes` with the exact size to fit `value`, and then copy the 4 bytes of `value` to `bytes`. The deserialization is equally simple, copy the first 4 bytes of `bytes` to `deserializedValue`.

### Casting Pointers

Using [`reinterpret_cast<new_pointer_type>(pointer_type)`](https://en.cppreference.com/w/cpp/language/reinterpret_cast), we can leverage the pointer type system to do the work for us. When we specify the fundamental type of a pointer, we are specifying how many contiguous bytes in memory a pointer is pointing to at once.

```cpp
#include <iostream>
#include <vector>

using std::cout;
using std::vector;

int main() {
    const int value = 42;

    // Serialization
    vector<unsigned char> bytes(sizeof(value));
    *reinterpret_cast<int *>(bytes.data()) = value;
    printBytes(bytes);

    // Deserialization
    const int deserializedValue = *reinterpret_cast<int *>(bytes.data());
    cout << deserializedValue << '\n';
}
```

```text
0x2a 0x00 0x00 0x00 
42
```

In this case, `bytes.data()` has the type `unsigned char *`, which points to the first byte on the vector, but when we cast it to `int *` it will point to the first 4 bytes on the vector because that is the `sizeof(int)`. So when we dereference this pointer using the `*` operator, the left side of the assignment is the same as an `int` variable to which we can write `value` to.

### Endianness

An interesting question that arises after serializing `int value = 42` is why the most significant byte value is `0x2a`, not the least significant one? The explanation for that comes from [endianness](https://en.wikipedia.org/wiki/Endianness), which is the order in which bytes are lay down into memory. Most desktop computer architectures rely on little-endian representation where bytes are ordered from the least to the most significant one.

## Serializing Dynamic-sized Structures

While serializing fundamental types is quite straightforward, structures with dynamic size require a different approach in order to convey the information of what size its dynamic part currently has and to make it occupy only the size required to represent it. The following code is an simple example of how to serialize a vector of integer values, by first serializing the number of elements it currently has followed by the serialized elements themselves.

```cpp
vector<unsigned char> serialize(const vector<int> &values) {
    // Bytes must fit the size of values followed by each element of it
    vector<unsigned char> bytes(
        sizeof(values.size()) +
        values.size() * sizeof(int)
    );
    
    // Let us keep track of the writing position
    unsigned char *writePosition = bytes.data();

    // Size is serialized first as it will be needed first when deserializing
    *reinterpret_cast<decltype(values.size()) *>(writePosition) = values.size();
    writePosition += sizeof(values.size());

    // Copies the bytes of all elements at once
    memcpy(writePosition, values.data(), values.size() * sizeof(int));

    return bytes;
}
```

When deserializing, we need to extract the number of elements first so we can know how many elements must be pre-allocated on the dynamic structure and how many bytes must be read to fill in those elements.

```cpp
vector<int> deserialize(const vector<unsigned char> &bytes) {
    // Let us keep track of the reading position
    const unsigned char *readPosition = bytes.data();

    // Retrieve the size of the vector
    const auto size = 
        *reinterpret_cast<const vector<int>::size_type *>(readPosition);
    readPosition += sizeof(size);

    // Create vector and fill all of its elements at once
    vector<int> values(size);
    memcpy(values.data(), readPosition, size * sizeof(int));

    return values;
}
```

Then a simple working example would look like this:

```cpp
int main() {
    const vector<int> values = {42, 1234567890, -42};

    // Serialization
    auto bytes = serialize(values);
    printBytes(bytes);

    // Deserialization
    const vector<int> deserializedValues = deserialize(bytes);
    cout << "size: " << deserializedValues.size() << " values: ";
    for (auto deserializedValue : deserializedValues) {
        cout << deserializedValue << ' ';
    }
    cout << '\n';
}
```

```text
0x03 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x2a 0x00 0x00 0x00 0xd2 0x02 0x96 0x49 0xd6 0xff 0xff 0xff 
size: 3 values: 42 1234567890 -42
```

The first 8 bytes represent the number of elements in the vector, in this case 3 elements, then what follows is 3 sequences of 4 bytes, each representing the corresponding element of the vector: `42`, `1234567890` and `-42`.

### A Note on Error Checking

Our implementation of dynamic-sized structures deserialization reads bytes without checking first if there is enough bytes available on `bytes` vector to complete the operation. That is unsafe because these bytes may be corrupted or tempered, and an overflow can crash the program or open it to security issues. A very simple solution is to throw an exception in this case.

```cpp
if (readPosition + size > bytes.size()) {
    throw std::runtime_error("Not enough bytes available");
}
```

## Serializing Classes

Classes are just sets of data members, being them fundamental types, other classes or dynamic structures, then summing up all the technics presented above.

- allocate buffer to hold the exact size
- serialize class to buffer
- support serializing nested classes

```cpp
```

## Conclusion

- serializing pointers and references
- avoid memory allocations when knowing max structure size
