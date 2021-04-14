# Ziplist

## structure

```C
// "src/ziplist.c"

#define ZIPLIST_HEADER_SIZE     (sizeof(uint32_t)*2+sizeof(uint16_t))
#define ZIPLIST_END_SIZE        (sizeof(uint8_t))
// zl_bytes  zl_tail_offset  zl_length   entrys 		zl_end(255   0xff)
// 32byte    32byte			 16byte 	 ....			8byte

unsigned char *ziplistNew(void) {
    unsigned int bytes = ZIPLIST_HEADER_SIZE+ZIPLIST_END_SIZE;
    unsigned char *zl = zmalloc(bytes);
    ZIPLIST_BYTES(zl) = intrev32ifbe(bytes);
    ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(ZIPLIST_HEADER_SIZE);
    ZIPLIST_LENGTH(zl) = 0;
    zl[bytes-1] = ZIP_END;
    return zl;
}

// <prevlen>                <encoding>                <entry-data>

// prevlen 
// 0-253(1byte)/254(5byte)    

//cncoding
#define ZIP_STR_MASK 0xc0
#define ZIP_INT_MASK 0x30

#define ZIP_STR_06B (0 << 6)
/*  |00pppppp| - 1 byte
       String value with length less than or equal to 63 bytes (6 bits).
       "pppppp" represents the unsigned 6 bit length.*/
#define ZIP_STR_14B (1 << 6)
/* |01pppppp|qqqqqqqq| - 2 bytes
       String value with length less than or equal to 16383 bytes (14 bits).
       IMPORTANT: The 14 bit number is stored in big endian.*/
#define ZIP_STR_32B (2 << 6)
/*  |10000000|qqqqqqqq|rrrrrrrr|ssssssss|tttttttt| - 5 bytes
       String value with length greater than or equal to 16384 bytes.
       Only the 4 bytes following the first byte represents the length
       up to 2^32-1. The 6 lower bits of the first byte are not used and
       are set to zero.
       IMPORTANT: The 32 bit number is stored in big endian.*/
#define ZIP_INT_16B (0xc0 | 0<<4)
/*  |11000000| - 3 bytes
       Integer encoded as int16_t (2 bytes).*/
#define ZIP_INT_32B (0xc0 | 1<<4)
/*  |11010000| - 5 bytes
       Integer encoded as int32_t (4 bytes).*/
#define ZIP_INT_64B (0xc0 | 2<<4)
/*  |11100000| - 9 bytes
       Integer encoded as int64_t (8 bytes).*/
#define ZIP_INT_24B (0xc0 | 3<<4)
/*  |11110000| - 4 bytes
       Integer encoded as 24 bit signed (3 bytes).*/
#define ZIP_INT_8B 0xfe
/*  |11111110| - 2 bytes
       Integer encoded as 8 bit signed (1 byte).*/
#define ZIP_INT_IMM_MASK 0x0f   /* Mask to extract the 4 bits value. To add
                                   one is needed to reconstruct the value. */
/*  |1111xxxx| - (with xxxx between 0001 and 1101) immediate 4 bit integer.
       Unsigned integer from 0 to 12. The encoded value is actually from
       1 to 13 because 0000 and 1111 can not be used, so 1 should be
       subtracted from the encoded 4 bit value to obtain the right value.*/
  |11111111| - End of ziplist special entry.

typedef struct zlentry {
    unsigned int prevrawlensize; /* Bytes used to encode the previous entry len*/
    unsigned int prevrawlen;     /* Previous entry len. */
    unsigned int lensize;        /* Bytes used to encode this entry type/len.
                                    For example strings have a 1, 2 or 5 bytes
                                    header. Integers always use a single byte.*/
    unsigned int len;            /* Bytes used to represent the actual entry.
                                    For strings this is just the string length
                                    while for integers it is 1, 2, 3, 4, 8 or
                                    0 (for 4 bit immediate) depending on the
                                    number range. */
    unsigned int headersize;     /* prevrawlensize + lensize. */
    unsigned char encoding;      /* Set to ZIP_STR_* or ZIP_INT_* depending on
                                    the entry encoding. However for 4 bits
                                    immediate integers this can assume a range
                                    of values and must be range-checked. */
    unsigned char *p;            /* Pointer to the very start of the entry, that
                                    is, this points to prev-entry-len field. */
} zlentry;



```





