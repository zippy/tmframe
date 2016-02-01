# TMFRAME

TMFRAME, pronounced "time frame", is a simple and efficient
binary standard for encoding time series data.

specification
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

# overview of the format

A TMFRAME message always starts with a primary word.

Depending on the content of the low 3 bits of the primary word,
the primary word may be the only bytes in the message.
However, there may also be additional bytes following the
primary word that complete the message.

TMFRAME messages will either be 8 bytes (primary word only);
16 bytes long (primary word + UDE word only); or
greater than 16 bytes long.

Frequently a TMFRAME message will consist of one primary word,
one UDE word, and a variable length payload.

The primary word and UDE word are always 64-bits. The payload
can be up to 2^43 bytes in length.

We illustrate the possible TMFRAME message lengths here:

a) primary word only

~~~
+---------------------------------------------------------------+
|      primary word (64-bits) with PTI={0, 1, 4, 5, or 6}       |
+---------------------------------------------------------------+
~~~

b) primary word and UDE word only:

~~~
+---------------------------------------------------------------+
|                primary word (64-bits) with PTI=7              |
+---------------------------------------------------------------+
|            User-defined-encoding (UDE) descriptor             |
+---------------------------------------------------------------+
~~~

c) primary word + UDE word + variable byte-length message:

~~~
+---------------------------------------------------------------+
|                primary word (64-bits) with PTI=7              |
+---------------------------------------------------------------+
|            User-defined-encoding (UDE) descriptor             |
+---------------------------------------------------------------+
|               variable length                                 |
|                message here                          ----------
|     (the UDE supplies the exact byte-count)          |
+-------------------------------------------------------
~~~

There are also two special payload types that are not UDE based,
as they handle the common case of attaching one or two
float64 values to a timestamp.

d) primary word + one float64

~~~
+---------------------------------------------------------------+
|                primary word (64-bits) with PTI=2              |
+---------------------------------------------------------------+
|                     V0 (float64)                              |
+---------------------------------------------------------------+
~~~

e) primary word + two float64

~~~
+---------------------------------------------------------------+
|                primary word (64-bits) with PTI=3              |
+---------------------------------------------------------------+
|                     V0 (float64)                              |
+---------------------------------------------------------------+
|                     V1 (float64)                              |
+---------------------------------------------------------------+
~~~


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

~~~

msb                  primary word (64-bits)                   lsb
+-----------------------------------------------------------+---+
|                        TMSTAMP                            |PTI|
+-----------------------------------------------------------+---+

TMSTAMP (61 bits) =
     The primary word is generated by starting
     with a 64-bit signed little endian integer, the number
     of nanoseconds since the unix epoch; then truncating off
     the lowest 3-bits and overwriting them with the value of PTI.
     The resulting TMSTAMP value is the 61 most significant
     bits of the timestamp and can be used directly as an
     integer timestamp by first copying the full 64-bits of the
     timeframe word and then zero-ing out the 3 bits of PTI.
     
PTI (3 bits) = Payload type indicator, decoded as follows:

    0 => a zero value is indicated for this timestamp.
         (the zero value can also be encoded, albeit
         less efficiently, by a UDE word with bits all 0).
         
         Use the zero-value for time-stamp only time-series.

         The primary word is the only word in this message.
         The next word will be the primary word of the next
         message on the wire.

         By convention, the 0 value can indicate the
         payload false for boolean series.

    1 => value of 1 (or by convention, a value of true for boolean series).

         The primary word is the only word in this message.

    2 => exactly one 64-bit float64 payload value follows.
         Nmemonic: The total number of 64-bit words in the message is 2.
         
    3 => exactly two 64-bit float64 payload values follow.
         Nmemonic: The total number of 64-bit words in the message is 3.
         
    4 => NULL: the null-value, a known and intentionally null value. Written as NULL.

         The primary word is the only word in this message.

    5 => NA: not-available, an unintentionally missing value.
         In statistics, this indicates that *any* value could
         have been the correct payload here, but that the
         observation was not recorded. a.k.a. "Missing data". Written as NA.

         The primary word is the only word in this message.

    6 => NaN: not-a-number, IEEE 754 floating point NaN value.
         Obtained when dividing zero by zero, for example. math.IsNaN()
         detects these.

         The primary word is the only word in this message.

    7 => user-defined-encoding (UDE) descriptor word follows.

