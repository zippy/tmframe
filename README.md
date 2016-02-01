# TMFRAME

TMFRAME, pronounced "time frame", is a simple and efficient
binary standard for encoding time series data.

SPECIFICATION
=============

The TMFRAME format allows very compact expression of time-series.
For example, for a simple time-series, the TMFRAME encoding
can be as simple as a sequence of 64-bit timestamps (whose
resolution is limited to 10 nanoseconds). However the same
format can be accompanied by much longer additional
event data if need be. Common situations where a single
float64 are needed for the timepoint's value are supported
with exactly two words (two 64-bit words; one for the
timestamp and one for the float64 payload).

# 1. number encoding rules

Integers and floating point numbers are used in the
protocol that follows, so we fix our definitions of these.

 * Integers: are encoded in little-endian format. Signed integers
    use two’s complement. Integers are signed unless otherwise
    noted.
 * float64, also known as 64-bit floating-point numbers: Encoded
   in little-endian IEEE-754 format.

# 2. primary word encoding

A TMFRAME message always starts with a primary word.

Depending on the content of the low 3 bits of the primary word,
the primary word may be the only bytes in the message.
However, there may also be additional bytes following the
primary word that complete the message.

~~~

msb            primary timeframe 64-bit word                  lsb
+-----------------------------------------------------------+---+
|                           A                               |PTI|
+-----------------------------------------------------------+---+

A (61 bits) =
     The primary timeframe 64-bit word is generated by starting
     with a 64-bit signed little endian integer, the number
     of nanoseconds since the unix epoch; then truncating off
     the lowest 3-bits and overwriting them with the value of PTI.
     The resulting A value is the 61 most significant
     bits of the timestamp and can be used directly as an
     integer timestamp by first copying the full 64-bits of the
     timeframe word and then zero-ing out the 3 bits of PTI.
     
PTI (3 bits) = Payload type indicator, decoded as follows:

    0 => a zero value is indicated for this timestamp.
         (the zero value can also be encoded, albeit
         less efficiently, by a UDE word with bits all 0).
    1 => exactly one 64-bit float64 value payload follows.
    2 => exactly two 64-bit float64 values follow.
    3 => time-stamp only, no other value follows. (no UDE
          follows; the next value will be another primary word)
    4 => user-defined-encoding (UDE) descriptor follows.
    5 => NULL: the null-value, a known and intentionally null value. Written as NULL.
    6 => NA: not-available, an unintentionally missing value.
         In statistics, this indicates that *any* value could
         have been the correct payload here, but that the
         observation was not recorded. a.k.a. "Missing data". Written as NA.
    7 => (invalid and forbidden; reserved for future extension)

~~~

# 3. User-defined-encoding descriptor

~~~
msb    user-defined-encoding (UDE) descriptor 64-bit word     lsb
+---------------------------------------------------------------+
|Q-BIT|    UTYPE    |                UCOUNT                     |
+---------------------------------------------------------------+

  Q-BIT => a single bit indicating in the UTYPE is a
      system defined type, or a user-defined type.

      0 => a Q-BIT of zero indicates the UTYPE is system-defined;
           either by this specification, or a later version of this
           specification.
      1 => a Q-BIT of 1 indicates a user-defined UTYPE.
           Users should feel free to define their own types
           within this range. Notice that testing whether the
           entire UBE (treated as a signed 64-bit int) is < 0
           suffices to determine if a user-defined UTYPE is in
           use.

  UCOUNT => is a 43-bit unsigned integer number of bytes that
       follow as a part of this message. Zero is allowed as a
       value in C, and is useful when the type information in D
       suffices to convey the event. Mask off the high 21-bits
       of the UDE to erase the UTYPE before using the count
       of byte count in UCOUNT. The payload starts immediately
       after the UDE, and can be up to 8TB long (2^43 bytes).
       Shorter payloads are recommended whenever possible.

  UTYPE => is a 20-bit unsigned integer giving the type of the
       message to follow. 
       
       Certain D values are pre-defined event-type descriptors,
       defined by this spec, and others are reserved for
       user defined extensions. The Q-BIT tells you which
       namespace is in use.

       System defined UTYPE values as of this writing are:

       0 => this is also a zero value payload. The corresponding
            UCOUNT must also be 0. There are no other words
            in this message. This allows encoders to not
            have to go back and compress out a zero value by
            writing a PTI of zero; although they are encouraged
            to do so whenever possible to save a word of space.

       1 => a TMFRAME-HEADER value follows, giving time-series
            metadata. To be described later.
            
       2 => a Capnproto encoded message segment follows.
       
       3 => a Binc encoded message follows.
       
       4 => a Msgpack encoded message follows.

       5 => a sequence of S-expressions (code or data) in zygomys
            parse format follows. See [https://github.com/glycerine/zygomys]
            (https://github.com/glycerine/zygomys)
       
~~~

Copyright (c) 2016, Jason E. Aten.

LICENSE: MIT license.
