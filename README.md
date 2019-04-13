
# Introduction

dcc compiles C programs using clang and adds explanations suitable for novice programmers
to compiler messages novice programmers are likely to encounter and not understand.

For example:

    $ dcc a.c
    a.c:3:15: warning: address of stack memory associated with local variable 'counter' returned [-Wreturn-stack-address]
            return &counter;

    dcc explanation: you are trying to return a pointer to the local variable 'counter'.
      You can not do this because counter will not exist after the function returns.
      See more information here: https://comp1511unsw.github.io/dcc/stack_use_after_return.html


dcc adds code to the binary which detects run-time errors and prints information
likely to be helpful to novice programmers, including
printing values of variable in lines used near where the run-time error occurred.

For example:

    $ dcc buffer_overflow.c
    $ ./a.out
    a.c:6:3: runtime error: index 10 out of bounds for type 'int [10]'
    
    Execution stopped here in main() in buffer_overflow.c at line 6:
    
        int a[10];
        for (int i = 0; i <= 10; i++) {
    -->     a[i] = i * i;
        }
    }
    
    Values when execution stopped:
    
    a = {0, 1, 4, 9, 16, 25, 36, 49, 64, 81}
    i = 10

dcc can alternatively embed code to detect use of uninitialized variables
and print a message a novice programmer can hopefully understand. For example:

    $ dcc --memory uninitialized.c
    $ ./a.out
    uninitialized.c:6 runtime error uninitialized variable used
    
    Execution stopped here in main() in uninitialized.c at line 6:

        int a[1000];
        a[42] = 42;
    -->    if (a[argc]) {
        a[43] = 43;
    }

    Values when execution stopped:

    argc = 1
    a[42] = 42
    a[43] = -1094795586 <-- warning appears to be uninitialized value
    a[argc] = -1094795586 <-- warning appears to be uninitialized value

# Valgrind

dcc can alternatively embed code in the binary to run valgrind instead of the binary:

    $ dcc --valgrind buffer_overflow.c
    $ ./a.out
    Runtime error: uninitialized variable accessed.
    
    Execution stopped here in main() in uninitialized-array-element.c at line 6:

        int a[1000];
        a[42] = 42;
    -->    if (a[argc]) {
        a[43] = 43;
    }

    Values when execution stopped:

    argc = 1
    a[42] = 42
    a[43] = 0
    a[argc] = 0

valgrind is slower but more comprehensive in its detection of uninitialized variables than MemorySanitizer.

# Leak checking

dcc can also embed code to check for memory-leaks:

    $ dcc --valgrind --leak-check leak.c
    $ ./a.out
    Error: free not called for memory allocated with malloc in function main in leak.c at line 3.

This option can not also be used for (the default) Address sanitizer but error are not intercepted and may be  cryptic
for novice programmers.

# Local Variable Use After Function Return Detection

    $ dcc --use_after_return bad_function.c
    $ ./a.out
	bad_function.c:22 runtime error - stack use after return
	
	dcc explanation: You have used a pointer to a local variable that no longer exists.
	  When a function returns its local variables are destroyed.
	
	For more information see: https://comp1511unsw.github.io/dcc//stack_use_after_return.html
	Execution stopped here in main() in bad_function at line 22:
	
	
		int *a = f(42);
	-->	printf("%d\n", a[0]);
	}


valgrind also usually detect this type of error, e.g.:

    $ dcc --use_after_return bad_function.c
    $ ./a.out
	Runtime error: access to function variables after function has returned
	You have used a pointer to a local variable that no longer exists.
	When a function returns its local variables are destroyed.
	
	For more information see: https://comp1511unsw.github.io/dcc//stack_use_after_return.html'
	
	
	Execution stopped here in main() in tests/run_time/bad_function.c at line 22:
	
	
	int main(void) {
	-->	printf("%d\n", *f(50));
	}


# Run-time Error Handling Implementation

* dcc by default enables clang's  AddressSanitizer (`-fsanitize=address`) and UndefinedBehaviorSanitizer (`-fsanitize=undefined`) extensions.

* dcc embeds in the binary produced a xz-compressed tar file (see [compile.py]) containing the C source files for the program and some Python code which is executed if a runtime error occurs.

* Sanitizer errors are intercepted by a shim for the function `__asan_on_error` in [main_wrapper.c].

* A set of signals produced by runtime errors are trapped by `_signal_handler` in [main_wrapper.c].

* Both functions call `_explain_error` in [main_wrapper.c] which creates a temporary directory,
extracts into it the program source and Python from the embedded tar file, and executes the Python code, which:

    * runs the Python ([start_gdb.py]) to print an error message that a novice programmer will understand, then

    * starts gdb, and uses it to print current values of variables used in source lines near where the error occurred.

# Dirtying Stack Pages to Facilitate Uninitialized Variable Detection

Linux initializes stack pages to zero.  As a consequence novice programmers  writing small programs with few function calls
are likely to find zero in uninitialized local variables.  This often results in apparently correct behaviour from a
invalid program with uninitialized local variables.

dcc embeds code in the binary which initializes the first few megabytes of the stack to 0xbe (see `clear-stack` in [main_wrapper.c].

When printing variable values, dcc warns the user if a variable looks to consist of 0xbe bytes, and thus is likely uninitialized.

# Build Instructions

    $ git clone https://github.com/COMP1511UNSW/dcc
    $ cd dcc
    $ make
    $ cp -p ./dcc /usr/local/bin/dcc
 
# Release instruction

	$ ./create_github_release.py 1.0 'Initial github release of dcc'
   
# Dependencies

clang, python3, gdb, valgrind 

# Author

Andrew Taylor (andrewt@unsw.edu.au)

Except help_cs50.py  is almost entirely from  https://github.com/cs50/help50-server/blob/master/helpers/clang.py

# License

GPLv3
