# Selective Coverage Instrumentation

* Input: a program with an annotated target location and relevant functions provided by Static Analyzer
* Output: instrumented program
* Exe: instrument only the relevant functions in the target program
* Purpose: enable DAFL to selectively receive coverage feedback only from the dependent parts of the program during the next fuzzing phase
