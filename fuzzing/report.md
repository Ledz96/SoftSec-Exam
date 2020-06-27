# Setup and Methodology

The goal of the first section of the report is to describe the methodology used for fuzzing the program.

## Design of the fuzzer stub

The fuzzing stub is a fundamental part of the fuzzing process, as it helps generating inputs that might be optimal for crashing the program.

The randomly generated data is first of all put through a test: if the size is too small, the fuzzer returns correctly. This is done in order to have large enough inputs, with a REGEX_LEN (the size of a regex) hardcoded and changed between different runs. As small inputs will always return with no mistakes, the fuzzer will smartly increase the input size.

As the fuzzer automatically tweaks bytes in the input files (starting from the seeds), including a separator character between regex and text was not deemed too much of an effective design choice. As a consequence, it was rather chosen to separate the first REGEX_LEN bytes from the rest, using the first block to form the regex and the second one to form the text to analyze. As the behavior of the program with different sized regex is interesting, multiple runs were performed tweaking the value of the parameter.

The separated parts of the data are then put into two null terminated strings, and the regex is provided along with its length as an argument to the parse() function.

An error control is then performed in order not to provide a NULL input to the SearchExpression constructor; in case an error is present, the program simply returns 0, as no crash happened and the error was correctly handled.

Finally, the method search of the searchExp object is invoked.

If the program terminates correctly, the allocated variables are freed, in order to avoid stub-related memory leaks.


## Choice of seeds 

In order to try and analyse as much code as possible, many different sets of inputs have been provided as seeds.

A first set of seeds has been built by constructing valid regexes (aimed at recognizing elements such as email addresses or websites, repeated words etc.) of varying size (approximately 8 to 500 characters, with some regexes being repeated at times), followed by text of various kind and size (documents written in English, HTML from web pages, lists of names or domains etc.).

A second set of seeds has been built by randomly generating bytes and inserting them into a few different files.

A third set of seeds has been built by generating a high number of regex symbols, either by themselves or combined randomly, both with and without a text following.

## Choice of sanitizers

At the beginning of the fuzzing campaign, only the ASAN provided with '-fsanitize=fuzzer,address' was included. 

Following this, after detecting a possible unsigend integer overflow when inspecting the code to find the reason for a stack overflow, I added the options 'unsigned-integer-overflow' and -fsanitize-trap=all, aimed at making the fuzzer stop the program in case of a detected overflow.

## Choice of fuzzers

As the number of bugs found with libfuzzer has been sufficient, no use of ulterior fuzzers was deemed necessary.

## Campaign setup

When it wasn't left running, the fuzzer was launched with the following command, aiming at multiple "short" runs:
```
./build/bin/fuzz -jobs=8 -runs=16384 -workers=8 ../seeds -max_len=32768
```
Number of runs and maximum length of the file were also often tinkered with, depending on the minimum length allowed in the stub.

# Bug reports

In this second section, an individual report will be provided for each bug found.

Each report will contain a small description of the bug, a reference to the file necessary to trigger it, and one or more suggested solution (along with a comment on the one I personally adopted, in case many are provided).

# Bug 1: Heap use after free


## Description (2 points)
The delete performed at parse.cpp:239:9 is sometimes being called on an already deleted pointer, leading to a heap use after free that crashes the program.

When the user provides a regex pattern containing an error after a certain number of allowed characters (for example, an umbalanced closed parenthesis, as in the input presented), the program jumps to an 'error' label, where, after checking they are not nullptr, it first deletes the pointer 'head', then 'next'. The way destructors are implemented, however, leads to the destruction not only of head, but also of all the objects of the linked list. Therefore, if 'next' still points to an object that has been added to the linked list in the previous step, it will be first deleted in the call to head's destructor, leading to an error when the second delete is called.

The bug is detected by the address sanitizer.

## Proof of Concept (1 point)

crash-e4166adfbaba8f465b59d2ae4e5300e99595be04


