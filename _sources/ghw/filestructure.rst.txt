GHW file structure
##################

.. WARNING::
    This section was written while trying to understand libghw and may be inaccurate or
    outdated.

GHW is a binary format, divided into multiple sections and using native endianness.
Currently the following sections exist:

* The file header starts with the magic header "GHDLwave\\n" and contains some basic
  information like endianness word size, etc. In the future the base time unit may be
  added, which currently is hard coded to femto seconds.
* The strings section contains all data strings used by other sections, which refer to
  these strings by its position index (starting with 1 not 0). I.e. the names of types,
  signals, instances, processes and enum literals are stored here.
* The types section contains all type and subtype definitions.
* *TODO* "well known types"
* The hierarchy section describes the design hierarchy
* The snapshot section contains initial values for all scalar signals. Composite signals
  like arrays and records are not stored directly, but by storing it scalar components.
  The value of such a composite signal can be obtained by retriving the scalar member
  signal ids from the hierarchy section and reading the values of these members.
* The cycles section saves for each simulation cycle the time delta to its predecessor
  and the values of those signals, which changed at the current cycle.
* At the end of the file, a directory section stores the start position of each section.

Sometimes integer values are compressed to reduce the file size. In this case,they are
always stored in little endian format and only the lower 7 bits of each byte are used to
store the data value. An set MSB of each byte indicates, that an additional byte is
required to hold the value. So a 127 would be saved as 0x7f, but an 128 as 0x01, 0x80.
This format is refered to as *uleb128* in case of an unsigned integer, *sleb128* in case
of a 32 bit signed integer an *lsleb128* in case of a 64 signed integer. A negative
value can be detected that bit 7 of the last byte is set. In this case one must not
forget to fill the remaining MSBs during decompression for a vaild two's complement.

File header
===========

Currently the file header at the beginning of a GHW file has the following structure:

+---------------+-----------------------------------+
| type          | content                           |
+===============+===================================+
| 9 byte array  | magic header b"GHDLwave\\n"       |
+---------------+-----------------------------------+
| 1 byte        | header length (currently 16)      |
+---------------+-----------------------------------+
| 1 byte        | file version major                |
+---------------+-----------------------------------+
| 1 byte        | file version minor                |
+---------------+-----------------------------------+
| 1 byte        | endianness: 1 for LE, 2 for BE    |
+---------------+-----------------------------------+
| 1 byte        | word size (4)                     |
+---------------+-----------------------------------+
| 1 byte        | file offset: ?                    |
+---------------+-----------------------------------+
| 1 byte        | zero byte                         |
+---------------+-----------------------------------+

Future additions could include the base time unit. Currently its hard coded to femto
seconds.

Strings
=======

+-----------------------+-------------------------------------------+
| type                  | content                                   |
+=======================+===========================================+
| 8 bytes               | magic header "STR" + 5 zero bytes         |
+-----------------------+-------------------------------------------+
| 4 byte signed integer | number of strings                         |
+-----------------------+-------------------------------------------+
| 4 byte signed integer | number of bytes of all expanded strings?  |
+-----------------------+-------------------------------------------+
| string entry 1        | has variable length                       |
+-----------------------+-------------------------------------------+
| string entry 2        | has variable length                       |
+-----------------------+-------------------------------------------+
| string entry ...      | has variable length                       |
+-----------------------+-------------------------------------------+
| 1 byte                | zero byte                                 |
+-----------------------+-------------------------------------------+
| 4 bytes               | "EOS" + zero byte                         |
+-----------------------+-------------------------------------------+

All strings are encoded with a subset the ASCII encoding. At the beginning of each
string entry a compressed unsigned integer (type is *uleb32*) denotes the number of
characters at the beginning of the previous string, that must be prepended to the string
of this entry. Obviously that number must be zero for the first entry. I.e. the
successive strings "std_ulogic" and "std_logic" would result in the following entries:

