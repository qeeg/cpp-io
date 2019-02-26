# A proposal to add std::byte-based IO to the C++

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

	inline unspecified read;
	inline unspecified write;

	}

## Type `floating_point_format`

	enum class floating_point_format
	{
		iec559,
		native
	};

## Type `bom_handling`

	enum class bom_handling
	{
		none,
		read_write
	};

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
	};

## Class `stream_base`

	class stream_base
	{
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
	};

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

## Class `stream`

	class stream : public input_stream, public output_stream
	{
	public:
		stream(format f = {});
	};

## Customization points
### `io::read`

	namespace customization_points
	{

	void read(input_stream& s, byte var);
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
	
	struct read_customization_point
	{
		template <typename T>
		void operator()(input_stream& s, T& var);
	};
	
	}
	
	inline customization_points::read_customization_point read;

### `io::write`

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
	void write(output_stream& s, span<const byte> buffer);
	template <size_t Extent>
	void write(output_stream& s, span<const char8_t, Extent> buffer);
	template <size_t Extent>
	void write(output_stream& s, span<const char16_t, Extent> buffer);
	template <size_t Extent>
	void write(output_stream& s, span<const char32_t, Extent> buffer);

	struct write_customization_point
	{
		template <typename T>
		void operator()(output_stream& s, const T& var);
	};

	}

	inline customization_points::write_customization_point write;

## Open issues

* Concepts vs inheritance
* Dependency on `std::ios_base`.
* Error handling
