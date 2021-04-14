# Pattern

|   Pattern   |                         Description                          |
| :---------: | :----------------------------------------------------------: |
|      *      |                Match zero or more characters                 |
|      ?      |                  Match any single character                  |
|    [...]    |             Match any of the characters in a set             |
| ?(patterns) |   Match zero or one occurrences of the patterns (extglob)    |
| *(patterns) |   Match zero or more occurrences of the patterns (extglob)   |
| +(patterns) |   Match one or more occurrences of the patterns (extglob)    |
| @(patterns) |        Match one occurrence of the patterns (extglob)        |
| !(patterns) | Match anything that doesn't match one of the patterns (extglob) |