# CBI: CrOS Board Info

[TOC]

## Terms

When we talk about CBI, we use the following definitions:

*   Device: A computer running Chrome OS.
*   Family: A set of devices which share the same reference design. (e.g. Fizz).
*   Model: Within a family, devices are distinguished by non-customizable
    hardware features (e.g. chassis, motherboard). They’re grouped as a model
    and usually given a code name (e.g. Teemo, Sion). Within a model, devices
    are distinguished by SKU.
*   SKU: Numeric value describing customizable hardware features (e.g. CPU,
    memory, storage, touch panel, keyboard, etc.)
*   Project: Avoided in this doc for its (too) wide usage.

## Background

It’s desired to make a single firmware image support different board versions
and variants to simplify development and manufacturing. To do so, firmware
needs to identify the hardware, first. Chrome OS devices historically encoded
such information in resistor straps. Today we store such information in EEPROM
and call it CrOS board info (a.k.a. CBI). This page describes CBI data format,
implementation, advantages, and caveats.

One of the obvious advantages is that EEPROMs can store more data using fewer
pins. As the EC is connected to more chips, its I/O pins are becoming more
precious. I2C buses exist nearly in all boards. Thus, adding an I2C EEPROM
virtually consumes no extra pins.

Second, resistor strapping is significantly harder to be altered. Though this
provides protection against accidental loss of information, hardware information
becomes too rigid. EEPROM contents can be altered if proper tools and
instructions are provided. It allows vendors and RMA centers to stock generic
boards and convert them to a specific model as demands are revealed. This
flexibility is one of the biggest advantages of the CBI/EEPROM solution over
resistor strapping.

Since EEPROMs are physically separated from other data storage, if we choose,
they can be locked down. Chrome OS already has this ‘write-protect’ feature
built in the design. Data stored in write-protected storage can be changed only
through authorized access. We can simply hook an EEPROM’s WP pin to this wire
to get the access control aligned to other firmware storage and provide the same
level of tamper resistance.

## CBI Image Format

 Offset| # bytes | data name        |  description
-------|---------|------------------|---------------------------------------------------
 0     | 3       |MAGIC             | 0x43, 0x42, 0x49 (‘CBI’)
 3     | 1       |CRC               | 8-bit CRC of everything after CRC through ITEM\_n
 4     | 1       |FORMAT\_VER\_MINOR| The data format version. It’s 0.0 (0x00, 0x00) for the initial version.
 5     | 1       |FORMAT\_VER\_MAJOR|
 6     | 2       |TOTAL\_SIZE       | Total number of bytes in the entire data.
 8     | x       |ITEM\_0,..ITEM\_n | List of data items. The format is explained below.

### MAGIC

It’s fixed and invariable across format versions. This allows ECs and tools to
quickly identify CBI data before parsing it.

### CRC

8-bit CRC of data\[4\] through data\[TOTAL\_SIZE\]. Invariable across format
versions. It allows ECs and tools to detect data corruption.

### FORMAT\_VERSION

Stores the data format version. Starts with 0.0 (0x00, 0x00). ECs and tools are
expected to be able to parse all the future versions as long as the major
version is equal to or smaller than the parser’s version. Stored in little
endian (1st half: minor version. 2nd half: major version) so that it can be
interpreted as a single 16-bit value on little endian machines (x86 \& ARM).

### TOTAL\_SIZE

The number of bytes in the entire data structure. This field helps older parsers
compute proper CRC without knowledge on the new data items. The max is 0xFFFF.

## Data Fields

Following after the header fields are data items. They are stored in an array
where each element uses the following format:

 Offset| # bytes | data name        |  description
-------|---------|------------------|---------------------------------------------------
 0     | 1       | \<tag\>          | A numeric value uniquely assigned to each item.
 1     | 1       | \<size\>         | Total size of the item (i.e. \<tag\>\<size\>\<data\>)
 2     | x       | \<value\>        | Value can be a string, raw data, or an integer.

Integer values are stored in the little endian format.
They are written using the smallest size to fit the value and on read, padded
with as many zeros as the reader expects.

Here are a list of standard data items. See `ec/include/cros_board_info.h`
for the latest information. Data sizes can vary project to project. Optional
items are not listed but can be added as needed by the project.

Name              |   tag  | type     | sticky | required |  description
------------------|--------|----------|--------|----------|---------------------------------
BOARD\_VERSION    | 0      | integer  | yes    | yes      | Board version (0, 1, 2, ...)
OEM\_ID           | 1      | integer  | yes    | yes      | OEM ID
MODEL\_ID         | 5      | integer  | no     | no       | ID assigned to each model.
SKU\_ID           | 2      | integer  | no     | yes      | ID assigned to each SKU.
DRAM\_PART\_NUM   | 3      | string   | yes    | no       | DRAM part name in ascii characters
OEM\_NAME         | 4      | string   | yes    | no       | OEM name in ascii characters

