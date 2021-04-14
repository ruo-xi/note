# String

## structure

```c
typedef struct redisObject {
    
    
    unsigned type:4;
    
    #define OBJ_ENCODING_INT 1     /* Encoded as integer */
	#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
	#define OBJ_ENCODING_RAW 0     /* Raw representation */
    unsigned encoding:4;
    unsigned lru:LRU_BITS;//24bit/3byte
    						/* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;







```



### method