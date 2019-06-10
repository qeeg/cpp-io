# A proposal to add std::byte-based IO to C++

Note: early draft.

## Motivation

C++ has text streams for a long time. However, there is no comfortable way to read and write binary data. One can argue that it is possible to [ab]use `char`-based text streams that provide unformatted IO but it has many drawbacks:

* The API still works in terms of `char` which means `reinterpret_cast` if you use `std::byte` in your code base.
* Streams operate in terms of `std::char_traits` which makes no sense when doing binary IO. In particular, `std::ios::pos_type` is a very painful type to work with but is required in many IO operations.
* Stream open mode doesn't make a lot of sense and you'd always want to make sure to force it to have `std::ios_base::binary`.
* Stream objects carry a lot of text formatting flags that are irrelevant when doing binary IO. This leads to wasted memory.
* By default, stream operations don't throw exceptions. This usually means some wrapper code to force exceptions.
* If you want to do IO in memory, you're stuck with string streams that operate using `std::basic_stream`. Most binary data is stored in `std::vector` which leads to awful performance due to unnecessary copies.
* There is no agreed standard for customization points for binary IO.

This proposal tries to fix all mentioned issues.

## Prior art

This proposal is based on [ftz Serialization](https://gitlab.com/ftz/serialization) library which was initially written in 2010 targeting C++98 and was gradually updated to C++20. In particular, the following problems were encountered:

* There was no portable way to determine the native endianness, especially since sizes of all fundamental types can be 1 and all fixed-width types are optional. This was fixed by `std::endian` in C++20.
* There was no easy way to convert integers from native representation to two's complement and vice versa. This was fixed by requiring all integers to be two's complement in C++20.
* There is no easy way to convert integers from native endianness to specific endianness and vice versa. There is an `std::byteswap` proposal but it doesn't solve the general case because C++ allows systems that are neither big nor little endian.
* There is no easy way to convert floating point number from native represenation to ISO/IEC/IEEE 60559 and vice versa. This makes makes portable serialization of floating point numbers very hard on non-IEC platforms.

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

## Future work

It is hopeful that this proposal will be used as a basis for a modern Unicode-aware text streams because text is still bytes under the hood. However, this requires solving a lot of other difficult problems. Once modern text streams are finished, it is hopeful that legacy text streams will be deprecated and eventually removed.

This proposal doesn't rule out more low-level library that exposes complex details of modern operating systems. However, the design of this library has been intentionally kept as simple as possible to be novice-friendly.

## Header `<io>` synopsis

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
	class stream_base;
	class input_stream;
	class output_stream;
	class stream;
	
	// IO concepts
	template <typename T>
	concept CustomlyReadable = see below;
	template <typename T>
	concept CustomlyWritable = see below;
	
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

## Class `format`

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

TODO

#### Constructor

	constexpr format(endian endianness = endian::native,
		floating_point_format float_format = floating_point_format::native,
		bom_handling bh = bom_handling::none);

*Ensures:* `endianness_ == endianness`, `float_format_ == float_format` and `bom_handling_ == bh`.

#### Member functions

	constexpr endian get_endianness() const noexcept;

*Returns:* `endianness_`.

	constexpr void set_endianness(endian new_endianness) noexcept;

*Ensures:* `endianness_ == new_endianness`.

	constexpr floating_point_format get_floating_point_format() const noexcept;

*Returns:* `float_format_`.

	constexpr void set_floating_point_format(floating_point_format new_format)
		noexcept;

*Ensures:* `float_format_ == new_format`.

	constexpr bom_handling get_bom_handling() const noexcept;

*Returns:* `bom_handling_`.

	constexpr void set_bom_handling(bom_handling new_handling) noexcept;

*Ensures:* `bom_handling_ == new_handling`.

## Error handling

	const error_category& category() noexcept;

*Returns:* A reference to an object of a type derived from class `error_category`. All calls to this function shall return references to the same object.

*Remarks:* The object’s `default_error_condition` and `equivalent` virtual functions shall behave as specified for the class `error_category`. The object’s `name` virtual function shall return a pointer to the string `"io"`.

	error_code make_error_code(io_errc e) noexcept;

*Returns:* `error_code(static_cast<int>(e), io::category())`.

	error_condition make_error_condition(io_errc e) noexcept;

*Returns:* `error_condition(static_cast<int>(e), io::category())`.

## Class `io_error`

	class io_error : public system_error
	{
	public:
		io_error(const string& message, error_code ec);
		io_error(const char* message, error_code ec);
	};

TODO

## Stream base classes

### Class `stream_base`

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
		virtual streamsize get_position() = 0;
		virtual void set_position(streamsize position) = 0;
		virtual void seek_position(streamoff offset, ios_base::seekdir direction)
			= 0;
	private:
		format format_; // exposition only
	};