Sticky fields are those which are set before SMT and prefrashed to EEPROMs.
They're not expected to be changed after boards are manufactured. Non-sticky fields can be changed for example at RMA center if write protect is disabled.

Required fields are expected to exist on all devices.

### BOARD\_VERSION

The number assigned to each hardware version. This field ideally should be
synchronously incremented as the project progresses and should not differ among
models.

Assignment can be different from project to project but it’s recommended to be
a single-byte field (instead of two-byte field where the upper half describes
phases (EVT, DVT, PVT) and the lower half describes numeric versions in each
phase).

### OEM\_ID

A number assigned to each OEM. Software stack can use this field to select
customizations specific to an OEM. It’s commonly used to control power LEDs
because OEMs tend to prefer a consistent LED behavior across the brand.

### MODEL\_ID

A number assigned to each model. Numbering is managed within each OEM. So,
\<OEM\_ID, MODEL\_ID\> should form a unique ID within the family. This allows a
host tool such as mosys to share the code to derive a model name among variants
in the family. It also allows EC to identify a battery pack, which is usually a
per-model configuration.

### SKU\_ID

The number used to describe hardware configuration or hardware features.
Assignment varies family to family and usually is shared among all models in
the same family.

Some family uses it as an index for a SKU table where hardware features are
described for SKUs. An array of C struct can be auto-generated
([go/cros-publish-hw-config-runtime]).

It can also be a bitmap where each bit or set of bits represents one hardware
feature. For example, it can contain information such as CPU type, form factor,
touch panel, camera, keyboard backlight, etc. Top 8 bits can be reserved for
OEM specific features to allow OEMs to customize devices independently.

### DRAM\_PART\_NUM ([go/octopus-dram-part-number-retrieval])

A string value that is used to identify a DRAM. This is different from RAM ID,
which is an index to a table where RAM configuration parameters are stored. RAM
ID is encoded in resistor strapping. This makes RAM ID visually validatable
(as opposed to being a field in CBI). DRAM\_PART\_NUM is used to track the
inventory.

### OEM\_NAME (TBD)

OEM name in ascii string.

## Software

### EC firmware

CBI should be read up to two times (back-to-back) per boot per image. That is,
on normal boot, EC RO should read it and EC RW should read it once. Once reads
are attempted, the result is preserved even if both reads fail. This would
prevent the system from running in an inconsistent state. [b/80236608] explains
why we don’t let the EC keep reading CBI.

The earliest timing CBI can be read is at HOOK\_INIT with
HOOK\_PRIO\_INIT\_I2C + 1.

### cbi-util

A tool which runs on a build machine. It creates a EEPROM image file with given
field values. Manufacturers use CBI image files to pre-flash EEPROMs in large
volume. This tool also can print the contents of a CBI image file.

### ectool

A tool which runs on a host (i.e. Chromebooks). It can fetch CBI data from the
EEPROM (through the EC). Manufacturers can use this tool to validate the EEPROM
contents. If the board is not write protected, ectool can change the CBI. Note
that some fields are not expected to be changed after the board is manufactured.
The data of existing fields can be changed and resized.

### mosys

Mosys is a tool which runs on the host to provide various information of the
host. One of its sub-command, platform, retrieves board information (e.g. SKU,
OEM) from SMBIOS, which is populated by coreboot. We’ll update coreboot so that
it’ll fetch board information from EEPROM (via EC) instead of resistor strapping.

### cbi\_info

This is a script used by debugd to include CBI dump in user feedback reports.
Currently, it prints only BOARD\_VERSION, OEM\_ID, and SKU\_ID.

## Limitation

It requires the EC to initialize an I2C controller and its driver. There may be
rare cases where board info is needed before these are initialized. To handle
such cases, we may need to add simplified I2C API. Simplified I2C API would
have the following features:

*   Does not do write
*   May only read a single byte at a time
*   May run at lower frequency
*   May assume no multitasking (thus implement no concurrency control)
*   Exist only in RO copy

[go/cros-publish-hw-config-runtime]: https://goto.google.com/cros-publish-hw-config-runtime
[go/octopus-dram-part-number-retrieval]: https://docs.google.com/document/d/17WugKTbeDBWYe4GplmraOKyX9tb_g76Vu5Qckb75oXs/edit?usp=sharing
[b/80236608]: https://b.corp.google.com/issues/80236608