## Suggested Fix (1 point)
Multiple solutions may be adopted in order to solve the problem.

A first approach might be to only delete the 'next' pointer in case its value might have changed from the previous step, and only delete head otherwise. On a more careful analysis, we may however infer that, with the current implementation, there is no way that 'next' may change value right before an error.

A second approach, based on what we stated previously, could be to just avoid deleting the object pointed by 'next': in fact, with the current implementation, it will always be deleted along with the delete of 'head'.

The proposed solution, however, is more simple: by setting next to nullptr right after it is assigned or added to the head pointer, the following check will always state that next == nullptr, and the delete will not be performed.



# Bug 2: Stack overflow


## Description (2 points)
The _parse_internal() function implemented in parse.cpp:110 calls itself recursively in different instances: an overwhelming number of recursive calls can completely fill the stack, leading to a stack overflow.

When a '(' character is provided as part of a regex, a recursive call is performed in parse.cpp:120.
As a consequence, if an input with a high enough number of open parentheses is provided, a stack overflow will happen, causing the program to crash.

The bug is detected by the address sanitizer.

## Proof of Concept (1 point)

crash-3169a588e3f4ef1b1dcf9159339608ece91fbc3f 

Note: in order to reproduce this bug, a large enough REGEX_LEN is necessary. A secondary fuzz_long.cpp file has been provided for this purpose.


## Suggested Fix (1 point)
Multiple solutions may be adopted in order to solve the problem.

In the stub, it is possible to limit the size of the provided regex. This solution, however, does not fix the problem itself, but rather avoids triggering it, and is therefore not desirable.

As a 'depth' parameter is already present in the _parse_internal() funtion, the easiest and preferred solution is to only allow recursion to reach up to a certain depth, and trigger an error in case said depth is passed.



# Bug 3: Stack overflow


## Description (2 points)
The destructor of the class RegularExpression is defined recursively: in RegularExpression.cpp:20, a delete for the object referred to in the 'next_' attribute of the current object is called.  
In case a long linked list is defined, this may lead to stack overflow.

When a repeat N expression is created in parse.cpp:96, the number of nodes is defined by an integer specified in the regex. In case the integer is too big, too big of a linked list will be created.  
This will then be cause of stack overflow when a delete is called on the node.

The bug is detected by the address sanitizer.

## Proof of Concept (1 point)

crash-1f104b58146a321169db2d32902fdf628dc54f34


## Suggested Fix (1 point)
In order to fix the bug, it is enough to only allow numbers up to a certain threshold for the size of the linked list in the repeatNExpression, causing an error in case a number bigger than the threshold is provided.



# Bug 4: Unsigned integer overflow


## Description (2 points)
In the function try_parse_range, implemented starting from parse.cpp:21, an unsigned integer may be calculated, based on the user input. This number might overflow, leading to an unsigned integer overflow.

In the line parse.cpp:59, the value of max is multiplied by 10 and added to a digit between 0 and 9, based on the input. With a very large number following a comma, 'max' will overflow.

## Proof of Concept (1 point)

crash-9e813a45eb8dee409543e0947b697da45129a547 


## Suggested Fix (1 point)
In order to fix the bug, it is enough to perform a check before increasing the value of 'max', to make sure that the resulting value is still lower than the maximum unsigned int. 



# Bug 5: Unsigned integer overflow


## Description (2 points)
In the function try_parse_range, implemented starting from parse.cpp:21, an unsigned integer may be calculated, based on the user input. This number might overflow, leading to an unsigned integer overflow.

In the line parse.cpp:45, the value of min is multiplied by 10 and added to a digit between 0 and 9, based on the input. With a very large number, 'min' will overflow.

## Proof of Concept (1 point)

crash-019fc60acef434ad658e7a59c5feb2d8348e06e9


## Suggested Fix (1 point)
In order to fix the bug, it is enough to perform a check before increasing the value of 'min', to make sure that the resulting value is still lower than the maximum unsigned int. 