::

    [[0, "std_ulogic"], [4, "logic]]

Integers of type *uleb32* have a variable length. The first byte contains bits 4 down to
0 of the integer, the next byte bits 9 down to 5, etc. in their lower bits. A set bit 7
indicates, that a further bit is required for the value of this integer. A set bit 6 or 
5 (or in the sense of logical or, not logical xor) indicates, that the current bit is a
character as part of a string, else it is part of a compressed integer. As strings are
not zero terminated, this is the only possibility to detect the beginning of a new
string entry, but reduces the number of ASCII characters available.

Types
=====

The signal type and subtype definitions are stored here. To understand these GHW type
definitions, chapter 5 of *IEEE Std 1076-XXXX: IEEE Standard for VHDL Language Reference
Manual* is good lecture. Only scalar and composite types without its operations are
necessary for GHW files. Scalar types consist of enums, ints, physical types and floats.
Scalar subtypes are a collection of elements. In case all elements are of the same type,
it is called an array, otherwise a record. Both can be un constrained, partially
or fully constrained, which relates to the contained array indices. If the boundaries
for all contained array indices are known, the type is fully constrained. There are also
subtypes of composite types.

+-----------------------+-------------------------------------------+
| type                  | content                                   |
+===============+===================================================+
| 8 bytes               | magic header "TYP" + 5 zero bytes         |
+-----------------------+-------------------------------------------+
| 4 byte signed integer | number of types                           |
+-----------------------+-------------------------------------------+
| variable length       | type 1                                    |
+-----------------------+-------------------------------------------+
| variable length       | type 2                                    |
+-----------------------+-------------------------------------------+
| variable length       | type ...                                  |
+-----------------------+-------------------------------------------+
| 1 byte                | zero byte                                 |
+-----------------------+-------------------------------------------+

The size and content of a type entry depends on the kind of type, but every entry starts
with an unsigned byte denoting the kind of entry. The exact value is defined by the
*Ghdl_Rtik* enum, though not all *Ghdl_Rtik* values denote a type. Following is the ID
of the string of the type name. This ID is a compressed unsigned integer (*uleb128*).

Base types
----------

Base types include:

* Type common (unused)
* Type enum
* Type scalar
* Type physical
* Type array
* Type record

::

    TODO: figure out the purpose of type common

Subtypes
--------

Subtypes include:

* Subtype scalar
* Subtype array
* Subtype unbounded array
* Subtype record
* Subtype unbounded record

Ghdl_Rtik_Type_B1, Ghdl_Rtik_Type_E8 (enum):

+---------------+-----------------------------------+
| type          | content                           |
+===============+===================================+
| 1 byte (u8)   | kind of type (ghdl_rtik)          |
+---------------+-----------------------------------+
| uleb128       | id of type name in strings section|
+---------------+-----------------------------------+
| uleb128       | number of literals                |
+---------------+-----------------------------------+
| uleb128[]     | array of literal string ids       |
+---------------+-----------------------------------+

Ghdl_Rtik_Type_I32, Ghdl_Rtik_Type_I64, Ghdl_Rtik_Type_F64 (scalar):

+---------------+-----------------------------------+
| type          | content                           |
+===============+===================================+
| 1 byte (u8)   | kind of type (ghdl_rtik)          |
+---------------+-----------------------------------+
| uleb128       | id of type name in strings section|
+---------------+-----------------------------------+
| uleb128       | number of unit definitions        |
+---------------+-----------------------------------+
| GhwUnit[]     | unit definitions                  |
+---------------+-----------------------------------+

Ghdl_Rtik_Type_P32, Ghdl_Rtik_Type_P64 (physical):

+---------------+-----------------------------------+
| type          | content                           |
+===============+===================================+
| 1 byte (u8)   | kind of type (ghdl_rtik)          |
+---------------+-----------------------------------+
| uleb128       | id of type name in strings section|
+---------------+-----------------------------------+

Ghdl_Rtik_Subtype_Scalar (subtype of any scalar type)

+---------------+-----------------------------------+
| type          | content                           |
+===============+===================================+
| 1 byte (u8)   | kind of type (ghdl_rtik)          |
+---------------+-----------------------------------+
| uleb128       | id of type name in strings section|
+---------------+-----------------------------------+
| GhwRange      | value range                       |
+---------------+-----------------------------------+

Ghdl_Rtik_Type_Array

+---------------+-----------------------------------+
| type          | content                           |
+===============+===================================+
| 1 byte (u8)   | kind of type (ghdl_rtik)          |
+---------------+-----------------------------------+
| uleb128       | id of type name in strings section|
+---------------+-----------------------------------+
| uleb128       | number of dimensions              |
+---------------+-----------------------------------+
| uleb128[]     | type id of each range             |
+---------------+-----------------------------------+

Ghdl_Rtik_Type_Subtype_Array

+---------------+-----------------------------------+
| type          | content                           |
+===============+===================================+
| 1 byte (u8)   | kind of type (ghdl_rtik)          |
+---------------+-----------------------------------+
| uleb128       | id of type name in strings section|
+---------------+-----------------------------------+
| TODO          | composite bounds                  |
+---------------+-----------------------------------+

Ghdl_Rtik_Type_Record, Ghdl_Rtik_Type_Unbounded_Record

+---------------+-----------------------------------+
| type          | content                           |
+===============+===================================+
| 1 byte (u8)   | kind of type (ghdl_rtik)          |
+---------------+-----------------------------------+
| uleb128       | id of type name in strings section|
+---------------+-----------------------------------+
| uleb128       | number of elements                |
+---------------+-----------------------------------+
| 2*uleb128[]   | string and type id of each element|
+---------------+-----------------------------------+

Ghdl_Rtik_Subtype_Record

+---------------+-----------------------------------------------+
| type          | content                                       |
+===============+===============================================+
| 1 byte (u8)   | kind of type (ghdl_rtik)                      |
+---------------+-----------------------------------------------+
| uleb128       | id of type name in strings section            |
+---------------+-----------------------------------------------+
| uleb128       | id of base type                               |
+---------------+-----------------------------------------------+
| TODO          | composite bounds, if base type is unbounded   |
+---------------+-----------------------------------------------+


Ghdl_Rtik_Subtype_Unbounded_Record, Ghdl_Rtik_Subtype_Unbounded_Array

+---------------+-----------------------------------+
| type          | content                           |
+===============+===================================+
| 1 byte (u8)   | kind of type (ghdl_rtik)          |
+---------------+-----------------------------------+
| uleb128       | id of type name in strings section|
+---------------+-----------------------------------+
| uleb128       | id of base type                   |
+---------------+-----------------------------------+

Helper entries
--------------

GhwRange

CompositeBounds: see https://github.com/ghdl/ghdl/blob/master/src/grt/grt-waves.adb#L1283

Well known types
================

+---------------+-----------------------------------+
| type          | content                           |
+===============+===================================+
| 8 bytes       | magic header "WKT" + 5 zero bytes |
+---------------+-----------------------------------+
| --            | `TODO`                            |
+---------------+-----------------------------------+

Hierarchy
=========

The hierarchy tree of the simulation.

+---------------+-----------------------------------+
| type          | content                           |
+===============+===================================+
| 8 bytes       | magic header "HIE" + 5 zero bytes |
+---------------+-----------------------------------+
| i32           | number of subcomponents           |
+---------------+-----------------------------------+
| i32           | number of scope signals           |
+---------------+-----------------------------------+
| i32           | number of (scalar) signals        |
+---------------+-----------------------------------+
| GhwHie[]      | Hierarchy items                   |
+---------------+-----------------------------------+
| byte          | zero byte                         |
+---------------+-----------------------------------+

Hierarchy elements can be classified into 3 categories:

1) Blocks, which describe hierarchy kinds block, generate*, instance, generic or package
2) Signals, which describe signals and ports
3) Meta class, which are used to mark the end of a scope or the hierarchy

