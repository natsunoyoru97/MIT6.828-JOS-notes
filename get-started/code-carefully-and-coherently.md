# Code Carefully And Coherently

Not only do we have to write code that works, but also we have to consider **all the corner cases** .

Also note **JOS CODING STANDARDS**, the ``CODING`` file in the ``lab`` folder:

> * No space after the name of a function in a call <br>
>    For example, ``printf("hello")`` not ``printf ("hello")``.
> 
> * One space after keywords "if", "for", "while", "switch". <br>
>   For example, ``if (x)`` not ``if(x)``.
> 
> * Space before braces. <br>
>  For example, ``if (x) {`` not ``if (x){``.
> * Function names are all lower-case separated by underscores.
> * Beginning-of-line indentation via tabs, not spaces.
> * **Preprocessor macros are always UPPERCASE.** <br>
>  There are a few grandfathered exceptions: assert, panic, static_assert, offsetof.
> * Pointer types have spaces: ``(uint16_t *)`` not ``(uint16_t*)``.
> * Multi-word names are lower_case_with_underscores.
> * Comments in imported code are usually C ``/* ... */`` comments. <br>
>  Comments in new code are C++ style //.
> * In a function definition, the function name starts a new line. <br>
>  Then you can ``grep -n '^foo' */*.c`` to find the definition of foo.
> * Functions that take no arguments are declared ``f(void)`` not ``f()``.

Code given in ALL labs will be consistent with the coding standards above.