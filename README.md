# nmea

This a design document for the type of interface and implementation
the "ideal" NMEA1083 parser should be in C.

There is not yet sufficient call to sit down and code this but were the need to arise,
this is the design document on which it would be based.

## Requirements

An NMEA parser in C should have the following requirements:

1. Written in pure, idiomatic C, with no dependencies outside of what a
reasonable C compiler would provide.

1. Friendly for MCUs and small SoCs (let's use the *IoT* buzzword):
    - small memory footprint
    - no runtime memory allocation
    - functional across a wide range of processor word sizes
    - functional without floating point
    - suitable for bare-metal use (no runtime linker or syscall dependencies)
    - algorithmically efficient

1. Good interface design:
    - can be called on large buffers with multiple sentences
        or on the single-character input from a UART
    - obtain parsing status and GPS output as it is parsed
    - diagnose failed parsing (bad checksum, etc)
    - can be used to quickly generate arbitrary sentences

1. Complete support of NMEA0183 sentences and common manufacturer extensions
such as `PUBX`.
    - see <https://gpsd.gitlab.io/gpsd/NMEA.html>

1. Good documentation:
    - how to call and use the code
    - how to build and link with different toolchains/build systems
    - give a basic reference/understanding of NMEA data being handled

### Current work

These existing libraries were examined and do not meet the requirements
for one or more reasons.

1. [jacketizer/libnmea](https://github.com/jacketizer/libnmea):
    - limited sentence support
    - relies on runtime linking (not good for MCU use)

1. [jons/libnmea](https://github.com/jons/libnmea):
    - complex call structure:
        ```c
        if (rxlen = read(fd, rxbuf, BUFSZ))
            if (nmea_concat(&nbuf, rxbuf, (size_t)rxlen))
                while (nmea_scan(&nbuf, &msg))
                {
                    evlen = nmea_parse(evbuf, MSGSZ, &msg);
        ```
    - relies on dual copies with `nmea_peek()` and `nmea_parse()`

1. [Paulxia/nmealib](https://github.com/Paulxia/nmealib):
    - extremely complex (14 separate header files!)
    - documentation very sparse: http://nmea.sourceforge.net/#gpsjet

1. [kosma/minmea](https://github.com/kosma/minmea);
Very good library, only a couple issues:
    - relies on `timegm` which is not available in all places,
        and the alternative `mktime`
        has been known to overflow on some architectures
    - limited frame support
    - interface is clunky: requires pushing/popping parsing structs
        on the stack of the caller
    - documentation limited: no Doxygen or man page docs,
        hard to grok function behavior without diving into the code

1. [canboat/canboat](https://github.com/canboat/canboat):
    - enormous and encyclopedic, also supports NMEA2000
    - not something I would try to get working on an MCU

1. [wdalmut/libgps](https://github.com/wdalmut/libgps):
    - interface gives no feedback about what is happening with parsed data

1. [mikalhart/TinyGPSPlus](https://github.com/mikalhart/TinyGPSPlus)
    - an excellent library in every way, gorgeous interface
    - not in pure C, very unfortunately

## Design

The interface for an ideal library might look something like:

```c
/* GPS statement type */
enum gps_type {
    GPS_INVALID    = 0,
    /* etc... */
    GPS_GLL,
    /* etc... */
    GPS_CHECKSUM_FAIL = UINT8_MAX
};

struct gps_stat_gll {
    uint16_t    lat;
    uint16_t    lon;
    uint32_t    lat_dec;
    uint32_t    lon_dec;
    uint8_t     lat_S:1;
    uint8_t     lon_W:1;
    uint8_t     status:1;
    uint8_t     faa_mode:1;
    uint8_t     hour;
    uint8_t     min;
    uint8_t     sec;
    /* NOTE variable sequence to allow compiler packing */
};

struct gps_sentence {
    enum gps_type   type:8;
    const char      *sentence;
union {
struct gps_stat_gll gll;
/* other sentence structs here */
};
};


/**
 * Read at most 'len' Bytes from 'buf', copying into internal parse buffer.
 *
 * Returns NULL if parsing is still in progress,
 * otherwise a pointer to a struct of parsed data.
 *
 * If 'parsed' is given, it will be populated with number of bytes parsed,
 * which may be less than 'len' if 'buf' contained multiple sentences.
 *
 * NOTE:
 * - returned struct may have 'type' set to GPS_INVALID or GPS_CHECKSUM_FAIL
 * - 'sentence' field of returned struct will point to a valid string
 *   containing what characters were parsed (for use in debugging or printing)
 */
struct gps_sentence *gps_parse(const char *buf, size_t len, size_t *parsed);


/* Optional use of floating point */
#if USE_FLOAT
struct gps_coord_double {
    double  lat;
    double  lon;
    uint8_t valid:1;
};

/**
 * If 'gss' points to a sentence type that contains valid coordinates,
 * returns coordinates in double format
 */
struct gps_coord_double gps_coord_double(struct gps_sentence *gss);
#endif


/* Use without floating point */
struct gps_coord_int {
    uint32_t    lat;
    uint32_t    lon:
    uint16_t    scaling:15; /* eg 1000 scaling for 4404.14036 lat = 4404140 */
    uint8_t     valid:1;
};

/**
 * If 'gss' points to a sentence type that contains valid coordinates,
 * returns coordinates in scaled integer format
 */
struct gps_coord_int gps_coord_int(struct gps_sentence *gss, uint16_t scaling);
```
