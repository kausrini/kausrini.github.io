---
layout: post
title: Buffer Security Check - Dissassembly Explained
subtitle: Security Cookie Initialization
cover-img: /assets/img/2022_11_02/SecurityInitCookieC.jpg
thumbnail-img: /assets/img/2022_11_02/SecurityInitCookieCThumb.jpg
share-img: /assets/img/2022_11_02/SecurityInitCookieC.jpg
tags: [malware, analysis, buffer overflow, stack cookie, "0xBB40E64E"]
---

One of the common techniques during malware analysis is to identify interesting API calls and understanding how the malware uses them. While analyzing a sample [1], I came across the following function calls embedded as strings.

```
GetCurrentProcess
GetCurrentProcessId
GetCurrentThreadId

GetSystemTimeAsFileTime
QueryPerformanceCounter
GetTickCount
```

Obviously, I was excited when I saw these since the second half of the above API queries are often used as Anti-Debugging API calls and the first half is used for various purposes including process injection (create a suspended thread within the current process and inject code into it etc.).

But this was not the case. The above API calls were made by a module that was inserted into the code by the compiler (like MSVC). I was not the first person to be disappointed by this as you can see StackExchange posts [2] from 2015, jumping to the same conclusions as I did.

So, it is useful to recognize such compiler inserted code stubs while reversing a malware, to speed up your analysis and ignore API calls that are not relevant. In this post, I am going to discuss about this compiler inserted stub, how it looks and briefly provide an overview on its purpose.

# Disassembly
![__security_init_cookie](/assets/img/2022_11_02/SecurityInitCookie.jpg){: .mx-auto.d-block :}
<center><em>Figure 1: __security_init_cookie</em></center>

As you can see in the above disassembly, the function FUN_00417ac0 does the following
1. Checks the value of DAT_004d26d4 against a default value 0xBB40E64E.
2. Generates a "random" cookie by XOR'ing multiple variables.
3. Replaces the value of DAT_004d26d4 with the generated random cookie.

Random Cookie = `System Time ^ Current Process Id ^ Current Thread Id ^ Tick Count ^ Performance Counter`

# Theory

The above generated Random Cookie is also called a "Stack Cookie" or "Security Cookie" used to detect buffer overrun/overflow in a stack. Since this is a well discussed topic, I'll keep this short and provide a brief overview.

When a function is called (assuming stdcall - standard calling convention for windows API), 

1. The function parameters are pushed into the stack from right to left. 
2. The current Instruction Pointer (EIP) is pushed to the stack. This will be the "Return Address" for the function when the stack unwinds.
3. Current stack base pointer (EBP) is pushed into the stack
4. Space is allocated for local variables by increasing the stack size.

I have left out various steps in the function call since they are not relevant to the topic of discussion. At the end of above, the stack will look like the below image (from Practical Malware Analysis)

![Individual Stack Frame](/assets/img/2022_11_02/IndividualStackFrame.jpg){: .mx-auto.d-block :}
<center><em>Figure 2: Individual Stack Frame</em></center>

As you can see, the stack frames grow "downwards" or in other words, the base of the stack is always at higher memory address than the top of the stack. 

As a result, when a "Local Variable N" overflows in the stack, it will overwrite data in the previous memory locations. In this example, it would overwrite the following in this order,
1. All local variables from Local Variable N, till Local Variable 1. 
2. Base Pointer - Old EBP stored in the stack frame. 
3. Instruction Pointer - EIP stored in the stack frame. 
4. Arguments pushed into the stack before the function call. 
.. and so on. 

The goal of an attacker would be to overwrite the EIP, pointing it to their own malicious code in memory. This way, when the vulnerable function returns, it will jump to the malicious code instead thereby exploiting a buffer overflow vulnerability. 

Stack Cookies are used to prevent this issue. The compiler creates code to initialize a stack cookie (DAT_004d26d4 from Fig 1) and inserts code stubs in a function prologue and epilogue in the following manner. 

The function prologue would include a new step that pushes the initialized stack cookie between EBP and local variables. The function epilogue would include a step right before it returns to the EIP. This step would check if the cookie value was unmodified.

If there was a buffer overflow in this function, the cookie value would have been overwritten and the compiler inserted stub in the function epilogue would exit the process instead of returning to the next instruction.

The cookie value is randomly generated by the compiler in this case to prevent the attacker from guessing the value while overwriting it. Please refer to Microsoft's documentation regarding compiler options to enable this security cookie [3][4].


# References
1. [https://www.virustotal.com/gui/file/5d48f8503446780ca198fb5a5c7ccebc7a8ca729f9a0cad78a2fc5f381500d13](https://www.virustotal.com/gui/file/5d48f8503446780ca198fb5a5c7ccebc7a8ca729f9a0cad78a2fc5f381500d13)
2. [https://reverseengineering.stackexchange.com/questions/6879/defeat-queryperformancecounter-as-anti-debugging-trick/6880](https://reverseengineering.stackexchange.com/questions/6879/defeat-queryperformancecounter-as-anti-debugging-trick/6880)
3. [https://learn.microsoft.com/en-us/cpp/build/reference/gs-buffer-security-check?view=msvc-160#gs-buffers](https://learn.microsoft.com/en-us/cpp/build/reference/gs-buffer-security-check?view=msvc-160#gs-buffers)
4. [https://learn.microsoft.com/en-us/cpp/c-runtime-library/reference/security-init-cookie?view=msvc-170](https://learn.microsoft.com/en-us/cpp/c-runtime-library/reference/security-init-cookie?view=msvc-170)
