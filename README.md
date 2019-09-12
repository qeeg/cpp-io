# A proposal to add std::byte-based IO to C++

Note: early draft.

## Motivation

C++ has text streams for a long time. However, there is no comfortable way to read and write binary data. One can argue that it is possible to [ab]use `char`-based text streams that provide unformatted IO but it has many drawbacks:

* The API still works in terms of `char` which means `reinterpret_cast` if you use `std::byte` in your code base.
* Streams operate in terms of `std::char_traits` which makes no sense when doing binary IO. In particular, `std::ios::pos_type` is a very painful type to work with but is required in many IO operations.
* Stream open mode doesn't make a lot of sense and you'd always want to make sure to force it to have `std::ios_base::binary`.
* Stream objects carry a lot of text formatting flags that are irrelevant when doing binary IO. This leads to wasted memory.
* By default, stream operations don't throw exceptions. This usually means some wrapper code to force exceptions.
* If you want to do IO in memory, you're stuck with string streams that operate using `std::string`. Most binary data is stored in `std::vector` which leads to awful performance due to unnecessary copies.
* There is no agreed standard for customization points for binary IO.

This proposal tries to fix all mentioned issues.

## Prior art

This proposal is based on [ftz Serialization](https://gitlab.com/ftz/serialization) library which was initially written in 2010 targeting C++98 and was gradually updated to C++20. In particular, the following problems were encountered:

* There was no portable way to determine the native endianness, especially since sizes of all fundamental types can be 1 and all fixed-width types are optional. This was fixed by `std::endian` in C++20.
* There was no easy way to convert integers from native representation to two's complement and vice versa. This was fixed by requiring all integers to be two's complement in C++20.
* There is no easy way to convert integers from native endianness to specific endianness and vice versa. There is an `std::byteswap` proposal but it doesn't solve the general case because C++ allows systems that are neither big nor little endian.
* There is no easy way to convert floating point number from native represenation to ISO/IEC/IEEE 60559 and vice versa. This makes makes portable serialization of floating point numbers very hard on non-IEC platforms. [P1468](https://wg21.link/p1468) should fix this.

While the author thinks that having endianness and floating point convertion functions available publicly is a good idea, they leave them as implementation details in this paper.

Thoughts on [Boost.Serialization](https://www.boost.org/doc/libs/1_69_0/libs/serialization/doc/index.html):

* It uses confusing operator overloading akin to standard text streams which leads to several problems such as unnecessary complexity of `>>` and `<<` returning a reference to the archive.
* It doesn't support portable serialization of floating point values.
* It tries to do too much by adding version number to customization points, performing magic on pointers, arrays, several standard containers and general purpose boost classes.
* Unfortunate macro to split `load` and `save` customization points.
* It still uses standard text streams as archives.

Thoughts on [Cereal](https://uscilab.github.io/cereal/index.html)

* It decided to inherit several Boost problems for the sake of compatibility.
* Strange `operator()` syntax for IO.
* Will not compile if `CHAR_BIT > 8`.
* Undefined behavior when detecting native endianness due to strict aliasing violation.
* Doesn't support portable serialization of floating point values, but gives helpful `static_assert` in case of non-IEC platform.
* Still uses standard text streams as archives.

## Design goals

* Always use `std::byte` instead of `char` when meaning raw bytes.
* Do not do any text processing or hold any text-related data inside stream classes, even as template parameters.
* Provide intuitive customization points.
* Support different endiannesses and floating point formats.
* Stream classes should efficiently map to OS API in case of file IO.

## Design decisions

* It was chosen to put all new types into separate namespace `std::io`. This follows the model ranges took where they define more modern versions of old facilities inside a new namespace.
* The general inheritance hierarchy of legacy text streams has more or less been preserved, hovewer, the classes have been renamed as follows:
  * `std::ios_base` and `std::basic_ios` -> `std::io::stream_base`.
  * `std::basic_istream` -> `std::io::input_stream`.
  * `std::basic_ostream` -> `std::io::output_stream`.
  * `std::basic_stream` -> `std::io::stream`.
  * `std::basic_istringstream` -> `std::io::basic_input_memory_stream`.
  * `std::basic_ostringstream` -> `std::io::basic_output_memory_stream`.
  * `std::basic_stringstream` -> `std::io::basic_memory_stream`.
  * `std::basic_ifstream` -> `std::io::input_file_stream`.
  * `std::basic_ofstream` -> `std::io::output_file_stream`.
  * `std::basic_fstream` -> `std::io::file_stream`.
* The `streambuf` part of legacy text streams has been dropped.
* Fixed size streams have been added:
  * `std::io::input_span_stream`.
  * `std::io::output_span_stream`.
  * `std::io::span_stream`.
* Since the explicit goal of this proposal is to do IO in terms of `std::byte`, `CharT` and `Traits` template parameters have been removed.
* All text formatting flags have been removed. A new class `std::io::format` has been introduced for binary format. The separate class is used in order to make the change of stream format atomic.
* Parts of legacy text streams related to `std::ios_base::iostate` have been removed. It is better to report any specific errors via exceptions and since binary files usually have fixed layout and always start chunks of data with size, any kind of IO error is usually unrecoverable.
* Since there is no more buffering because of lack of `streambuf` and operating systems only expose a single file position that is used both for reading and writing, the interface has been changed accordingly:
  * `tellg` and `tellp` -> `get_position`.
  * Single argument versions of `seekg` and `seekp` -> `set_position`.
  * Double argument versions of `seekg` and `seekp` -> `seek_position`.
* `std::basic_ios::pos_type` has been replaced with `std::streamoff`.
* `std::basic_ios::off_type` has been replaced with `std::streamoff`.
* `std::ios_base::seekdir` has been replaced with `std::io::seek_direction`.
* `gcount`, `get`, `getline`, `ignore`, `peek`, `readsome`, `putback`, `unget` and `put` member functions were removed because they don't make sense during binary IO.
* `read` member function takes `std::span<std::byte>`.
* `write` member function takes `std::span<const std::byte>`.
* `flush` member function was removed as there is no buffering.
* `operator>>` and `operator<<` have been replaced with `std::io::read` and `std::io::write` customization points.

## Tutorial

### Example 1: Writing integer with default format

```c++
#include <io>
#include <iostream>

int main()
{
	unsigned int value = 42;

	// Create a stream. This stream will write to dynamically allocated memory
	std::io::output_memory_stream s;

	// Write the value to the stream
	std::io::write(s, value);

	// Get reference to the buffer of the stream
	const auto& buffer = s.get_buffer();

	// Print the buffer
	for (auto byte : buffer)
	{
		std::cout << std::to_integer<int>(byte) << ' ';
	}
}
```

The result is implementation defined. For a random value it would depend on `CHAR_BIT`, `sizeof(unsigned int)` and `std::endian::native`. On AMD64 this will print:

```
42 0 0 0
```

This is because `CHAR_BIT` is 8, `sizeof(unsigned int)` is 4 and `std::endian::native == std::endian::little`.

We can be more strict and have more portable layout:

### Example 2: Writing integer with specific layout

```c++
#include <cstdint>
#include <io>
#include <iostream>

static_assert(CHAR_BIT == 8)

int main()
{
	std::uint32_t value = 42;

	// Create a specific binary format
	std::io::format f{std::endian::big};

	// Create a stream with our format
	std::io::output_memory_stream s{f};

	// Write the value to the stream
	std::io::write(s, value);

	// Get reference to the buffer of the stream
	const auto& buffer = s.get_buffer();

	// Print the buffer
	for (auto byte : buffer)
	{
		std::cout << std::to_integer<int>(byte) << ' ';
	}
}
```

This will either fail to compile on systems where `CHAR_BIT != 8` or print:

```
0 0 0 42
```

### Example 3: Working with user defined type

```c++
#include <io>

struct MyType
{
	int a;
	float b;

	void read(std::io::input_stream& stream)
	{
		std::io::read(stream, a);
		std::io::read(stream, b);
	}

	void write(std::io::output_stream& stream) const
	{
		std::io::write(stream, a);
		std::io::write(stream, b);
	}
};

int main()
{
	MyType t{1, 2.0f};
	std::io::output_memory_stream s;

	// std::io::write will automatically pickup "write" member function if it
	// has a valid signature.
	std::io::write(s, t);

	const auto& buffer = s.get_buffer();

	// Print the buffer
	for (auto byte : buffer)
	{
		std::cout << std::to_integer<int>(byte) << ' ';
	}
}
```

### Example 4: Working with 3rd party type

```c++
struct VendorType // Can't modify interface
{
	int a;
	float b;
};

// Add "read" and "write" as free functions. They will be picked up
// automatically.
void read(std::io::input_stream& stream, VendorType& vt)
{
	std::io::read(stream, vt.a);
	std::io::read(stream, vt.b);
}

void write(std::io::output_stream& stream, const VendorType& vt)
{
	std::io::write(stream, vt.a);
	std::io::write(stream, vt.b);
}

int main()
{
	VendorType vt{1, 2.0f};
	std::io::output_memory_stream s;

	// std::io::write will automatically pickup "write" non-member function if
	// it has a valid signature.
	std::io::write(s, t);

	const auto& buffer = s.get_buffer();

	// Print the buffer
	for (auto byte : buffer)
	{
		std::cout << std::to_integer<int>(byte) << ' ';
	}
}
```

### Example 5: Resource Interchange File Format

There are 2 flavors of RIFF files: little-endian and big-endian. Endianness is determined by the ID of the first chunk. ASCII "RIFF" means little-endian, ASCII "RIFX" means big-endian. We can just read the chunk ID as sequence of bytes, set the format of the stream to the correct endianness and read the rest of the file as usual.

```c++
#include <io>
#include <array>

// C++ doesn't have ASCII literals but we can use UTF-8 literals instead.
constexpr std::array<std::byte, 4> RIFFChunkID{
	std::byte{u8'R'}, std::byte{u8'I'}, std::byte{u8'F'}, std::byte{u8'F'}};
constexpr std::array<std::byte, 4> RIFXChunkID{
	std::byte{u8'R'}, std::byte{u8'I'}, std::byte{u8'F'}, std::byte{u8'X'}};
	
class RIFFFile
{
public:
	RIFFFile(std::io::input_stream& stream)
	{
		this->read(stream);
	}

	void read(std::io::input_stream& stream)
	{
		std::array<std::byte, 4> chunk_id;
		std::io::read(stream, chunk_id);
		if (chunk_id == RIFFChunkID)
		{
			// We have little endian file.
			// Set format to little endian.
			auto format = stream.get_format();
			format.set_endianness(std::endian::little);
			stream.set_format(format);
		}
		else if (chunk_id == RIFXChunkID)
		{
			// We have big endian file.
			// Set format to big endian.
			auto format = stream.get_format();
			format.set_endianness(std::endian::big);
			stream.set_format(format);
		}
		else
		{
			throw /* ... */
		}
		// We have set correct endianness based on the 1st chunk id.
		// The rest of the file will be deserialized correctly according to
		// our format.
		std::uint32_t chunk_size;
		std::io::read(stream, chunk_size);
		/* ... */
	}
private:
	/* ... */
}
```

TODO: More tutorials? More explanations.

## Implementation experience

Most of the proposal can be implemented in ISO C++. Low level conversions inside `std::io::read` and `std::io::write` require knowledge of implementation defined format of integers and floating point numbers. File IO requires calling operating system API. The following table provides some examples:

| Function        | POSIX   | Windows            | UEFI                            |
| --------------- | ------- | ------------------ | ------------------------------- |
| Constructor     | `open`  | `CreateFile`       | `EFI_FILE_PROTOCOL.Open`        |
| Destructor      | `close` | `CloseHandle`      | `EFI_FILE_PROTOCOL.Close`       |
| `get_position`  | `lseek` | `SetFilePointerEx` | `EFI_FILE_PROTOCOL.GetPosition` |
| `set_position`  | `lseek` | `SetFilePointerEx` | `EFI_FILE_PROTOCOL.SetPosition` |
| `seek_position` | `lseek` | `SetFilePointerEx` | No 1:1 mapping                  |
| `read`          | `read`  | `ReadFile`         | `EFI_FILE_PROTOCOL.Read`        |
| `write`         | `write` | `WriteFile`        | `EFI_FILE_PROTOCOL.Write`       |

## Future work

It is hopeful that `std::io::format` will be used to handle Unicode encoding schemes during file and network IO so Unicode layer will only need to handle encoding forms.

This proposal doesn't rule out more low-level library that exposes complex details of modern operating systems. However, the design of this library has been intentionally kept as simple as possible to be novice-friendly.

## Open issues

* Concepts vs inheritance
* `std::io::format` as part of the stream class or as separate argument to `std::io::read` and `std::io::write`.
* `std::span` vs `std::contiguous_range`
* Error handling
* `std::filesystem::path_view`
* Remove `std::io::floating_point_format` if P1468 is accepted.

## Wording

All text is relative to [n4830](https://wg21.link/n4830).

Move clauses 29.1 - 29.10 into a new clause 29.2 "Legacy text IO".

Add a new clause 29.1 "Binary IO".

### 29.1.? General [io.general]

TODO

### 29.1.? Header `<io>` synopsis [io.syn]

```c++
namespace std
{
namespace io
{

enum class floating_point_format
{
	iec559,
	native
};

enum class bom_handling
{
	none,
	read_write
};

class format;
	
enum class io_errc
{
	invalid_argument = implementation-defined,
	value_too_large = implementation-defined,
	reached_end_of_file = implementation-defined,
	physical_error = implementation-defined,
	file_too_large = implementation-defined
};

}
	
template <> struct is_error_code_enum<io::io_errc> : public true_type { };

namespace io
{

// Error handling
error_code make_error_code(io_errc e) noexcept;
error_condition make_error_condition(io_errc e) noexcept;

const error_category& category() noexcept;

class io_error;

// Stream base classes
enum class seek_direction
{
	beginning,
	current,
	end
};

class stream_base;
class input_stream;
class output_stream;
class stream;

// IO concepts
template <typename T>
concept customly_readable = see below;
template <typename T>
concept customly_writable = see below;

// Customization points
inline unspecified read;
inline unspecified write;

// Span streams
class input_span_stream;
class output_span_stream;
class span_stream;

// Memory streams
template <typename Container>
class basic_input_memory_stream;
template <typename Container>
class basic_output_memory_stream;
template <typename Container>
class basic_memory_stream;

using input_memory_stream = basic_input_memory_stream<vector<byte>>;
using output_memory_stream = basic_output_memory_stream<vector<byte>>;
using memory_stream = basic_memory_stream<vector<byte>>;

// File streams
class input_file_stream;
class output_file_stream;
class file_stream;

}
}
```

### 29.1.? Class `format` [io.format]

```c++
class format final
{
public:
	// Constructor
	constexpr format(endian endianness = endian::native,
		floating_point_format float_format = floating_point_format::native,
		bom_handling bh = bom_handling::none);

	// Member functions
	constexpr endian get_endianness() const noexcept;
	constexpr void set_endianness(endian new_endianness) noexcept;
	constexpr floating_point_format get_floating_point_format() const noexcept;
	constexpr void set_floating_point_format(floating_point_format new_format)
		noexcept;
	constexpr bom_handling get_bom_handling() const noexcept;
	constexpr void set_bom_handling(bom_handling new_handling) noexcept;
private:
	endian endianness_; // exposition only
	floating_point_format float_format_; // exposition only
	bom_handling bom_handling_; // exposition only
};
```

TODO

#### 29.1.?.? Constructor [io.format.cons]

```c++
constexpr format(endian endianness = endian::native,
	floating_point_format float_format = floating_point_format::native,
	bom_handling bh = bom_handling::none);
```

*Ensures:* `endianness_ == endianness`, `float_format_ == float_format` and `bom_handling_ == bh`.

#### 29.1.?.? Member functions [io.format.members]

```c++
constexpr endian get_endianness() const noexcept;
```

*Returns:* `endianness_`.

```c++
constexpr void set_endianness(endian new_endianness) noexcept;
```

*Ensures:* `endianness_ == new_endianness`.

```c++
constexpr floating_point_format get_floating_point_format() const noexcept;
```

*Returns:* `float_format_`.

```c++
constexpr void set_floating_point_format(floating_point_format new_format)
	noexcept;
```

*Ensures:* `float_format_ == new_format`.

```c++
constexpr bom_handling get_bom_handling() const noexcept;
```

*Returns:* `bom_handling_`.

```c++
constexpr void set_bom_handling(bom_handling new_handling) noexcept;
```

*Ensures:* `bom_handling_ == new_handling`.

### 29.1.? Error handling [io.errors]

```c++
const error_category& category() noexcept;
```

*Returns:* A reference to an object of a type derived from class `error_category`. All calls to this function shall return references to the same object.

*Remarks:* The object’s `default_error_condition` and `equivalent` virtual functions shall behave as specified for the class `error_category`. The object’s `name` virtual function shall return a pointer to the string `"io"`.

```c++
error_code make_error_code(io_errc e) noexcept;
```

*Returns:* `error_code(static_cast<int>(e), io::category())`.

```c++
error_condition make_error_condition(io_errc e) noexcept;
```

*Returns:* `error_condition(static_cast<int>(e), io::category())`.

### 29.1.? Class `io_error` [ioerr.ioerr]

```c++
class io_error : public system_error
{
public:
	io_error(const string& message, error_code ec);
	io_error(const char* message, error_code ec);
};
```

TODO

### 29.1.? Stream base classes [streams.base]

#### 29.1.?.? Class `stream_base` [stream.base]

```c++
class stream_base
{
public:
	// Constructor and destructor
	constexpr stream_base(format f = {});
	virtual ~stream_base() = default;

	// Format
	constexpr format get_format() const noexcept;
	constexpr void set_format(format f) noexcept;

	// Position
	virtual streamoff get_position() = 0;
	virtual void set_position(streamoff position) = 0;
	virtual void seek_position(streamoff offset, seek_direction direction) = 0;
private:
	format format_; // exposition only
};
```

TODO

##### 29.1.?.?.? Constructor and destructor [stream.base.cons]

```c++
constexpr stream_base(format f = {});
```

*Ensures:* `format_ == f`.

##### 29.1.?.?.? Format [stream.base.format]

```c++
constexpr format get_format() const noexcept;
```

*Returns:* `format_`.

```c++
constexpr void set_format(format f) noexcept;
```

*Ensures:* `format_ == f`.

##### 29.1.?.?.? Position [stream.base.position]

```c++
virtual streamoff get_position() = 0;
```

*Returns:* Current position of the stream.

```c++
virtual void set_position(streamoff position) = 0;
```

*Effects:* Sets the position of the stream to the given value.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if position is negative and the stream doesn't support that.
* `value_too_large` - if position is greater than the maximum size supported by the stream.

```c++
virtual void seek_position(streamoff offset, seek_direction direction) = 0;
```

*Effects:* TODO

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if resulting position is negative and the stream doesn't support that.
* `value_too_large` - if resulting position cannot be represented as type `streamoff` or is greater than the maximum size supported by the stream.

#### 29.1.?.? Class `input_stream` [input.stream]

```c++
class input_stream : public virtual stream_base
{
public:
	// Constructor
	input_stream(format f = {});

	// Reading
	virtual void read(span<byte> buffer) = 0;
};
```

TODO

##### 29.1.?.?.? Constructor [input.stream.cons]

```c++
input_stream(format f = {});
```

*Ensures:* `get_format() == f`.

##### 29.1.?.?.? Reading [input.stream.read]

```c++
virtual void read(span<byte> buffer) = 0;
```

*Effects:* Reads `ssize(buffer)` bytes from the stream and advances the position by that amount.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `value_too_large` - if resulting position cannot be represented as type `streamoff`.
* `reached_end_of_file` - tried to read past the end of stream.
* `physical_error` - if physical I/O error has occured.

#### 29.1.?.? Class `output_stream` [output.stream]

```c++
class output_stream : public virtual stream_base
{
public:
	// Constructor
	output_stream(format f = {});

	// Writing
	virtual void write(span<const byte> buffer) = 0;
};
```

TODO

##### 29.1.?.?.? Constructor [output.stream.cons]

```c++
output_stream(format f = {});
```

*Ensures:* `get_format() == f`.

##### 29.1.?.?.? Writing [output.stream.write]

```c++
virtual void write(span<const byte> buffer) = 0;
```

*Effects:* Writes `ssize(buffer)` bytes to the stream and advances the position by that amount.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `value_too_large` - if resulting position cannot be represented as type `streamoff`.
* `file_too_large` - tried to write past the maximum size supported by the stream.
* `physical_error` - if physical I/O error has occured.

#### 29.1.?.? Class `stream` [stream]

```c++
class stream : public input_stream, public output_stream
{
public:
	// Constructor
	stream(format f = {});
};
```

TODO

##### 29.1.?.?.? Constructor [stream.cons]

```c++
stream(format f = {});
```

*Ensures:* `get_format() == f`.

### 29.1.? IO concepts [io.concepts]

#### 29.1.?.? Concept `customly_readable` [io.concept.readable]

```c++
template <typename T>
concept customly_readable =
	requires(T object, input_stream& stream)
	{
		object.read(stream);
	};
```

TODO

#### 29.1.?.? Concept `customly_writable` [io.concept.writable]

```c++
template <typename T>
concept customly_writable =
	requires(const T object, output_stream& stream)
	{
		object.write(stream);
	};
```

TODO

### 29.1.? Customization points [???]
#### 29.1.?.1 `io::read` [io.read]

The name `read` denotes a customization point object. The expression `io::read(s, E)` for some subexpression `s` of type `input_stream` and subexpression `E` with type `T` has the following effects:

* If `T` is `byte`, reads one byte from the stream and assigns it to `E`.
* If `T` is `integral`, reads `sizeof(T)` bytes from the stream, performs conversion of bytes from stream endianness to native endianness and assigns the result to object representation of `E`.
* If `T` is `floating_point`, reads `sizeof(T)` bytes from the stream and:
  * If stream floating point format is native, assigns the bytes to the object representation of `E`.
  * If stream floating point format is `iec559`, performs conversion of bytes treated as an ISO/IEC/IEEE 60559 floating point representation in stream endianness to native format and assigns the result to the object representation of `E`.
* If `T` is a span of bytes, reads `ssize(E)` bytes from the stream and assigns them to `E`.
* If `T` is a span of `char8_t` and:
  * If stream BOM handling is `none`, for every element `C` in the given span performs `io::read(s, C)`.
  * If stream BOM handling is `read_write`, reads 3 bytes from the stream and:
    * If read bytes are not equal to UTF-8 BOM, the position is reverted to the one before the read and an exception is thrown.
    * Otherwise, for every element `C` in the given span performs `io::read(s, C)`.
* If `T` is a span of `char16_t`and:
  * If stream BOM handling is `none`, for every element `C` in the given span performs `io::read(s, C)`.
  * If stream BOM handling is `read_write`, reads 2 bytes from the stream and:
    * If read bytes are not equal to UTF-16 BOM, the position is reverted to the one before the read an exception is thrown.
    * If read bytes are equal to UTF-16 BOM:
      * Temporary sets the stream endianness to the endianness of read BOM.
      * For every element `C` in the given span performs `io::read(s, C)`.
      * Reverts stream endianness to the original value.
* If `T` is a span of `char32_t` and:
  * If stream BOM handling is `none`, for every element `C` in the given span performs `io::read(s, C)`.
  * If stream BOM handling is `read_write`, reads 4 bytes from the stream and:
    * If read bytes are not equal to UTF-32 BOM, the position is reverted to the one before the read an exception is thrown.
    * If read bytes are equal to UTF-32 BOM:
      * Temporary sets the stream endianness to the endianness of read BOM.
      * For every element `C` in the given span performs `io::read(s, C)`.
      * Reverts stream endianness to the original value.
* If `T` is `customly_readable`, calls `E.read(s)`.

Example implementation:

```c++
namespace customization_points
{

void read(input_stream& s, byte& var);
void read(input_stream& s, integral auto& var);
void read(input_stream& s, floating_point auto& var);
template <size_t Extent>
void read(input_stream& s, span<byte, Extent> buffer);
template <size_t Extent>
void read(input_stream& s, span<char8_t, Extent> buffer);
template <size_t Extent>
void read(input_stream& s, span<char16_t, Extent> buffer);
template <size_t Extent>
void read(input_stream& s, span<char32_t, Extent> buffer);
void read(input_stream& s, customly_readable auto& var);

struct read_customization_point
{
	template <typename T>
	void operator()(input_stream& s, T& var);
};

}

inline customization_points::read_customization_point read;
```

#### 29.1.?.2 `io::write` [io.write]

The name `write` denotes a customization point object. The expression `io::write(s, E)` for some subexpression `s` of type `output_stream` and subexpression `E` with type `T` has the following effects:

* If `T` is `byte`, writes it to the stream.
* If `T` is `integral` or an enumeration type, performs conversion of object representation of `E` from native endianness to stream endianness and writes the result to the stream.
* If `T` is `floating_point` and:
  * If stream floating point format is native, writes the object representation of `E` to the stream.
  * If stream floating point format is `iec559`, performs conversion of object representation of `E` from native format to ISO/IEC/IEEE 60559 format in stream endianness and writes the result to the stream.
* If `T` is a span of bytes, writes `ssize(E)` bytes to the stream.
* If `T` is a span of `char8_t` and:
  * If stream BOM handling is `none`, for every element `C` in the given span performs `io::write(s, C)`.
  * If stream BOM handling is `read_write`, writes UTF-8 BOM to the stream and for every element `C` in the given span performs `io::write(s, C)`.
* If `T` is a span of `char16_t` and:
  * If stream BOM handling is `none`, for every element `C` in the given span performs `io::write(s, C)`.
  * If stream BOM handling is `read_write`, writes UTF-16 BOM in the stream endianness to the stream and for every element `C` in the given span performs `io::write(s, C)`.
* If `T` is a span of `char32_t` and:
  * If stream BOM handling is `none`, for every element `C` in the given span performs `io::write(s, C)`.
  * If stream BOM handling is `read_write`, writes UTF-32 BOM in the stream endianness to the stream and for every element `C` in the given span performs `io::write(s, C)`.
* If `T` is `customly_writable`, calls `E.write(s)`.

Example implementation:

```c++
namespace customization_points
{

void write(output_stream& s, byte var);
template <typename T>
	requires integral<T> || is_enum_v<T>
void write(output_stream& s, T var);
void write(output_stream& s, floating_point auto var);
template <size_t Extent>
void write(output_stream& s, span<const byte, Extent> buffer);
template <size_t Extent>
void write(output_stream& s, span<const char8_t, Extent> buffer);
template <size_t Extent>
void write(output_stream& s, span<const char16_t, Extent> buffer);
template <size_t Extent>
void write(output_stream& s, span<const char32_t, Extent> buffer);
void write(output_stream& s, customly_writable const auto& var);

struct write_customization_point
{
	template <typename T>
	void operator()(output_stream& s, const T& var);
};

}

inline customization_points::write_customization_point write;
```

### 29.1.? Span streams [span.streams]

#### 29.1.?.1 Class `input_span_stream` [input.span.stream]

```c++
class input_span_stream final : public input_stream
{
public:
	// Constructors
	input_span_stream(format f = {});
	input_span_stream(span<const byte> buffer, format f = {});

	// Position
	streamoff get_position() override;
	void set_position(streamoff position) override;
	void seek_position(streamoff offset, seek_direction direction) override;

	// Reading
	void read(span<byte> buffer) override;

	// Buffer management
	span<const byte> get_buffer() const noexcept;
	void set_buffer(span<const byte> new_buffer) noexcept;
private:
	span<const byte> buffer_; // exposition only
	ptrdiff_t position_; // exposition only
};
```

TODO

##### 29.1.?.?.? Constructors [input.span.stream.cons]

```c++
input_span_stream(format f = {});
```

*Ensures:*
* `get_format() == f`,
* `empty(buffer_) == true`,
* `position_ == 0`.

```c++
input_span_stream(span<const byte> buffer, format f = {});
```

*Ensures:*
* `get_format() == f`,
* `data(buffer_) == data(buffer)`,
* `size(buffer_) == size(buffer)`,
* `position_ == 0`.

##### 29.1.?.?.? Position [input.span.stream.position]

```c++
streamoff get_position() override;
```

*Returns:* `position_`.

```c++
void set_position(streamoff position) override;
```

*Ensures:* `position_ == position`.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if position is negative.
* `value_too_large` - if position cannot be represented as type `ptrdiff_t`.

```c++
void seek_position(streamoff offset, seek_direction direction) override;
```

*Effects:* TODO

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if resulting position is negative.
* `value_too_large` - if resulting position cannot be represented as type `streamoff` or `ptrdiff_t`.

##### 29.1.?.?.? Reading [input.span.stream.read]

```c++
void read(span<byte> buffer) override;
```

*Effects:* Reads `ssize(buffer)` bytes from the stream and advances the position by that amount.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `value_too_large` - if `position_ + ssize(buffer)` cannot be represented as type `streamoff`.
* `reached_end_of_file` - if `(position_ + ssize(buffer)) > ssize(buffer_)`.

##### 29.1.?.?.? Buffer management [input.span.stream.buffer]

```c++
span<const byte> get_buffer() const noexcept;
```

*Returns:* `buffer_`.

```c++
void set_buffer(span<const byte> new_buffer) noexcept;
```

*Ensures:*
* `data(buffer_) == data(new_buffer)`,
* `size(buffer_) == size(new_buffer)`,
* `position_ == 0`.

#### 29.1.?.2 Class `output_span_stream` [output.span.stream]

```c++
class output_span_stream final : public output_stream
{
public:
	// Constructors
	output_span_stream(format f = {});
	output_span_stream(span<byte> buffer, format f = {});

	// Position
	streamoff get_position() override;
	void set_position(streamoff position) override;
	void seek_position(streamoff offset, seek_direction direction) override;

	// Writing
	void write(span<const byte> buffer) override;

	// Buffer management
	span<byte> get_buffer() const noexcept;
	void set_buffer(span<byte> new_buffer) noexcept;
private:
	span<byte> buffer_; // exposition only
	ptrdiff_t position_; // exposition only
};
```

TODO

##### 29.1.?.?.? Constructors [output.span.stream.cons]

```c++
output_span_stream(format f = {});
```

*Ensures:*
* `get_format() == f`,
* `empty(buffer_) == true`,
* `position_ == 0`.

```c++
output_span_stream(span<byte> buffer, format f = {});
```

*Ensures:*
* `get_format() == f`,
* `data(buffer_) == data(buffer)`,
* `size(buffer_) == size(buffer)`,
* `position_ == 0`.

##### 29.1.?.?.? Position [output.span.stream.position]

```c++
streamoff get_position() override;
```

*Returns:* `position_`.

```c++
void set_position(streamoff position) override;
```

*Ensures:* `position_ == position`.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if position is negative.
* `value_too_large` - if position cannot be represented as type `ptrdiff_t`.

```c++
void seek_position(streamoff offset, seek_direction direction) override;
```

*Effects:* TODO

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if resulting position is negative.
* `value_too_large` - if resulting position cannot be represented as type `streamoff` or `ptrdiff_t`.

##### 29.1.?.?.? Writing [output.span.stream.write]

```c++
void write(span<const byte> buffer) override;
```

*Effects:* Writes `ssize(buffer)` bytes to the stream and advances the position by that amount.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `value_too_large` - if `position_ + ssize(buffer)` cannot be represented as type `streamoff`.
* `file_too_large` - if `(position_ + ssize(buffer)) > ssize(buffer_)`.

##### 29.1.?.?.? Buffer management [output.span.stream.buffer]

```c++
span<byte> get_buffer() const noexcept;
```

*Returns:* `buffer_`.

```c++
void set_buffer(span<byte> new_buffer) noexcept;
```

*Ensures:*
* `data(buffer_) == data(new_buffer)`,
* `size(buffer_) == size(new_buffer)`,
* `position_ == 0`.

#### 29.1.?.3 Class `span_stream` [span.stream]

```c++
class span_stream final : public stream
{
public:
	// Constructors
	span_stream(format f = {});
	span_stream(span<byte> buffer, format f = {});

	// Position
	streamoff get_position() override;
	void set_position(streamoff position) override;
	void seek_position(streamoff offset, seek_direction direction) override;

	// Reading
	void read(span<byte> buffer) override;

	// Writing
	void write(span<const byte> buffer) override;

	// Buffer management
	span<byte> get_buffer() const noexcept;
	void set_buffer(span<byte> new_buffer) noexcept;
private:
	span<byte> buffer_; // exposition only
	ptrdiff_t position_; // exposition only
};
```

TODO

##### 29.1.?.?.? Constructors [span.stream.cons]

```c++
span_stream(format f = {});
```

*Ensures:*
* `get_format() == f`,
* `empty(buffer_) == true`,
* `position_ == 0`.

```c++
span_stream(span<byte> buffer, format f = {});
```

*Ensures:*
* `get_format() == f`,
* `data(buffer_) == data(buffer)`,
* `size(buffer_) == size(buffer)`,
* `position_ == 0`.

##### 29.1.?.?.? Position [output.span.stream.position]

```c++
streamoff get_position() override;
```

*Returns:* `position_`.

```c++
void set_position(streamoff position) override;
```

*Ensures:* `position_ == position`.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if position is negative.
* `value_too_large` - if position cannot be represented as type `ptrdiff_t`.

```c++
void seek_position(streamoff offset, seek_direction direction) override;
```

*Effects:* TODO

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if resulting position is negative.
* `value_too_large` - if resulting position cannot be represented as type `streamoff` or `ptrdiff_t`.

##### 29.1.?.?.? Reading [span.stream.read]

```c++
void read(span<byte> buffer) override;
```

*Effects:* Reads `ssize(buffer)` bytes from the stream and advances the position by that amount.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `value_too_large` - if `position_ + ssize(buffer)` cannot be represented as type `streamoff`.
* `reached_end_of_file` - if `(position_ + ssize(buffer)) > ssize(buffer_)`.

##### 29.1.?.?.? Writing [span.stream.write]

```c++
void write(span<const byte> buffer) override;
```

*Effects:* Writes `ssize(buffer)` bytes to the stream and advances the position by that amount.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `value_too_large` - if `position_ + ssize(buffer)` cannot be represented as type `streamoff`.
* `file_too_large` - if `(position_ + ssize(buffer)) > ssize(buffer_)`.

##### 29.1.?.?.? Buffer management [span.stream.buffer]

```c++
span<byte> get_buffer() const noexcept;
```

*Returns:* `buffer_`.

```c++
void set_buffer(span<byte> new_buffer) noexcept;
```

*Ensures:*
* `data(buffer_) == data(new_buffer)`,
* `size(buffer_) == size(new_buffer)`,
* `position_ == 0`.

### 29.1.? Memory streams [memory.streams]

#### 29.1.?.1 Class template `basic_input_memory_stream` [input.memory.stream]

```c++
template <typename Container>
class basic_input_memory_stream final : public input_stream
{
public:
	// Constructors
	basic_input_memory_stream(format f = {});
	basic_input_memory_stream(const Container& c, format f = {});
	basic_input_memory_stream(Container&& c, format f = {});

	// Position
	streamoff get_position() override;
	void set_position(streamoff position) override;
	void seek_position(streamoff offset, seek_direction direction) override;

	// Reading
	void read(span<byte> buffer) override;

	// Buffer management
	const Container& get_buffer() const noexcept &;
	Container get_buffer() noexcept &&;
	void set_buffer(const Container& new_buffer);
	void set_buffer(Container&& new_buffer);
	void reset_buffer() noexcept;
private:
	Container buffer_; // exposition only
	typename Container::difference_type position_; // exposition only
};
```

TODO

##### 29.1.?.?.? Constructors [input.memory.stream.cons]

```c++
basic_input_memory_stream(format f = {});
```

*Ensures:*
* `get_format() == f`,
* `buffer_ == Container{}`,
* `position_ == 0`.

```c++
basic_input_memory_stream(const Container& c, format f = {});
```

*Effects:* Initializes `buffer_` with `c`.

*Ensures:*
* `get_format() == f`,
* `position_ == 0`.

```c++
basic_input_memory_stream(Container&& c, format f = {});
```

*Effects:* Initializes `buffer_` with `move(c)`.

*Ensures:*
* `get_format() == f`,
* `position_ == 0`.

##### 29.1.?.?.? Position [input.memory.stream.position]

```c++
streamoff get_position() override;
```

*Returns:* `position_`.

```c++
void set_position(streamoff position) override;
```

*Ensures:* `position_ == position`.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if position is negative.
* `value_too_large` - if position if position cannot be represented as type `typename Container::difference_type`.

```c++
void seek_position(streamoff offset, seek_direction direction) override;
```

*Effects:* TODO

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if resulting position is negative.
* `value_too_large` - if resulting position cannot be represented as type `streamoff` or `typename Container::difference_type`.

##### 29.1.?.?.? Reading [input.memory.stream.read]

```c++
void read(span<byte> buffer) override;
```

*Effects:* Reads `ssize(buffer)` bytes from the stream and advances the position by that amount.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `value_too_large` - if `position_ + ssize(buffer)` cannot be represented as type `streamoff`.
* `reached_end_of_file` - if `(position_ + ssize(buffer)) > ssize(buffer_)`.

##### 29.1.?.?.? Buffer management [input.memory.stream.buffer]

```c++
const Container& get_buffer() const noexcept &;
```

*Returns:* `buffer_`.

```c++
Container get_buffer() noexcept &&;
```

*Returns:* `move(buffer_)`.

```c++
void set_buffer(const Container& new_buffer);
```

*Ensures:*
* `buffer_ == new_buffer`.
* `position_ == 0`.

```c++
void set_buffer(Container&& new_buffer);
```

*Effects:* Move assigns `new_buffer` to `buffer_`.

*Ensures:* `position_ == 0`.

```c++
void reset_buffer() noexcept;
```

*Effects:* Equivalent to `buffer_.clear()`.

*Ensures:* `position_ == 0`.

#### 29.1.?.2 Class template `basic_output_memory_stream` [output.memory.stream]

```c++
template <typename Container>
class basic_output_memory_stream final : public output_stream
{
public:
	// Constructors
	basic_output_memory_stream(format f = {});
	basic_output_memory_stream(const Container& c, format f = {});
	basic_output_memory_stream(Container&& c, format f = {});

	// Position
	streamoff get_position() override;
	void set_position(streamoff position) override;
	void seek_position(streamoff offset, seek_direction direction) override;

	// Writing
	void write(span<const byte> buffer) override;

	// Buffer management
	const Container& get_buffer() const noexcept &;
	Container get_buffer() noexcept &&;
	void set_buffer(const Container& new_buffer);
	void set_buffer(Container&& new_buffer);
	void reset_buffer() noexcept;
private:
	Container buffer_; // exposition only
	typename Container::difference_type position_; // exposition only
};
```

TODO

##### 29.1.?.?.? Constructors [output.memory.stream.cons]

```c++
basic_output_memory_stream(format f = {});
```

*Ensures:*
* `get_format() == f`,
* `buffer_ == Container{}`,
* `position_ == 0`.

```c++
basic_output_memory_stream(const Container& c, format f = {});
```

*Effects:* Initializes `buffer_` with `c`.

*Ensures:*
* `get_format() == f`,
* `position_ == 0`.

```c++
basic_output_memory_stream(Container&& c, format f = {});
```

*Effects:* Initializes `buffer_` with `move(c)`.

*Ensures:*
* `get_format() == f`,
* `position_ == 0`.

##### 29.1.?.?.? Position [output.memory.stream.position]

```c++
streamoff get_position() override;
```

*Returns:* `position_`.

```c++
void set_position(streamoff position) override;
```

*Ensures:* `position_ == position`.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if position is negative.
* `value_too_large` - if position if position cannot be represented as type `typename Container::difference_type`.

```c++
void seek_position(streamoff offset, seek_direction direction) override;
```

*Effects:* TODO

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if resulting position is negative.
* `value_too_large` - if resulting position cannot be represented as type `streamoff` or `typename Container::difference_type`.

##### 29.1.?.?.? Writing [output.memory.stream.write]

```c++
void write(span<const byte> buffer) override;
```

*Effects:* Writes `ssize(buffer)` bytes to the stream and advances the position by that amount.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `value_too_large` - if `position_ + ssize(buffer)` cannot be represented as type `streamoff`.
* `file_too_large` - if `(position_ + ssize(buffer)) > buffer_.max_size()`.

##### 29.1.?.?.? Buffer management [output.memory.stream.buffer]

```c++
const Container& get_buffer() const noexcept &;
```

*Returns:* `buffer_`.

```c++
Container get_buffer() noexcept &&;
```

*Returns:* `move(buffer_)`.

```c++
void set_buffer(const Container& new_buffer);
```

*Ensures:*
* `buffer_ == new_buffer`.
* `position_ == 0`.

```c++
void set_buffer(Container&& new_buffer);
```

*Effects:* Move assigns `new_buffer` to `buffer_`.

*Ensures:* `position_ == 0`.

```c++
void reset_buffer() noexcept;
```

*Effects:* Equivalent to `buffer_.clear()`.

*Ensures:* `position_ == 0`.

#### 29.1.?.3 Class template `basic_memory_stream` [memory.stream]

```c++
template <typename Container>
class basic_memory_stream final : public stream
{
public:
	// Constructors
	basic_memory_stream(format f = {});
	basic_memory_stream(const Container& c, format f = {});
	basic_memory_stream(Container&& c, format f = {});

	// Position
	streamoff get_position() override;
	void set_position(streamoff position) override;
	void seek_position(streamoff offset, seek_direction direction) override;

	// Reading
	void read(span<byte> buffer) override;

	// Writing
	void write(span<const byte> buffer) override;

	// Buffer management
	const Container& get_buffer() const noexcept &;
	Container get_buffer() noexcept &&;
	void set_buffer(const Container& new_buffer);
	void set_buffer(Container&& new_buffer);
	void reset_buffer() noexcept;
private:
	Container buffer_; // exposition only
	typename Container::difference_type position_; // exposition only
};
```

TODO

##### 29.1.?.?.? Constructors [memory.stream.cons]

```c++
basic_memory_stream(format f = {});
```

*Ensures:*
* `get_format() == f`,
* `buffer_ == Container{}`,
* `position_ == 0`.

```c++
basic_memory_stream(const Container& c, format f = {});
```

*Effects:* Initializes `buffer_` with `c`.

*Ensures:*
* `get_format() == f`,
* `position_ == 0`.

```c++
basic_memory_stream(Container&& c, format f = {});
```

*Effects:* Initializes `buffer_` with `move(c)`.

*Ensures:*
* `get_format() == f`,
* `position_ == 0`.

##### 29.1.?.?.? Position [memory.stream.position]

```c++
streamoff get_position() override;
```

*Returns:* `position_`.

```c++
void set_position(streamoff position) override;
```

*Ensures:* `position_ == position`.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if position is negative.
* `value_too_large` - if position if position cannot be represented as type `typename Container::difference_type`.

```c++
void seek_position(streamoff offset, seek_direction direction) override;
```

*Effects:* TODO

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if resulting position is negative.
* `value_too_large` - if resulting position cannot be represented as type `streamoff` or `typename Container::difference_type`.

##### 29.1.?.?.? Reading [memory.stream.read]

```c++
void read(span<byte> buffer) override;
```

*Effects:* Reads `ssize(buffer)` bytes from the stream and advances the position by that amount.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `value_too_large` - if `position_ + ssize(buffer)` cannot be represented as type `streamoff`.
* `reached_end_of_file` - if `(position_ + ssize(buffer)) > ssize(buffer_)`.

##### 29.1.?.?.? Writing [memory.stream.write]

```c++
void write(span<const byte> buffer) override;
```

*Effects:* Writes `ssize(buffer)` bytes to the stream and advances the position by that amount.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `value_too_large` - if `position_ + ssize(buffer)` cannot be represented as type `streamoff`.
* `file_too_large` - if `(position_ + ssize(buffer)) > buffer_.max_size()`.

##### 29.1.?.?.? Buffer management [memory.stream.buffer]

```c++
const Container& get_buffer() const noexcept &;
```

*Returns:* `buffer_`.

```c++
Container get_buffer() noexcept &&;
```

*Returns:* `move(buffer_)`.

```c++
void set_buffer(const Container& new_buffer);
```

*Ensures:*
* `buffer_ == new_buffer`.
* `position_ == 0`.

```c++
void set_buffer(Container&& new_buffer);
```

*Effects:* Move assigns `new_buffer` to `buffer_`.

*Ensures:* `position_ == 0`.

	void reset_buffer() noexcept;

*Effects:* Equivalent to `buffer_.clear()`.

*Ensures:* `position_ == 0`.

### 29.1.? File streams [file.streams???] (naming conflict)

#### 29.1.?.1 Class `input_file_stream` [input.file.stream]

```c++
class input_file_stream final : public input_stream
{
public:
	// Construct/copy/destroy
	input_file_stream(const filesystem::path& file_name, format f = {});
	input_file_stream(const input_file_stream&) = delete;
	input_file_stream(input_file_stream&&) = delete;
	~input_file_stream();
	input_file_stream& operator=(const input_file_stream&) = delete;
	input_file_stream& operator=(input_file_stream&&) = delete;

	// Position
	streamoff get_position() override;
	void set_position(streamoff position) override;
	void seek_position(streamoff offset, seek_direction direction) override;

	// Reading
	void read(span<byte> buffer) override;
};
```

TODO

#### 29.1.?.2 Class `output_file_stream` [output.file.stream]

```c++
class output_file_stream final : public output_stream
{
public:
	// Construct/copy/destroy
	output_file_stream(const filesystem::path& file_name, format f = {});
	output_file_stream(const output_file_stream&) = delete;
	output_file_stream(output_file_stream&&) = delete;
	~output_file_stream();
	output_file_stream& operator=(const output_file_stream&) = delete;
	output_file_stream& operator=(output_file_stream&&) = delete;

	// Position
	streamoff get_position() override;
	void set_position(streamoff position) override;
	void seek_position(streamoff offset, seek_direction direction) override;

	// Writing
	void write(span<const byte> buffer) override;
};
```

TODO

#### 29.1.?.3 Class `file_stream` [file.stream]

```c++
class file_stream final : public stream
{
public:
	// Construct/copy/destroy
	file_stream(const filesystem::path& file_name, format f = {});
	file_stream(const file_stream&) = delete;
	file_stream(file_stream&&) = delete;
	~file_stream();
	file_stream& operator=(const file_stream&) = delete;
	file_stream& operator=(file_stream&&) = delete;

	// Position
	streamoff get_position() override;
	void set_position(streamoff position) override;
	void seek_position(streamoff offset, seek_direction direction) override;

	// Reading
	void read(span<byte> buffer) override;

	// Writing
	void write(span<const byte> buffer) override;
};
```

TODO