TODO

#### Constructor and destructor

	constexpr stream_base(format f = {});

*Ensures:* `format_ == f`.

#### Format

	constexpr format get_format() const noexcept;

*Returns:* `format_`.

	constexpr void set_format(format f) noexcept;

*Ensures:* `format_ == f`.

#### Position

	virtual streamsize get_position() = 0;

*Returns:* Current position of the stream.

	virtual void set_position(streamsize position) = 0;

*Effects:* Sets the position of the stream to the given value.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if position is negative and the stream doesn't support that.
* `value_too_large` - if position is greater than the maximum size supported by the stream.

<!-- -->

	virtual void seek_position(streamoff offset, ios_base::seekdir direction) = 0;

*Effects:* TODO

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if resulting position is negative and the stream doesn't support that.
* `value_too_large` - if resulting position is greater than the maximum size supported by the stream.

### Class `input_stream`

	class input_stream : public virtual stream_base
	{
	public:
		// Constructor
		input_stream(format f = {});
		
		// Reading
		virtual void read(span<byte> buffer) = 0;
	};

TODO

#### Constructor

	input_stream(format f = {});

*Ensures:* `get_format() == f`.

#### Reading

	virtual void read(span<byte> buffer) = 0;

*Effects:* Reads `size(buffer)` bytes from the stream and advances the position by that amount.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `reached_end_of_file` - tried to read past the end of stream.
* `physical_error` - if physical I/O error has occured.

### Class `output_stream`

	class output_stream : public virtual stream_base
	{
	public:
		// Constructor
		output_stream(format f = {});
		
		// Writing
		virtual void write(span<const byte> buffer) = 0;
	};

TODO

#### Constructor

	output_stream(format f = {});

*Ensures:* `get_format() == f`.

#### Writing

	virtual void write(span<const byte> buffer) = 0;

*Effects:* Writes `size(buffer)` bytes to the stream and advances the position by that amount.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `file_too_large` - tried to write past the maximum size supported by the stream.
* `physical_error` - if physical I/O error has occured.

### Class `stream`

	class stream : public input_stream, public output_stream
	{
	public:
		// Constructor
		stream(format f = {});
	};

TODO

#### Constructor

	stream(format f = {});

*Ensures:* `get_format() == f`.

## IO concepts

### Concept `CustomlyReadable`

	template <typename T>
	concept CustomlyReadable =
		requires(T object, input_stream& stream)
		{
			object.read(stream);
		};

TODO

### Concept `CustomlyWritable`

	template <typename T>
	concept CustomlyWritable =
		requires(const T object, output_stream& stream)
		{
			object.write(stream);
		};

TODO

## Customization points
### `io::read`

The name `read` denotes a customization point object. The expression `io::read(s, E)` for some subexpression `s` of type `input_stream` and subexpression `E` with type `T` has the following effects:

* If `T` is `byte`, reads one byte from the stream and assigns it to `E`.
* If `T` is `Integral`, reads `sizeof(T)` bytes from the stream, performs conversion of bytes from stream endianness to native endianness and assigns the result to object representation of `E`.
* If `T` is `FloatingPoint`, reads `sizeof(T)` bytes from the stream and:
  * If stream floating point format is native, assigns the bytes to the object representation of `E`.
  * If stream floating point format is `iec559`, performs conversion of bytes treated as an ISO/IEC/IEEE 60559 floating point representation in stream endianness to native format and assigns the result to the object representation of `E`.