Each element (GhwHie) starts as follows:

+---------------+---------------------------------------------------+
| type          | content                                           |
+===============+===================================================+
| unsigned byte | kind of this hierarchy element (ghw_hie_kind)     |
+---------------+---------------------------------------------------+
| uleb128       | hierarchy id of its parent                        |
+---------------+---------------------------------------------------+
| uleb128       | id of name string                                 |
+---------------+---------------------------------------------------+
| uleb128       | id of its brother                                 |
+---------------+---------------------------------------------------+

all ids can be zero, i.e. if it is the top level element, it hase no parent.

*TODO block*
*TODO signal*

End of header marker
====================

+---------------+-----------------------------------+
| type          | content                           |
+===============+===================================+
| 4 bytes       | magic header "EOH" + 1 zero byte  |
+---------------+-----------------------------------+

Snapshot
========

+---------------+---------------------------------------+
| type          | content                               |
+===============+=======================================+
| 8 bytes       | magic header "SNP" + 5 zero bytes     |
+---------------+---------------------------------------+
| i64           | current time                          |
+---------------+---------------------------------------+
| GhwVal[]      | inital values of all scalar signals   |
+---------------+---------------------------------------+
| 4 bytes       | content: "ESN\0"                      |
+---------------+---------------------------------------+

Cycles
======

