# Innodb

### Variable

* page_size

  ```mysql
  show global status like "Innodb_page_size"   //16384 byte = 16kb
  ```

### structure

#### page

| name               | size (byte) | desc                               |
| ------------------ | ----------- | ---------------------------------- |
| File Header        | 38          | common info of page                |
| Page Header        | 56          | info of data page                  |
| Infimum + Supremun | 26          | smallest and biggest record        |
| User Record        | not sure    | Actually used line records         |
| Free Space         | not sure    | Unused space size of the page      |
| Page Directory     | not sure    | index of record                    |
| File Trailer       | 8           | Check the completeness of the page |

#### line record

COMPACT

* Variable length field length list

  varchar text blob varbinary

* NULL value column

  Unified manage the columns that can be NULL, If there is no column that allows NULL to be stored in the table, the list of NULL values does not exist either

* Record header information

  | name         | size (bit) | desc                         |
  | ------------ | ---------- | ---------------------------- |
  | reverse      | 1          | not use                      |
  | reverse      | 1          | not use                      |
  | delete_mase  | 1          | mark the record delet or not |
  | min_rec_mask | 1          |                              |
  |              |            |                              |
  |              |            |                              |
  |              |            |                              |
  |              |            |                              |

  