* If `T` is a span of bytes, reads `size(E)` bytes from the stream and assigns them to `E`.
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
* If `T` is `CustomlyReadable`, calls `E.read(s)`.

Example implementation:

	namespace customization_points
	{

	void read(input_stream& s, byte& var);
	void read(input_stream& s, Integral auto& var);
	template <typename T>
		requires is_floating_point_v<T>
	void read(input_stream& s, T& var);
	template <size_t Extent>
	void read(input_stream& s, span<byte, Extent> buffer);
	template <size_t Extent>
	void read(input_stream& s, span<char8_t, Extent> buffer);
	template <size_t Extent>
	void read(input_stream& s, span<char16_t, Extent> buffer);
	template <size_t Extent>
	void read(input_stream& s, span<char32_t, Extent> buffer);
	void read(input_stream& s, CustomlyReadable auto& var);
	
	struct read_customization_point
	{
		template <typename T>
		void operator()(input_stream& s, T& var);
	};
	
	}
	
	inline customization_points::read_customization_point read;

### `io::write`

The name `write` denotes a customization point object. The expression `io::write(s, E)` for some subexpression `s` of type `output_stream` and subexpression `E` with type `T` has the following effects:

* If `T` is `byte`, writes it to the stream.
* If `T` is `Integral` or an enumeration type, performs conversion of object representation of `E` from native endianness to stream endianness and writes the result to the stream.
* If `T` is `FloatingPoint` and:
  * If stream floating point format is native, writes the object representation of `E` to the stream.
  * If stream floating point format is `iec559`, performs conversion of object representation of `E` from native format to ISO/IEC/IEEE 60559 format in stream endianness and writes the result to the stream.
* If `T` is a span of bytes, writes `size(E)` bytes to the stream.
* If `T` is a span of `char8_t` and:
  * If stream BOM handling is `none`, for every element `C` in the given span performs `io::write(s, C)`.
  * If stream BOM handling is `read_write`, writes UTF-8 BOM to the stream and for every element `C` in the given span performs `io::write(s, C)`.
* If `T` is a span of `char16_t` and:
  * If stream BOM handling is `none`, for every element `C` in the given span performs `io::write(s, C)`.
  * If stream BOM handling is `read_write`, writes UTF-16 BOM in the stream endianness to the stream and for every element `C` in the given span performs `io::write(s, C)`.
* If `T` is a span of `char32_t` and:
  * If stream BOM handling is `none`, for every element `C` in the given span performs `io::write(s, C)`.
  * If stream BOM handling is `read_write`, writes UTF-32 BOM in the stream endianness to the stream and for every element `C` in the given span performs `io::write(s, C)`.
* If `T` is `CustomlyWritable`, calls `E.write(s)`.

Example implementation:

	namespace customization_points
	{

	void write(output_stream& s, byte var);
	template <typename T>
		requires Integral<T> || is_enum_v<T>
	void write(output_stream& s, T var);
	template <typename T>
		requires is_floating_point_v<T>
	void write(output_stream& s, T var);
	template <size_t Extent>
	void write(output_stream& s, span<const byte, Extent> buffer);
	template <size_t Extent>
	void write(output_stream& s, span<const char8_t, Extent> buffer);
	template <size_t Extent>
	void write(output_stream& s, span<const char16_t, Extent> buffer);
	template <size_t Extent>
	void write(output_stream& s, span<const char32_t, Extent> buffer);
	void write(output_stream& s, CustomlyWritable const auto& var);

	struct write_customization_point
	{
		template <typename T>
		void operator()(output_stream& s, const T& var);
	};

	}

	inline customization_points::write_customization_point write;

## Span streams

