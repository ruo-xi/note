# Storage archtecture

## File

### Disk

* Addressing    ms
* bandwidth     G/M

### Memory

* Addressing   ns
* bandwidth   ????

### I/O buffer

disk,track(磁道),sector(扇区  512 byte): need index

os default read 4k from disk;

## Relational database

row-level storage

### base disk

#### example:

* mysql
* oracle

#### data page 

* capacity:       multiples of 4k

#### index:            

* capacity:        multiples of 4k

### base memory

#### example

* HANA

#### feature

* The same data in the memory occupies less capacity than the disk capacity

## Cache

### example

* redis
* memchched

### infrastructure

* 