~~~

# 3. User-defined-encoding descriptor

~~~
msb    user-defined-encoding (UDE) descriptor 64-bit word     lsb
+---------------------------------------------------------------+
|Q-BIT|    UTYPE    |                UCOUNT                     |
+---------------------------------------------------------------+

or equivalently:

msb    user-defined-encoding (UDE) descriptor 64-bit word     lsb
+---------------------------------------------------------------+
| EVTNUM (21-bits)  |                UCOUNT (43-bits)           |
+---------------------------------------------------------------+


  Q-BIT => a single bit indicating in the UTYPE is a
      system defined type, or a user-defined type.

      0 => a Q-BIT of zero indicates the UTYPE is system-defined;
           either by this specification, or a later version of this
           specification. The corresponding EVTNUM event
           numbers will be zero or positive.
      1 => a Q-BIT of 1 indicates a user-defined UTYPE.
           If needed, Users should feel free to define their
           own type extensions with Q-BIT set to 1 and
           with UTYPE between [2, 2^20]. The corresponding
           EVTNUM even numbers will be negative, as
           EVTNUM takes its sign from the Q-BIT and its
           absolute value from the UTYPE, using the two's
           compliment encoding for integers.

  UCOUNT => is a 43-bit unsigned integer number of bytes that
       follow as a part of this message. Zero is allowed as a
       value in UCOUNT, and is useful when the type information in UTYPE
       suffices to convey the event. Mask off the high 21-bits
       of the UDE to erase the UTYPE and Q-BIT before using the count
       of bytes found in UCOUNT. The payload starts immediately
       after the UDE word, and can be up to 8TB long (2^43 bytes).
       Shorter payloads are recommended whenever possible.

       There is no requirement that UCOUNT be padded to
       any alignment boundary. It should be the exact length
       of the payload in bytes.

       The next message's primary word will commence after the
       UCOUNT bytes that follow the UDE.

  UTYPE => is a 20-bit unsigned integer giving the type of the
       message to follow. 
       
       Certain UTYPE values are pre-defined event-type descriptors,
       defined by this spec, and others are reserved for
       user defined extensions. The Q-BIT tells you which
       namespace is in use.

  EVTNUM => this is the concatenation of Q-BIT and UTYPE.

       Putting the Q-BIT and UTYPE together, we get a
       21-bit signed integer capable of expressing
       values in the range [-(2^20), (2^20)-1]. We will refer
       to 21-bit signed integer as the EVTNUM value.

       There is one pre-defined user-defined event number.
       The one pre-defined user EVTNUM value is:

       -1 => an error message string in utf8 follows.

       Any custom user-defined types added by the user will
       therefore start at EVTNUM = -2. The last usable EVTNUM is
       the -1 * (2^20) value; so over one million user
       defined event types are available.

       System defined EVTNUM values as of this writing are:

       0 => this is also a zero value payload. The corresponding
            UCOUNT must also be 0. There are no other words
            in this message. This allows encoders to not
            have to go back and compress out a zero value by
            writing a PTI of zero; although they are encouraged
            to do so whenever possible to save a word of space.

       1 => an error message string in utf8 follows.

       2 => a TMFRAME-HEADER value follows, giving time-series
            metadata. To be described later.
            
       3 => a Msgpack encoded message follows.
       
       4 => a Binc encoded message follows.
       
       5 => a Capnproto encoded message segment follows.

       6 => a sequence of S-expressions (code or data) in zygomys
            parse format follows. [note 1]
 
       7 => the payload is a UTF-8 encoded string.
~~~

After any variable length payload that follows the UDE word, the
next TMFRAME message will commence with its primary word.

This concludes the specification of the TMFRAME format.

# conclusion

TMFRAME is a very simple yet flexible format for time series data. It allows
very compact and dense information capture, while providing the
ability to convey and attach full event information to each timepoint as
required.

### notes

[1] For zygomys parse format, see [https://github.com/glycerine/zygomys](https://github.com/glycerine/zygomys)


Copyright (c) 2016, Jason E. Aten.

LICENSE: MIT license.