### Class `input_span_stream`

	class input_span_stream final : public input_stream
	{
	public:
		// Constructors
		input_span_stream(format f = {});
		input_span_stream(span<const byte> buffer, format f = {});
		
		// Position
		streamsize get_position() override;
		void set_position(streamsize position) override;
		void seek_position(streamoff offset, ios_base::seekdir direction) override;
		
		// Reading
		void read(span<byte> buffer) override;
		
		// Buffer management
		span<const byte> get_buffer() const noexcept;
		void set_buffer(span<const byte> new_buffer) noexcept;
	private:
		span<const byte> buffer_; // exposition only
		size_t position_; // exposition only
	};

TODO

#### Constructors

	input_span_stream(format f = {});

*Ensures:*
* `get_format() == f`,
* `empty(buffer_) == true`,
* `position_ == 0`.

<!-- -->

	input_span_stream(span<const byte> buffer, format f = {});

*Ensures:*
* `get_format() == f`,
* `data(buffer_) == data(buffer)`,
* `size(buffer_) == size(buffer)`,
* `position_ == 0`.

#### Position

	streamsize get_position() override;

*Returns:* `position_`.

	void set_position(streamsize position) override;

*Ensures:* `position_ == position`.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if position is negative.
* `value_too_large` - if position is greater than `numeric_limits<size_t>::max()`.

<!-- -->

	void seek_position(streamoff offset, ios_base::seekdir direction) override;

*Effects:* TODO

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if resulting position is negative.
* `value_too_large` - if resulting position is greater than `numeric_limits<size_t>::max()`.

#### Reading

	void read(span<byte> buffer) override;

*Effects:* Reads `size(buffer)` bytes from the stream and advances the position by that amount.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `reached_end_of_file` - if `(position_ + size(buffer)) > size(buffer_)`.

#### Buffer management

	span<const byte> get_buffer() const noexcept;

*Returns:* `buffer_`.

	void set_buffer(span<const byte> new_buffer) noexcept;

*Ensures:*
* `data(buffer_) == data(new_buffer)`,
* `size(buffer_) == size(new_buffer)`,
* `position_ == 0`.

### Class `output_span_stream`

	class output_span_stream final : public output_stream
	{
	public:
		// Constructors
		output_span_stream(format f = {});
		output_span_stream(span<byte> buffer, format f = {});
		
		// Position
		streamsize get_position() override;
		void set_position(streamsize position) override;
		void seek_position(streamoff offset, ios_base::seekdir direction) override;
		
		// Writing
		void write(span<const byte> buffer) override;
		
		// Buffer management
		span<byte> get_buffer() const noexcept;
		void set_buffer(span<byte> new_buffer) noexcept;
	private:
		span<byte> buffer_; // exposition only
		size_t position_; // exposition only
	};

TODO

#### Constructors

	output_span_stream(format f = {});

*Ensures:*
* `get_format() == f`,
* `empty(buffer_) == true`,
* `position_ == 0`.

<!-- -->

	output_span_stream(span<byte> buffer, format f = {});

*Ensures:*
* `get_format() == f`,
* `data(buffer_) == data(buffer)`,
* `size(buffer_) == size(buffer)`,
* `position_ == 0`.

#### Position

	streamsize get_position() override;

*Returns:* `position_`.

	void set_position(streamsize position) override;

*Ensures:* `position_ == position`.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if position is negative.
* `value_too_large` - if position is greater than `numeric_limits<size_t>::max()`.

<!-- -->

	void seek_position(streamoff offset, ios_base::seekdir direction) override;

*Effects:* TODO

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if resulting position is negative.
* `value_too_large` - if resulting position is greater than `numeric_limits<size_t>::max()`.

#### Writing

	void write(span<const byte> buffer) override;

*Effects:* Writes `size(buffer)` bytes to the stream and advances the position by that amount.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `file_too_large` - if `(position_ + size(buffer)) > size(buffer_)`.

#### Buffer management

	span<byte> get_buffer() const noexcept;

*Returns:* `buffer_`.

	void set_buffer(span<byte> new_buffer) noexcept;

*Ensures:*
* `data(buffer_) == data(new_buffer)`,
* `size(buffer_) == size(new_buffer)`,
* `position_ == 0`.