+---------------+-----------------------------------+
| type          | content                           |
+===============+===================================+
| 4 bytes       | magic header "CYC" + 1 zero bytes |
+---------------+-----------------------------------+
| Cycle[]       | Values of all simulation cycles   |
+---------------+-----------------------------------+

Single cycle content:

+---------------+-----------------------------------+
| type          | content                           |
+===============+===================================+
| lsleb         | Time delta to previous cycle      |
+---------------+-----------------------------------+
|[]             | values of changed signals         |
+---------------+-----------------------------------+

After the last cycle an *lsleb* with value -1 follows (instead of the next time delta)
to mark the end of the simulation run.

The value entries consist of the index of the scalar signal in the signal table and the
new signal value itself. The signal index is stored as *uleb128*. Further it is not the
signal index itself, but the difference to the previous signal in that cycle. Assuming
the signals 1,3 and 5 have changed in a cycle, the cycle would be stored as follows:
[1, sigval1, 2, sigval3, 2, sigval5, 0]. The 0 marks the end of this cycle. The type of
sigvalX depends of the type of the signal and is equivalent to the snapshot section
type.

Directory
=========

This section contains the byte start addresses of each previous section in the file.

+---------------+-----------------------------------+
| type          | content                           |
+===============+===================================+
| 4 bytes       | magic header "DIR" + zero byte    |
+---------------+-----------------------------------+
| 1 byte        | endianness: 1 for LE, 2 for BE    |
+---------------+-----------------------------------+
| 1 byte        | word size (4)                     |
+---------------+-----------------------------------+
| 1 byte        | file offset: ?                    |
+---------------+-----------------------------------+
| section addr[]| section ids and addresses         |
+---------------+-----------------------------------+

Tail
====

The tail section has a constant size of 12 bits and points to the begin of the directory
section, which can vary in size with future additions.

+---------------+-----------------------------------+
| type          | content                           |
+===============+===================================+
+---------------+-----------------------------------+
| 4 bytes       | "TAI\0"                           |
+---------------+-----------------------------------+
| 1 byte        | endianness: 1 for LE, 2 for BE    |
+---------------+-----------------------------------+
| 1 byte        | word size (4)                     |
+---------------+-----------------------------------+
| 1 byte        | file offset: ?                    |
+---------------+-----------------------------------+
| u32           | start address of dir section      |
+---------------+-----------------------------------+