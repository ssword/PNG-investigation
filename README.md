# PNG Investigation

A investigation into PNG format with Python, possibly Rust in the future.  
This project is inspired by [this passionate blog](https://www.da.vidbuchanan.co.uk/blog/hello-png.html) from David Buchanan.

Related materials including:

- [PNG specifications version 1.0](https://www.w3.org/TR/REC-png-961001) which was released in 1996.  




A minimum-viable PNG file has the following structure:

```code
PNG signature || "IHDR" chunk || "IDAT" chunk || "IEND" chunk
```

The PNG signature (aka "magic bytes") is defined as:

```binary
"89 50 4E 47 0D 0A 1A 0A" (hexadecimal bytes)
```

Or, expressed as a Python bytes literal:

```python
b'\x89PNG\r\n\x1a\n'
```

## PNG Chunks

After the signature, the rest of the PNG is just a sequence of Chunks. They each have the same overall structure:

```code
Length      - A 31-bit unsigned integer (the number of bytes in the Chunk Data field)
Chunk Type  - 4 bytes of ASCII upper or lower-case characters
Chunk Data  - "Length" bytes of raw data
CRC         - A CRC-32 checksum of the Chunk Type + Chunk Data
```

PNG uses Network Byte Order (aka "big-endian") to encode integers as bytes. "31-bit" is not a typo - PNG defines a "PNG four byte integer", which is limited to the range 0 to 231-1, to defend against the existence of C programmers.

The CRC field is a CRC-32 checksum. The spec gives a terse mathematical definition, but we can ignore all those details and use a library to handle it for us.

### The IHDR (Image Header) Chunk

The IHDR Chunk contains the most important metadata in a PNG - and in our simplified case, all the metadata of the PNG. It encodes the width and height of the image, the pixel format, and a couple of other details:

```table
Name                Size

Width               4 bytes
Height              4 bytes
Bit depth           1 byte
Colour type         1 byte
Compression method  1 byte
Filter method       1 byte
Interlace method    1 byte
```

### The IDAT (Image Data) Chunk

The IDAT chunk contains the image data itself, after it's been Filtered and then Compressed (to be explained shortly).
The data may be split over multiple consecutive IDAT chunks, but for our purposes, it can just go in one big chunk.

### The IEND (Image Trailer) Chunk

This chunk has length 0, and marks the end of the PNG file. Note that a zero-length chunk must still have all the same fields as any other chunk, including the CRC.

### Filtering

The idea of filtering is to make the image data more readily compressible.
You may recall that the IHDR chunk has a "Filter method" field. The only specified filter method is method 0, called "adaptive filtering" (the others are reserved for future revisions of the PNG format).
In Adaptive Filtering, each row of pixels is prefixed by a single byte that describes the Filter Type used for that particular row. There are 5 possible Filter Types, but for now, we're only going to care about type 0, which means "None".
If we had a tiny 3x2 pixel image comprised of all-white pixels, the filtered image data would look something like this: (byte values expressed in decimal)