### Class `span_stream`

	class span_stream final : public stream
	{
	public:
		// Constructors
		span_stream(format f = {});
		span_stream(span<byte> buffer, format f = {});
		
		// Position
		streamsize get_position() override;
		void set_position(streamsize position) override;
		void seek_position(streamoff offset, ios_base::seekdir direction) override;
		
		// Reading
		void read(span<byte> buffer) override;
		
		// Writing
		void write(span<const byte> buffer) override;
		
		// Buffer management
		span<byte> get_buffer() const noexcept;
		void set_buffer(span<byte> new_buffer) noexcept;
	private:
		span<byte> buffer_; // exposition only
		size_t position_; // exposition only
	};

TODO

#### Constructors

	span_stream(format f = {});

*Ensures:*
* `get_format() == f`,
* `empty(buffer_) == true`,
* `position_ == 0`.

<!-- -->

	span_stream(span<byte> buffer, format f = {});

*Ensures:*
* `get_format() == f`,
* `data(buffer_) == data(buffer)`,
* `size(buffer_) == size(buffer)`,
* `position_ == 0`.

#### Position

	streamsize get_position() override;

*Returns:* `position_`.

	void set_position(streamsize position) override;

*Ensures:* `position_ == position`.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if position is negative.
* `value_too_large` - if position is greater than `numeric_limits<size_t>::max()`.

<!-- -->

	void seek_position(streamoff offset, ios_base::seekdir direction) override;

*Effects:* TODO

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if resulting position is negative.
* `value_too_large` - if resulting position is greater than `numeric_limits<size_t>::max()`.

#### Reading

	void read(span<byte> buffer) override;

*Effects:* Reads `size(buffer)` bytes from the stream and advances the position by that amount.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `reached_end_of_file` - if `(position_ + size(buffer)) > size(buffer_)`.

#### Writing

	void write(span<const byte> buffer) override;

*Effects:* Writes `size(buffer)` bytes to the stream and advances the position by that amount.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `file_too_large` - if `(position_ + size(buffer)) > size(buffer_)`.

#### Buffer management

	span<byte> get_buffer() const noexcept;

*Returns:* `buffer_`.

	void set_buffer(span<byte> new_buffer) noexcept;

*Ensures:*
* `data(buffer_) == data(new_buffer)`,
* `size(buffer_) == size(new_buffer)`,
* `position_ == 0`.

## Memory streams

### Class template `basic_input_memory_stream`

	template <typename Container>
	class basic_input_memory_stream final : public input_stream
	{
	public:
		// Constructors
		basic_input_memory_stream(format f = {});
		basic_input_memory_stream(const Container& c, format f = {});
		basic_input_memory_stream(Container&& c, format f = {});
		
		// Position
		streamsize get_position() override;
		void set_position(streamsize position) override;
		void seek_position(streamoff offset, ios_base::seekdir direction) override;
		
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
		typename Container::size_type position_; // exposition only
	};

TODO

#### Constructors

	basic_input_memory_stream(format f = {});

*Ensures:*
* `get_format() == f`,
* `buffer_ == Container{}`,
* `position_ == 0`.

<!-- -->

	basic_input_memory_stream(const Container& c, format f = {});

*Effects:* Initializes `buffer_` with `c`.

*Ensures:*
* `get_format() == f`,
* `position_ == 0`.

<!-- -->

	basic_input_memory_stream(Container&& c, format f = {});

*Effects:* Initializes `buffer_` with `move(c)`.

*Ensures:*
* `get_format() == f`,
* `position_ == 0`.

#### Position

	streamsize get_position() override;

*Returns:* `position_`.

	void set_position(streamsize position) override;

*Ensures:* `position_ == position`.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if position is negative.
* `value_too_large` - if position is greater than `numeric_limits<typename Container::size_type>::max()`.

<!-- -->

	void seek_position(streamoff offset, ios_base::seekdir direction) override;

*Effects:* TODO

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if resulting position is negative.
* `value_too_large` - if resulting position is greater than `numeric_limits<typename Container::size_type>::max()`.

#### Reading

	void read(span<byte> buffer) override;

*Effects:* Reads `size(buffer)` bytes from the stream and advances the position by that amount.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `reached_end_of_file` - if `(position_ + size(buffer)) > size(buffer_)`.

