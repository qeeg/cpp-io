# A proposal to add std::byte-based IO to the C++

Note: early draft.

## Header `<io>` synopsis

	namespace std::io
	{
	
	enum class floating_point_format;
	enum class bom_handling;

	class format;
	class stream_base;
	class input_stream;
	class output_stream;
	class stream;
	
	template <typename T>
	concept CustomlyReadable = see below;
	template <typename T>
	concept CustomlyWritable = see below;
	
	inline unspecified read;
	inline unspecified write;
	
	class input_span_stream;
	class output_span_stream;
	class span_stream;
	
	template <Container = vector<byte>>
	class input_memory_stream;
	template <Container = vector<byte>>
	class output_memory_stream;
	template <Container = vector<byte>>
	class memory_stream;
	
	class input_file_stream;
	class output_file_stream;
	class file_stream;
	
	}

## Type `floating_point_format`

	enum class floating_point_format
	{
		iec559,
		native
	};

TODO

## Type `bom_handling`

	enum class bom_handling
	{
		none,
		read_write
	};

TODO

## Class `format`

	class format final
	{
	public:
		constexpr format(endian endianness = endian::native,
			floating_point_format float_format = floating_point_format::native,
			bom_handling bh = bom_handling::none);
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

	constexpr format(endian endianness = endian::native,
		floating_point_format float_format = floating_point_format::native,
		bom_handling bh = bom_handling::none);

*Ensures:* `endianness_ == endianness`, `float_format_ == float_format` and `bom_handling_ == bh`.

	constexpr endian get_endianness() const noexcept;

*Returns:* `enidianness_`.

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

## Class `stream_base`

	class stream_base
	{
	public:
		stream_base(format f = {});
		virtual ~stream_base() = default;
		format& get_format() noexcept;
		const format& get_format() const noexcept;
		virtual bool is_good() const = 0;
		virtual bool is_eof() const = 0;
		virtual bool is_fail() const = 0;
		virtual bool is_bad() const = 0;
		explicit operator bool() const;
		virtual ios_base::iostate get_state() const = 0;
		virtual void add_to_state(ios_base::iostate state) = 0;
		virtual void set_state(ios_base::iostate state) = 0;
		void clear_state();
	private:
		format format_; // exposition only
	};

TODO

	stream_base(format f = {});

*Ensures:* `format_ == f`.

	format& get_format() noexcept;
	const format& get_format() const noexcept;

*Returns:* `format_`.

## Class `input_stream`

	class input_stream : public virtual stream_base
	{
	public:
		input_stream(format f = {});
		virtual streamsize get_read_position() = 0;
		virtual void set_read_position(streamsize position) = 0;
		virtual void seek_read_position(streamoff offset,
			ios_base::seekdir direction) = 0;
		virtual void read(span<byte> buffer) = 0;
	};

TODO

	virtual streamsize get_read_position() = 0;

*Returns:* Current read position of the stream.

	virtual void set_read_position(streamsize position) = 0;

*Effects:* Sets the read position of the stream to the given value.

*Throws:* TODO

	virtual void seek_read_position(streamoff offset, ios_base::seekdir direction)
		= 0;

*Effects:* TODO

*Throws:* TODO

	virtual void read(span<byte> buffer) = 0;

*Effects:* Reads `size(buffer)` bytes from the stream and advances the read position by that amount.

*Throws:* TODO


## Class `output_stream`

	class output_stream : public virtual stream_base
	{
	public:
		output_stream(format f = {});
		virtual streamsize get_write_position() = 0;
		virtual void set_write_position(streamsize position) = 0;
		virtual void seek_write_position(streamoff offset,
			ios_base::seekdir direction) = 0;
		virtual void write(span<const byte> buffer) = 0;
	};

TODO

	virtual streamsize get_write_position() = 0;

*Returns:* Current write position of the stream.

	virtual void set_write_position(streamsize position) = 0;

*Effects:* Sets the write position of the stream to the given value.

*Throws:* TODO

	virtual void seek_write_position(streamoff offset, ios_base::seekdir direction)
		= 0;

*Effects:* TODO

*Throws:* TODO

	virtual void write(span<const byte> buffer) = 0;

*Effects:* Writes `size(buffer)` bytes to the stream and advances the write position by that amount.

*Throws:* TODO

## Class `stream`

	class stream : public input_stream, public output_stream
	{
	public:
		stream(format f = {});
	};

## Concept `CustomlyReadable`

	template <typename T>
	concept CustomlyReadable =
		requires(T object, input_stream& stream)
		{
			object.read(stream);
		};

TODO

## Concept `CustomlyWritable`

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
    * If read bytes are not equal to UTF-8 BOM, the read position is reverted to the one before the read and an exception is thrown.
    * Otherwise, for every element `C` in the given span performs `io::read(s, C)`.
* If `T` is a span of `char16_t`and:
  * If stream BOM handling is `none`, for every element `C` in the given span performs `io::read(s, C)`.
  * If stream BOM handling is `read_write`, reads 2 bytes from the stream and:
    * If read bytes are not equal to UTF-16 BOM, the read position is reverted to the one before the read an exception is thrown.
    * If read bytes are equal to UTF-16 BOM:
      * Temporary sets the stream endianness to the endianness of read BOM.
      * For every element `C` in the given span performs `io::read(s, C)`.
      * Reverts stream endianness to the original value.
* If `T` is a span of `char32_t` and:
  * If stream BOM handling is `none`, for every element `C` in the given span performs `io::read(s, C)`.
  * If stream BOM handling is `read_write`, reads 4 bytes from the stream and:
    * If read bytes are not equal to UTF-32 BOM, the read position is reverted to the one before the read an exception is thrown.
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
* If `T` is `Integral`, performs conversion of object representation of `E` from native endianness to stream endianness and writes the result to the stream.
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

## Class `input_span_stream`

	class input_span_stream final : public input_stream
	{
	public:
		input_span_stream(format f = {});
		input_span_stream(span<const byte> buffer, format f = {});
		bool is_good() const override;
		bool is_eof() const override;
		bool is_fail() const override;
		bool is_bad() const override;
		ios_base::iostate get_state() const override;
		void add_to_state(ios_base::iostate state) override;
		void set_state(ios_base::iostate state) override;
		streamsize get_read_position() override;
		void set_read_position(streamsize position) override;
		void seek_read_position(streamoff offset, ios_base::seekdir direction)
			override;
		void read(span<byte> buffer) override;
		span<const byte> get_buffer() const noexcept;
		void set_buffer(span<const byte> new_buffer);
	}

TODO

## Class `output_span_stream`

	class output_span_stream final : public output_stream
	{
	public:
		output_span_stream(format f = {});
		output_span_stream(span<byte> buffer, format f = {});
		bool is_good() const override;
		bool is_eof() const override;
		bool is_fail() const override;
		bool is_bad() const override;
		ios_base::iostate get_state() const override;
		void add_to_state(ios_base::iostate state) override;
		void set_state(ios_base::iostate state) override;
		streamsize get_write_position() override;
		void set_write_position(streamsize position) override;
		void seek_write_position(streamoff offset, ios_base::seekdir direction)
			override;
		void write(span<const byte> buffer) override;
		span<byte> get_buffer() const noexcept;
		void set_buffer(span<byte> new_buffer);
	}

TODO

## Class `span_stream`

	class span_stream final : public stream
	{
	public:
		span_stream(format f = {});
		span_stream(span<byte> buffer, format f = {});
		bool is_good() const override;
		bool is_eof() const override;
		bool is_fail() const override;
		bool is_bad() const override;
		ios_base::iostate get_state() const override;
		void add_to_state(ios_base::iostate state) override;
		void set_state(ios_base::iostate state) override;
		streamsize get_read_position() override;
		void set_read_position(streamsize position) override;
		void seek_read_position(streamoff offset, ios_base::seekdir direction)
			override;
		streamsize get_write_position() override;
		void set_write_position(streamsize position) override;
		void seek_write_position(streamoff offset, ios_base::seekdir direction)
			override;
		void read(span<byte> buffer) override;
		void write(span<const byte> buffer) override;
		span<byte> get_buffer() const noexcept;
		void set_buffer(span<byte> new_buffer);
	}

TODO

## Class template `input_memory_stream`

	template <Container = vector<byte>>
	class input_memory_stream final : public input_stream
	{
	public:
		input_memory_stream(format f = {});
		input_memory_stream(Container c, format f = {});
		bool is_good() const override;
		bool is_eof() const override;
		bool is_fail() const override;
		bool is_bad() const override;
		ios_base::iostate get_state() const override;
		void add_to_state(ios_base::iostate state) override;
		void set_state(ios_base::iostate state) override;
		streamsize get_read_position() override;
		void set_read_position(streamsize position) override;
		void seek_read_position(streamoff offset, ios_base::seekdir direction)
			override;
		void read(span<byte> buffer) override;
		const Container& get_buffer() const noexcept &;
		Container get_buffer() noexcept &&;
		void set_buffer(const Container& new_buffer);
		void set_buffer(Container&& new_buffer);
		void reset_buffer();
	}

TODO

## Class template `output_memory_stream`

	template <Container = vector<byte>>
	class output_memory_stream final : public output_stream
	{
	public:
		output_memory_stream(format f = {});
		output_memory_stream(Container c, format f = {});
		bool is_good() const override;
		bool is_eof() const override;
		bool is_fail() const override;
		bool is_bad() const override;
		ios_base::iostate get_state() const override;
		void add_to_state(ios_base::iostate state) override;
		void set_state(ios_base::iostate state) override;
		streamsize get_write_position() override;
		void set_write_position(streamsize position) override;
		void seek_write_position(streamoff offset, ios_base::seekdir direction)
			override;
		void write(span<const byte> buffer) override;
		const Container& get_buffer() const noexcept &;
		Container get_buffer() noexcept &&;
		void set_buffer(const Container& new_buffer);
		void set_buffer(Container&& new_buffer);
		void reset_buffer();
	}

TODO

## Class template `memory_stream`

	template <Container = vector<byte>>
	class memory_stream final : public stream
	{
	public:
		memory_stream(format f = {});
		memory_stream(Container c, format f = {});
		bool is_good() const override;
		bool is_eof() const override;
		bool is_fail() const override;
		bool is_bad() const override;
		ios_base::iostate get_state() const override;
		void add_to_state(ios_base::iostate state) override;
		void set_state(ios_base::iostate state) override;
		streamsize get_read_position() override;
		void set_read_position(streamsize position) override;
		void seek_read_position(streamoff offset, ios_base::seekdir direction)
			override;
		streamsize get_write_position() override;
		void set_write_position(streamsize position) override;
		void seek_write_position(streamoff offset, ios_base::seekdir direction)
			override;
		void read(span<byte> buffer) override;
		void write(span<const byte> buffer) override;
		const Container& get_buffer() const noexcept &;
		Container get_buffer() noexcept &&;
		void set_buffer(const Container& new_buffer);
		void set_buffer(Container&& new_buffer);
		void reset_buffer();
	}

TODO

## Class `input_file_stream`

	class input_file_stream final : public input_stream
	{
	public:
		input_file_stream(const filesystem::path& file_name, format f = {});
		bool is_good() const override;
		bool is_eof() const override;
		bool is_fail() const override;
		bool is_bad() const override;
		ios_base::iostate get_state() const override;
		void add_to_state(ios_base::iostate state) override;
		void set_state(ios_base::iostate state) override;
		streamsize get_read_position() override;
		void set_read_position(streamsize position) override;
		void seek_read_position(streamoff offset, ios_base::seekdir direction)
			override;
		void read(span<byte> buffer) override;
	}

TODO

## Class `output_file_stream`

	class output_file_stream final : public output_stream
	{
	public:
		output_file_stream(const filesystem::path& file_name, format f = {});
		bool is_good() const override;
		bool is_eof() const override;
		bool is_fail() const override;
		bool is_bad() const override;
		ios_base::iostate get_state() const override;
		void add_to_state(ios_base::iostate state) override;
		void set_state(ios_base::iostate state) override;
		streamsize get_write_position() override;
		void set_write_position(streamsize position) override;
		void seek_write_position(streamoff offset, ios_base::seekdir direction)
			override;
		void write(span<const byte> buffer) override;
	}

TODO

## Class `file_stream`

	class file_stream final : public stream
	{
	public:
		file_stream(const filesystem::path& file_name, format f = {});
		bool is_good() const override;
		bool is_eof() const override;
		bool is_fail() const override;
		bool is_bad() const override;
		ios_base::iostate get_state() const override;
		void add_to_state(ios_base::iostate state) override;
		void set_state(ios_base::iostate state) override;
		streamsize get_read_position() override;
		void set_read_position(streamsize position) override;
		void seek_read_position(streamoff offset, ios_base::seekdir direction)
			override;
		streamsize get_write_position() override;
		void set_write_position(streamsize position) override;
		void seek_write_position(streamoff offset, ios_base::seekdir direction)
			override;
		void read(span<byte> buffer) override;
		void write(span<const byte> buffer) override;
	}

TODO

## Open issues

* Concepts vs inheritance
* `std::span` vs `std::ContiguousRange`
* Dependency on `std::ios_base`.
* Error handling
* `filesystem::path_view`
