Endianness
==========

[Endianess](https://en.wikipedia.org/wiki/Endianness) is something which is both
quite simple and also potentially quite confusing. Put simply, an understanding
of endianess for your CPU and test-bench can be understood as follows:

- The memory bus for both the Bus and Harvard CPU works purely in terms of
  moving 8-bit bytes to and from byte addresses. It does this 4-bytes at a
  time for efficiency, but the data passing over the bus is never considered
  to be a 32-bit integer.

- The CPU can issue read and write transactions over the bus, and these may
  consist of 1, 2, or 4 consecutive bytes[1]. Those reads and writes
  will come from or go to 32-bit registers (or be treated as 32-bit instructions).
  When operating on 32-bit values, the CPU operates in big-endian mode.

So everything todo with endianess is todo with CPU and the way it interprets
16-bit or 32-bit. The memory on the end of the bus is completely un-aware
of endianess, and just deals with byte addresses (albeit while moving up to
four consecutive bytes at a time).

Endianness and the memory bus
-----------------------------

The bus has a number of multi-bit signals of interest:

- `address` : a 32-bit address represented as a 32-bit port
- `writedata` and `readdata` : 4 bytes represented as a 32-bit port
- `byteenable` : 4 byte enables represented as a 4-bit port

The address is simply an unsigned integer, with `address[31]` containing
the MSB, and `address[0]` the LSB. Technically the two low-order bits
are not needed, as all transactions must be aligned to a 4-byte boundary.
This is for exactly the same reason that MIPS only supports word-aligned
memory accesses, and is noted in the Avalon spec (3.6):

> The interconnect only supports aligned accesses. A host can only issue addresses that
> are a multiple of its data width in symbols. A host can write partial words by
> deasserting some byteenables. For example, the byteenables of a write of 2
> bytes at address 2 is 4’b1100.

The 32-bits of the data signals are mapped directly to addresses, as a 
sequence of bytes:

- `writedata[7:0]` refers to the byte at `address+0`
- `writedata[15:8]` refers to the byte at `address+1`
- `writedata[23:16]` refers to the byte at `address+2`
- `writedata[31:24]` refers to the byte at `address+3`

Each byte is ordered from MSB to LSB, so `writedata[7]` is the MSB of
`address+0` and `writedata[0]` is the LSB.
This is the most intuitive way of mapping it, as it means
that `writedata[x*8+y]` corresponds to bit `y` of address `address+x`.

The Avalon spec mentions this in chapter 11:

> To ensure correct data communication, the Avalon-MM interface specification
> requires that each master or slave port of all components in your system pass data in
> descending bit order with data bits 7 down to 0 representing byte offset 0.

Taking this information, and the early quote about byte-enables, we can
say that `byteenable[x]` controls whether `writedata[x*8+7:x*8]` will be
written to `address+x`. In a read transaction `byteenable[x]` determines
whether `address+x` is read or not.

Byte enables on _reads_ may seem unimportant when dealing with RAM, and
that's true. But for a real system with peripherals it may be important
to control exactly which byte is being read, as reading from a memory-mapped
byte may actually perfom an operation in the peripheral. For example, it
might retrieve the next byte from a read FIFO in a UART.

While your CPU should correctly issue byte-enables for reads, you don't
really need to worry about it in your test-bench. For example, if you
have a test-bench that does `lbu $1, 6($0)`, then it should result in 
a load of byte 6, which results in a transaction with `address=4` and
`byteenable=0b0100`. Only `readdata[23:16]` will contain valid data,
so it is pretty much up to your test-bench what to return for the
other bits - you could return 0s, Xs, or anything else. You might
worry about propagation of Xs if you were to return `readdata=32'hXXFEXXXX`,
but it is actually not a problem, as every single load instruction is
only dependent on a well-defined sub-set of `readdata`. In our example
of `lbu $1, 6($0)` it is _required_ that `$1` receives the value `32'h000000FE`.
There is simply no way in the ISA for those un-initialised bits to
affect the results of the instruction (if they do, then that is a problem
with the CPU implementation).

Endianess and the CPU
---------------------

Within the CPU you do have to care about endianess, but only at well-defined
points. Consider the following sequence:

```
lbu $1, 10($0)
addiu $1, $1, 23
sbu $1, 10($0)
```

This fragment only reads and writes arithmetic as bytes, so it is completely
endianness independent with respect to the data it reads and writes.

Consider this fragment of C code:

```
void f(uint8_t *p)
{
    // Load word
    uint32_t tmp=0;
    for(int i=0; i<4; i++){
        tmp=(tmp<<8)+p[i];
    }

    // Add constant
    tmp += 23;

    // Store word
    for(int i=0; i<4; i++){
        p[i]=tmp&0xFF;
        tmp=tmp>>8;
    }
}
```

If you look at the load and store, you might think "ah, it is reading and
writing 32-bit values in little-endian order!", and you would be completely
correct. However... this code will produce exactly the same results on 
a little-end and bit-endian CPU, as it only ever reads and writes bytes.
Yes arithmetic is done on the 32-bit register `tmp`, but registers do
not have endianess.

For a true example of endian dependent code, consider this fragment:

```
void g(uint32_t *p)
{
    uint32_t tmp = *p;
    tmp += 23;
    *p = tmp;
}
```

Now we have code that it is endian dependent - not because of `tmp += 23`,
but because of the 32-bit loads and stores. These are the points at
which endianess matters - when you load a multi-byte integer from byte storage,
and when you store a multi-byte integer to byte storage.

Imagine the state of our memory is:

- address[0x100] = 0x00
- address[0x101] = 0x01
- address[0x102] = 0x02
- address[0x103] = 0x03

From the RAMs point of view these are just bytes. The RAM itself has no opinion
on what 32-bit integers these might represent.

If we were to run `g((uint8_t*)0x100)` on a **little-endian CPU**,
the statement `tmp=*p` would cause a 32-bit load, so `tmp` would
receive the 32-bit integer `0x03020100 = 50462976`. The addition
`tmp += 23` would result in the new value `0x3020117 = 50462999`.
This would then be written back by `*p=tmp`:

- address[0x100] = 0x17
- address[0x101] = 0x01
- address[0x102] = 0x02
- address[0x103] = 0x03

In a **big-endian CPU**, the 32-bit load `tmp=*p` would result in the
32-bit integer `0x00010203 = 66051`. After adding 23 we get
`0x0001021A = 66074`, so the final memory state would be:

- address[0x100] = 0x00
- address[0x101] = 0x01
- address[0x102] = 0x02
- address[0x103] = 0x1A

Summary
-------

Overall the endianess conversions in your CPU can be summarised as:

> Apply endianness conversions when you convert bytes from memory into a 32-bit
> register, or when you store a 32-bit register back into bytes. Apart from
> that, ignore it.

Your test-bench and test-cases _may_ need to be aware of endianness - it depends
how you design it.

---

[1] - Technically LWL and LWR might result in 3 consecutive bytes, but we'll
      ignore that.