#### Buffer management

	const Container& get_buffer() const noexcept &;

*Returns:* `buffer_`.

	Container get_buffer() noexcept &&;

*Returns:* `move(buffer_)`.

	void set_buffer(const Container& new_buffer);

*Ensures:*
* `buffer_ == new_buffer`.
* `position_ == 0`.

<!-- -->

	void set_buffer(Container&& new_buffer);

*Effects:* Move assigns `new_buffer` to `buffer_`.

*Ensures:* `position_ == 0`.

	void reset_buffer() noexcept;

*Effects:* Equivalent to `buffer_.clear()`.

*Ensures:* `position_ == 0`.

### Class template `basic_output_memory_stream`

	template <typename Container>
	class basic_output_memory_stream final : public output_stream
	{
	public:
		// Constructors
		basic_output_memory_stream(format f = {});
		basic_output_memory_stream(const Container& c, format f = {});
		basic_output_memory_stream(Container&& c, format f = {});
		
		// Position
		streamsize get_position() override;
		void set_position(streamsize position) override;
		void seek_position(streamoff offset, ios_base::seekdir direction) override;
		
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
		typename Container::size_type position_; // exposition only
	};

TODO

#### Constructors

	basic_output_memory_stream(format f = {});

*Ensures:*
* `get_format() == f`,
* `buffer_ == Container{}`,
* `position_ == 0`.

<!-- -->

	basic_output_memory_stream(const Container& c, format f = {});

*Effects:* Initializes `buffer_` with `c`.

*Ensures:*
* `get_format() == f`,
* `position_ == 0`.

<!-- -->

	basic_output_memory_stream(Container&& c, format f = {});

*Effects:* Initializes `buffer_` with `move(c)`.

*Ensures:*
* `get_format() == f`,
* `position_ == 0`.

#### Position

	streamsize get_position() override;

*Returns:* `position_`.

	void set_position(streamsize position) override;

*Ensures:* `position_ == position`.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if position is negative.
* `value_too_large` - if position is greater than `numeric_limits<typename Container::size_type>::max()`.

<!-- -->

	void seek_position(streamoff offset, ios_base::seekdir direction) override;

*Effects:* TODO

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if resulting position is negative.
* `value_too_large` - if resulting position is greater than `numeric_limits<typename Container::size_type>::max()`.

#### Writing

	void write(span<const byte> buffer) override;

*Effects:* Writes `size(buffer)` bytes to the stream and advances the position by that amount.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `file_too_large` - if `(position_ + size(buffer)) > buffer_.max_size()`.

#### Buffer management

	const Container& get_buffer() const noexcept &;

*Returns:* `buffer_`.

	Container get_buffer() noexcept &&;

*Returns:* `move(buffer_)`.

	void set_buffer(const Container& new_buffer);

*Ensures:*
* `buffer_ == new_buffer`.
* `position_ == 0`.

<!-- -->

	void set_buffer(Container&& new_buffer);

*Effects:* Move assigns `new_buffer` to `buffer_`.

*Ensures:* `position_ == 0`.

	void reset_buffer() noexcept;

*Effects:* Equivalent to `buffer_.clear()`.

*Ensures:* `position_ == 0`.

### Class template `basic_memory_stream`

	template <typename Container>
	class basic_memory_stream final : public stream
	{
	public:
		// Constructors
		basic_memory_stream(format f = {});
		basic_memory_stream(const Container& c, format f = {});
		basic_memory_stream(Container&& c, format f = {});
		
		// Position
		streamsize get_position() override;
		void set_position(streamsize position) override;
		void seek_position(streamoff offset, ios_base::seekdir direction) override;
		
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
		typename Container::size_type position_; // exposition only
	};

TODO

#### Constructors

	basic_memory_stream(format f = {});

*Ensures:*
* `get_format() == f`,
* `buffer_ == Container{}`,
* `position_ == 0`.

<!-- -->

	basic_memory_stream(const Container& c, format f = {});

*Effects:* Initializes `buffer_` with `c`.

