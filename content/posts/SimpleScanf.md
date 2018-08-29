---
title: "SimpleScanf"
date: 2018-08-20T19:39:13+05:30
draft: true
---


_**scanf()**_ is widely used in c to read inputs from standard input(**stdin**).
I wrote a simple scanf routine to understand how input can be read from **stdin**
just like `scanf()`. 
It is not optimized but works. This article documents the learning process.

To keep things simple, my simplescanf() routine handles only basic inputs:
```text
1. %d (integers)
2. %c (char)
3. %s (string)
```
It does not support assignment suppression character `*`, field width or any other features.

## Function Prototype

scanf()  takes variable number of arguments. We don't know how many variables we are going to read from stdin.
Hence, the prototype of simplescanf() also takes variable argument and is declared as
```c
int simplescanf(const char *fmt, ...)
```
It returns the number of arguments it read from stdin.

## Reading variable input 

varibale arguments in c can be read using **va_start()**, **va_arg()** and **va_end()** routines.
`man va_arg` gives the details regarding these rotuines. Below is a simple example with inline
comment to understand how to use variable arguments in c.
```c
//accepts variable argument
void foo(const char *fmt, ...)
{
        char *str;
        int *val;
        va_list ap;

        //intialize ap to process variable arguments
        //fmt is the last param before variable arg starts
        //fmt points to string "s d" or "s" or whatever string is
        //being passed.
        //ap is initialized to list of variable args (&buff, &var,)
        va_start(ap, fmt);

        while (*fmt) {
                switch(*fmt) {
                case 's':
                        //call to va_arg gives the type & value of next variable arg
                        //from ap list(&buff here)
                        str = va_arg(ap, char*);
                        printf("%s\n", str);
                        break;
                case 'd':
                        //call to va_arg gives the type & value of next variable arg
                        //from ap list(&var here)
                        val = va_arg(ap, int *);
                        printf("%d\n", *val);
                        break;
                }
                ++fmt;
        }
        //invalidate the ap list
        va_end(ap);

}
int main()
{
        char buff[10] = "hello";
        int var = 100;
        // call with 1 argument
	//fmt in foo() points to "s"
        foo("s", &buff);
        // call with 2 arguments,
	//fmt in foo() points to "s d"
        foo("s d", &buff, &var);

        return 0;
}
```
Above program will give the following output.
```text
//first call to foo()
hello
//second call to foo()
hello
100
```
## Implementation
The implementation uses **fgets()** to read the input from **stdin**. While the case for reading
a string or an integer is pretty straightforward, there is a case for handling char inputs which
is worth mentioning.
Consider the program below to read two chars from **stdin**
```c
int main()
{
        int i = 0;
        char ch;
        for (i = 0; i < 2; i++) {
                scanf("%c", &ch);
                printf("Input number[%d]: %c ", i, ch);
        }
        printf("\n");
        return 0;
}
```
On giving the below input to the above program, we can see
```text
Input:
 a b

Output:
Input number[0]: a Input number[1]:
```
the program does not print the second char `b` as expected. Well, it may appear
that the program has failed to read, it is not the case actually. If you notice
the input, there is a whitespace after `a` (between `a` & `b`). whitespace is also a char and this 
is what is read by our program. Since, we are only reading two chars,the loop 
terminates and the program ends.

If you want to read the second char `b` in this case or ignore whitespaces/newline
in general, you will have to add a whitespace before `%c` in your program as shown
below
```c
int main()
{
        int i = 0;
        char ch;
        for (i = 0; i < 2; i++) {
		//whitespace added before %c
                scanf(" %c", &ch);
                printf("Input number[%d]: %c ", i, ch);
        }
        printf("\n");
        return 0;
}
```
Now on giving the same input to the above program, second char is also printed.
```text
Input:
 a b

Output:
Input number[0]: a Input number[1]: b
```

In case of integer or string, this behaviour is not observed. By default, any whitespace
or newline given as input is ignored.

Keeping the above things in mind, we can now discuss the implementation of **simplescanf()**

1.We read from **stdin** using **fillbuff()** which in turn uses fgets(). 

2.`fmt` points to the string(e.g "%s %d %c") passed to **simplescanf()**.

3.skip any whitespace in `fmt` using **isspace()**. `space` variable is used to handle the case of char input
as discussed above.

```c
......
......
//fillbuff returns pointer to a buffer read through fgets()
bptr = fillbuff(buff);
while (*fmt) {
	len = 0;
	//skip any space chars in fmt
	while (*fmt && isspace(*fmt)) {
		++fmt;
		// useful to handle the case of char
		++space;
	}
.....
.....
```
 The flow for reading integer and string is similar. Below is the code for reading a string.
 ```c
 if (*fmt == 's') {
 	//check() returns first non-space character or \n
 	//by parsing buffer pointer bptr
	 bptr = check(bptr);
	 if (*bptr == '\n') {
	 //if we encounter newline, we again have to read from stdin. 
	 //remember how scanf() works when you press enter ? It waits until 
	 //you enter an input other than newline or space
		 bptr = fillbuff(buff);
		 continue;
	 }
	 // get the next arguments which is address of the variable where
	 //string is going to be saved
	 str = va_arg(ap, char *);
	 //get the length of chars we are going to read
	 len = strcspn(bptr, SPACE);
	 //copy the 'len' chars from bptr to the destination str
	 strncpy(str, bptr, len);
	 //move the buffer pointer to point to element after 'len' chars
	 bptr += len;
	 //increase the count by 1 as we have read an input
	 ++count;
	 //reset the space to 0. This is to check space for %c case
	 space = 0;
 }
```
The comments explain what is happening in the code. `scanf()` waits for an input
other than whitespace/newline in case of integers/string. Hence, we make sure that 
we are reading a non-sapce character from stdin in our case too.

The flow for reading char is similar to strings/integer, except for the special case 
of char(with space or without - discussed above).
```c
 else if (*fmt == 'c') {
 	//only one char is going to be read
	 len = 1;
	 //we check for any space before '%c'
	 //It will decide whether to ignore or read whitespace/newline
	 if (space > 0) {
	 	// if space is greater than zero, behaviour is similar to
		// reading integer/string. read again from stdin until a 
		//non-space char is entered.
		 bptr = check(bptr);
		 if (*bptr == '\n') {
			 bptr = fillbuff(buff);
			 continue;
		 }
	 }
	 //At this point we are done with the case where whitespace is 
	 //before %c. Now we just want to copy & save the char to our variable
	//get the variable - char read from stdin will be saved in it
	 str = va_arg(ap, char *);
	 //copy char to str
	 strncpy(str, bptr, len);
	 //if newline was entered, we need to check for any inputs entered
	 //after newline
	 if (*bptr == '\n') {
		 bptr = fillbuff(buff);
		 //bptr is updated
		 len = 0;
	 }
	 //if newline was not entered we adjust bptr by +1(one char read)
	 //if it was newline, bptr is already updated in the if conition
	 //above
	 bptr += len;
	 //increase count of input read
	 ++count;
	 //reset space to zero
	 space = 0;
 }
```
## Usage
`simplescanf()` can be used the same way one would use `scanf()`
```text
simplescanf(" %c %d %s", &ch, &var, &buff);
```

Code for `simplescanf()` can be found [here](https://github.com/kumarsourav/Simple-Scanf/blob/master/simplescanf.c)
