---
title: pnglib
description: Static library to generate PNG image files
image: cover.jpg
categories:
    - Image Processing
weight: 3       # You can add weight to some posts to override the default sorting (date descending)
---

Static library to encode or decode PNG files by compressing data into zlib datastreams or decompressing from them.
The aim of this project was to generate an uncorrupted PNG file from scratch. [Source code](https://github.com/cmanziel/pnglib).

## PNG file structure

According to the [PNG specification](https://www.w3.org/TR/png-3) a PNG file has to contain at least each one of this chunks of data:

### IHDR

This is the image header chunk. It contains general information about the image like width, height, pixel data length and format and an 8-byte signature that every PNG file contains.

### IDAT

This is the chunk that contains the actual image pixel data. The original raw data, stored in the format indicated in the IHDR chunk, has to be compressed through the zlib library's DEFLATE routine and put into one or multiple IDAT chunks one after each other. An incorrect zlib's datastream will result in a corrupted PNG file.

### IEND

Chunk at the end of the image file.

## Image Encoding

Each chunk consists of three or four fields:

- **Length**: A four-byte unsigned integer giving the number of bytes in the chunk's *data* field.
- **Chunk type**: The data bytes appropriate to the chunk type, if any. This field can be of zero length.
- **Chunk data**: the actual data for the chunk in use
- **CRC**: A four-byte CRC calculated on the preceding bytes in the chunk, including the chunk type field and chunk data fields, but not including the length field.

When encoding a PNG file the library function *compress* calls the zlib's compression routines, sets the necessary fields for every chunk (like signatures and CRCs) and writes to the image file opened.

## Image Decoding

When decoding a PNG file the library searches all the IDAT chunks in the file, concatenates their data to get the whole zlib datastream, decompresses it to the original pixel data and returns it.