*Ensures:*
* `get_format() == f`,
* `position_ == 0`.

<!-- -->

	basic_memory_stream(Container&& c, format f = {});

*Effects:* Initializes `buffer_` with `move(c)`.

*Ensures:*
* `get_format() == f`,
* `position_ == 0`.

#### Position

	streamsize get_position() override;

*Returns:* `position_`.

	void set_position(streamsize position) override;

*Ensures:* `position_ == position`.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if position is negative.
* `value_too_large` - if position is greater than `numeric_limits<typename Container::size_type>::max()`.

<!-- -->

	void seek_position(streamoff offset, ios_base::seekdir direction) override;

*Effects:* TODO

*Throws:* `io_error` in case of error.

*Error conditions:*
* `invalid_argument` - if resulting position is negative.
* `value_too_large` - if resulting position is greater than `numeric_limits<typename Container::size_type>::max()`.

#### Reading

	void read(span<byte> buffer) override;

*Effects:* Reads `size(buffer)` bytes from the stream and advances the position by that amount.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `reached_end_of_file` - if `(position_ + size(buffer)) > size(buffer_)`.

#### Writing

	void write(span<const byte> buffer) override;

*Effects:* Writes `size(buffer)` bytes to the stream and advances the position by that amount.

*Throws:* `io_error` in case of error.

*Error conditions:*
* `file_too_large` - if `(position_ + size(buffer)) > buffer_.max_size()`.

#### Buffer management

	const Container& get_buffer() const noexcept &;

*Returns:* `buffer_`.

	Container get_buffer() noexcept &&;

*Returns:* `move(buffer_)`.

	void set_buffer(const Container& new_buffer);

*Ensures:*
* `buffer_ == new_buffer`.
* `position_ == 0`.

<!-- -->

	void set_buffer(Container&& new_buffer);

*Effects:* Move assigns `new_buffer` to `buffer_`.

*Ensures:* `position_ == 0`.

	void reset_buffer() noexcept;

*Effects:* Equivalent to `buffer_.clear()`.

*Ensures:* `position_ == 0`.

## File streams

### Class `input_file_stream`

	class input_file_stream final : public input_stream
	{
	public:
		// Construct/copy/destroy
		input_file_stream(const filesystem::path& file_name, format f = {});
		input_file_stream(const input_file_stream&) = delete;
		~input_file_stream();
		input_file_stream& operator=(const input_file_stream&) = delete;
		
		// Position
		streamsize get_position() override;
		void set_position(streamsize position) override;
		void seek_position(streamoff offset, ios_base::seekdir direction) override;
		
		// Reading
		void read(span<byte> buffer) override;
	};

TODO

### Class `output_file_stream`

	class output_file_stream final : public output_stream
	{
	public:
		// Construct/copy/destroy
		output_file_stream(const filesystem::path& file_name, format f = {});
		output_file_stream(const output_file_stream&) = delete;
		~output_file_stream();
		output_file_stream& operator=(const output_file_stream&) = delete;
		
		// Position
		streamsize get_position() override;
		void set_position(streamsize position) override;
		void seek_position(streamoff offset, ios_base::seekdir direction) override;
		
		// Writing
		void write(span<const byte> buffer) override;
	};

TODO

### Class `file_stream`

	class file_stream final : public stream
	{
	public:
		// Construct/copy/destroy
		file_stream(const filesystem::path& file_name, format f = {});
		file_stream(const file_stream&) = delete;
		~file_stream();
		file_stream& operator=(const file_stream&) = delete;
		
		// Position
		streamsize get_position() override;
		void set_position(streamsize position) override;
		void seek_position(streamoff offset, ios_base::seekdir direction) override;
		
		// Reading
		void read(span<byte> buffer) override;
		
		// Writing
		void write(span<const byte> buffer) override;
	};

TODO

## Open issues

* Concepts vs inheritance
* `format` as part of the stream class or as separate argument to `io::read` and `io::write`.
* `std::span` vs `std::ContiguousRange`
* Dependency on `std::ios_base`.
* Error handling
* `filesystem::path_view`
