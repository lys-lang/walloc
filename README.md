# walloc

walloc is a bare-bones implementation of `malloc` for use by C
programs when targetting WebAssembly.  It is a single-file
implementation with no dependencies: no stdlib, no JavaScript imports,
no emscripten.

Emscripten includes a couple of good malloc implementations; perhaps
consider using one of those?  But if you are really looking for a
bare-bones malloc, walloc is fine.

## Test

```
$ make CC=$LLVM/clang LD=$LLVM/wasm-ld JS=node test
clang -DNDEBUG -Oz --target=wasm32 -nostdlib -c -o test.o test.c
clang -DNDEBUG -Oz --target=wasm32 -nostdlib -c -o walloc.o walloc.c
wasm-ld --no-entry --import-memory -o test.wasm test.o walloc.o
node test.js
node test.js
wasm log: walloc bytes: 0
wasm log: allocated ptr: 0
wasm log: walloc bytes: 1
wasm log: allocated ptr: 131328
wasm log: walloc bytes: 2
wasm log: allocated ptr: 131336
wasm log: walloc bytes: 3
wasm log: allocated ptr: 131344
wasm log: walloc bytes: 4
...
```

You can link `walloc.c` into your program just by adding it to your link
line, as above.

## Size

The resulting wasm file is about 1.5 kB.

## Design

When a C program is compiled to WebAssembly, the resulting wasm module
(usually) has associated linear memory.  It can be compiled in a way
that the memory is created by the module when it's instantiated, or such
that the module is given a memory by its host.  By default, wasm modules
import their memory.

The linear memory has the usual data, stack, and heap segments.  The
data and stack are placed first.  The heap starts at the `&__heap_base`
symbol.  All bytes above `&__heap_base` can be used by the wasm program
as it likes.  So `&__heap_base` is the lower bound of memory managed by
walloc.

The upper bound of memory managed by walloc is the total size of the
memory, which is aligned on 64-kilobyte boundaries.  (WebAssembly
ensures this alignment.)  Walloc manages memory in 64-kb pages as well.
It starts with whatever memory is initially given to the module, and
will expand the memory if it runs out.  The host can specify a maximum
memory size, in pages; if no more pages are available, walloc's `malloc`
will simply return `NULL`; handling out-of-memory is up to the caller.

If you really care about the allocator's design, probably you should use
some other allocator whose characteristics are more well known!

That said, walloc has two allocation strategies: small and large
objects.

### Large objects

A large object is more than 256 bytes.

There is a global freelist of available large objects, each of which has
a header indicating its size.  When allocating, walloc does a best-fit
search through that list.  

Large object allocations are rounded up to 256-byte boundaries,
including the header.

If there is no object on the freelist that can satisfy an allocation,
walloc will expand the heap by the size of the allocation, or by half of
the current walloc heap size, whichever is larger.  The resulting page
or pages form a large object that can satisfy the allocation.

If the best object on the freelist has more than a chunk of space on the
end, it is split, and the tail put back on the freelist.  A chunk is 256
bytes.

So each page is 65536 bytes, and each chunk is 256 bytes, meaning there
are 256 chunks in a page.  So the first chunk in a page that begins an
allocated object, large or small, contains a header chunk.  The page
header has a byte for each chunk in the page.  The byte is 255 if the
corresponding chunk starts a large object, or otherwise if nonzero is
the granule size of the chunk.

When splitting large objects, we avoid starting a new large object on a
page header chunk.  A large object can only span where a page header
chunk would be if it includes the entire page.

Freeing a large object pushes it on the global freelist.  We know a
pointer is a large object by looking at the page header.  We know the
size of the allocation, because the large object header precedes the
allocation.

### Small objects

Small objects are allocated from segregated freelists.  The granule size
is 8 bytes; there is a free list for allocations of up to 1 granule, 2
granules, and so on up to 32 granules, which is 256 bytes, or a whole
chunk.  If there is nothing on the corresponding freelist, walloc will
allocate a new large object, then change its chunk kind in the page
header to the granule size.  It then goes through the fresh chunk,
threading the objects through each other onto a free list.

Freeing a small object pushes it back on its size class's free list.  We
know the size class (number of granules) by looking in the chunk kind in
the page header.

## License

`walloc` is available under the [Blue Oak Model
License](https://blueoakcouncil.org/license/1.0.0), version 1.0.0.  See
[LICENSE.md](./LICENSE.md) for full details.  If you are thinking of
using it and this license is a barrier to adoption, drop me a line at
`wingo@igalia.com`.