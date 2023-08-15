---
layout: post
title: "Cracking the Cracker: Ripping appart a blackhat tool"
date: 2021-11-7
---

Much of offensive security students and practitioners overlook the world that lies just below the scripts and the dreaded misconfigured permissions. You're probably already familiar with the famous Buffer Overflows - and overflows in general - and tired of frustratedly staring at the size of a buffer in GDB. Still, in the real world, not all exploitation consists of doing BoFs on a prepped Ubuntu, especially when the majority of the operating system market is governed by Microsoft.

In this post, I'm going to address some general and practical aspects of reversing, exploitation, and binary manipulation in Windows. While I've used Windows 11 to do this, Windows 10 will work exactly the same.

## Requirements

As a general rule, for working with Windows, I **highly recommend** the following tools (although I won't be using all of them):

- [ILSpy](https://github.com/icsharpcode/ILSpy)
- [dotPeek](https://www.jetbrains.com/es-es/decompiler/)
- [Cheat Engine](https://www.cheatengine.org/)
- A Linux VM or [WSL](https://docs.microsoft.com/es-es/windows/wsl/install) installed
- [Python3](https://www.python.org/downloads/)

Additionally, we'll need some experience with C, C#, and x86 and x64 assembly.

## Objective

I'll be "attacking" a brute-force tool that I've acquired. I'll be covering up some parts to avoid potential conflicts with the author of this tool. I also want to make it clear that I don't take responsibility for nor endorse the misuse of this tool.

![Picture](</assets/images/2021-11-07/(1).png>)

This tool tries combinations like `username:password` on a website to verify which credentials are valid. To avoid problems, it uses a list of proxies that can be switched. For ethical reasons, I've created my own list of proxies and set up a lab, thus avoiding attacking a real website.

We load a list of accounts and the list of proxies. Additionally, I've adjusted the number of threads to 300. But when we click that ominous "Start" button, it returns the following error:

![Picture](</assets/images/2021-11-07/(2).png>)

## Reversing and Exploitation

To set ourselves a goal, let's analyze how and why this window pops up. Before proceeding any further, we can take a look using ILSpy, as dnSpy is no longer maintained.

![Picture](</assets/images/2021-11-07/(3).png>)

We can distinguish 4 different DLL files within the same executable, but we only need to investigate the first one, as the other libraries are aesthetic. Furthermore, it's evident that the program is written in C#, which means the file is not exactly 32 or 64-bit assembly, but rather **Microsoft IL** (Intermediate Language), widely known as **MSCIL** (_Microsoft Common Intermediate Language_) or simply **CIL**, which is somewhat similar to Java bytecode. When a .NET program is compiled, it's converted into CIL, which despite being considered an assembly language, remains relatively high-level. This significantly facilitates its decompilation, but it eliminates the use of conventional static disassemblers like Ghidra or Cutter. From ILSpy, we can search for some keywords that appeared in the previous error to understand where to start:

![Picture](</assets/images/2021-11-07/(4).png>)

> If we click here, we can see the piece of code that interests us, but upon closer inspection, we realize that it doesn't actually do anything special.

```csharp
using System.Windows;

internal static class VerifyVersion
{
    internal static bool IsPaidVersion()
    {
        if (0 == 0)
        {
            Application.Current.Dispatcher.Invoke(() => MessageBox.Show("To access this feature, you need to purchase a paid version here: https://example.com \n\nPaid features:\n - Maximum of 999 threads (instead of 201);\n - All unlocked features;"));
            return false;
        }
        return true;
    }
}
```

But we know that this piece of code gets executed, so we can search for references to this function and see where it's used.

![Picture](</assets/images/2021-11-07/(5).png>)

This function is called in the classes `sxesac.Classes.CheckerHandler.VerifyUserOptions` and `sxesac.Classes.Helpers.ProxyDownloader+d__2.MoveNext`. Just by the names, we understand that the one we're interested in is the first one, not the second. Let's decompile the `VerifyUserOptions()` function to see how it works:

```csharp
using System.Windows;
using sxesac.Classes.Helpers;
using sxesac.Classes.Internal;
using sxesac.Helpers;

private bool VerifyUserOptions(int taskNumber)
{
    if (taskNumber == 0)
    {
        System.Windows.Application.Current.Dispatcher.Invoke(() => MessageBox.Show("How do you want to get work done if you don't have any employees? Gotta hire some, no?\nYou can't have 0 threads.\n\nTo continue, specify at least 1 thread.", "No threads", MessageBoxButton.OK, MessageBoxImage.Exclamation));
        return false;
    }
    if (LoadedAccountsCollection.Count == 0)
    {
        MessageBox.Show("Your accounts list is empty.\n\nTo continue, add an accounts list.", "Accounts list is empty", MessageBoxButton.OK, MessageBoxImage.Exclamation);
        return false;
    }
    if (taskNumber > LoadedAccountsCollection.Count)
    {
        int count = LoadedAccountsCollection.Count;
        MessageBox.Show($"XXXX lowered the threads number you specified ({taskNumber}) because you loaded {LoadedAccountsCollection.Count} accounts.\n\nXXXXX will continue checking accounts with {count} threads.", "You can't have more threads than accounts", MessageBoxButton.OK, MessageBoxImage.Exclamation);
        taskNumber = count;
        return true;
    }
    if (ProxyHelper.LoadedProxiesCollection.Count == 0)
    {
        MessageBox.Show("Your proxies list is empty.\n\nTo continue, add a proxy list.", "Proxy list is empty", MessageBoxButton.OK, MessageBoxImage.Exclamation);
        return false;
    }
    if (taskNumber > ProxyHelper.LoadedProxiesCollection.Count && ProxyModifier.proxyMethod != 0)
    {
        MessageBox.Show("Your thread number is bigger than the proxies you loaded.\n\nTo continue, lower your threads list or load more proxies using the Stack-Add feature.", "Threads are bigger than proxy number", MessageBoxButton.OK, MessageBoxImage.Exclamation);
        return false;
    }
    if (taskNumber > 201 && !VerifyVersion.IsPaidVersion())
    {
        return false;
    }
    Cleanup();
    isChecking = true;
    return true;
}
```

In the highlighted bold section, you can find the maximum number of threads allowed in the free version of the program along with a call to the function `sxesac.Classes.Internal.VerifyVersion.IsPaidVersion()`, which we encountered above. This seems like a clue that we're on the right track. However, if we look closely, we can see that all the conditionals end with `return false`, except for one:

```
if (taskNumber > LoadedAccountsCollection.Count)
{
    int count = LoadedAccountsCollection.Count;
    MessageBox.Show($"lowered the threads number you specified ({taskNumber}) because you loaded {LoadedAccountsCollection.Count} accounts.\n\nXXXXX will continue checking accounts with {count} threads.", "You can't have more threads than accounts", MessageBoxButton.OK, MessageBoxImage.Exclamation);
    taskNumber = count;
    return true;
}
```

It seems like something interesting. All the conditionals check if something is out of place and return `false` if something is wrong. In this part, that condition is not met, and we can exit the function early by returning true arbitrarily. Finally, we have found a bug.

Now let's open one of my favorite tools to take a closer look: Cheat Engine. We select the process and open it with the debugger:

![Picture](</assets/images/2021-11-07/(6).png>)

After opening the process and searching for the value 201, which is the number of threads, we get quite a few results. To filter them, we just need to change the value and scan again with the new value.

![Picture](</assets/images/2021-11-07/(7).png>)

And now we know where the thread count is located in memory. However, due to **ASLR** (Address Space Layout Randomization), this information won't be very useful the next time we open it with Cheat Engine. Let's find a way to overcome this using an option that CE provides to identify which parts of the code access that variable.

Please note that since **MSCIL is compiled at runtime**, what we'll be analyzing with CE from now on **will be proper assembly language**.

![Picture](</assets/images/2021-11-07/(8).png>)

Now, all we need is to have something interact with that variable, so let's click "Start" in the program and observe what accesses it.

![Picture](</assets/images/2021-11-07/(9).png>)

After clicking a few times to make sure that's the part we're interested in, we select it, and now we can see something more interesting: the variable is stored in `[rax+98]` (in hexadecimal notation). With Cheat Engine, we can inject our own code, but it's recommended to have experience with C and 32-bit and 64-bit assembly. Now let's open that part of the code:

![Picture](</assets/images/2021-11-07/(10).png>)

With `CTRL+A`, we open the CE's "Auto Assemble" module, where we can write our code. Starting from scratch on what we want to do now is extremely tedious, so let's go to `_Template > AOB Injection`. What this template does is select an AOB with the opcodes of the code portion where we want to inject. An AOB (Array Of Bytes) is, as the name suggests, a sequence of specific bytes.

CE will now ask us for the address where we want to inject and a name for the script. For the address, we'll leave it as it is. For the name, I recommend using a descriptive name **WITHOUT** spaces. Personally, I'll name it `_INJECT_`.

![Picture](</assets/images/2021-11-07/(11).png>)

What we want to do now is to save the memory address in a variable that we control, so let's create some memory space using the `alloc()` function.

```assembly
[ENABLE]

aobscan(INJECT,8B 80 98 00 00 00 5D) // should be unique
alloc(newmem,$1000,INJECT)
globalalloc(thread_count,4)

label(code)
label(return)

thread_count:
  dd 0

newmem:

code:
  mov [thread_count],rax
  mov eax,[rax+00000098]
  jmp return

INJECT:
  jmp newmem
  nop
return:
registersymbol(INJECT)

[DISABLE]

INJECT:
  db 8B 80 98 00 00 00

unregistersymbol(INJECT)
dealloc(newmem)
dealloc(thread_count)
```

I've highlighted the code that I added. The `globalalloc()` function reserves memory space for our variable and makes it globally accessible. It takes the arguments `thread_count` (what to name the variable) and `4` (the size in bytes). Then, I initialize the variable with `thread_count:` and set its value to 0 with `dd 0`. Later on, I save the value of rax in `thread_count`. After adding the script, select `File > Assign to current cheat table` so that we can use the script.

However, it's still not clear what this script does, right? What it does is store the address of `rax` (without adding the 98-byte difference) in the memory address pointed to by `thread_count`, **in other words, in our variable**. Still, if we don't access it, it won't be of much use. It's crucial to use `globalalloc()` instead of `alloc()` if we plan to access this variable manually, as **it makes it global and therefore accessible to the rest of the program and any added scripts**.

After assigning the script to our cheat table, it would be beneficial to give it a descriptive description and name.

![Picture](</assets/images/2021-11-07/(12).png>)

_Note: Some addresses might change due to restarting the program and Cheat Engine during certain checks. If there's something noticeable or worth mentioning, I'll explicitly mention it._

On the left, you can see checkboxes under the "Active" label. If we mark the checkbox next to a value or a variable, **we freeze it**, preventing it from changing its value. If we mark it on a script, the code in the section under the `[ENABLE]` label will be executed, and if we unmark it, the section under the `[DISABLE]` label will be executed in that script.

```assembly
[ENABLE]

aobscan(INJECT,8B 80 98 00 00 00 5D) // should be unique
alloc(newmem,$1000,INJECT)
globalalloc(thread_count,4)

label(code)
label(return)

thread_count:
  dd 0

newmem:

code:
  mov [thread_count],rax
  mov eax,[rax+00000098]
  jmp return

INJECT:
  jmp newmem
  nop
return:
registersymbol(INJECT)

[DISABLE]

INJECT:
  db 8B 80 98 00 00 00

unregistersymbol(INJECT)
dealloc(newmem)
dealloc(thread_count)

...
```

Now, let's see the value of our variable. To do this, let's select the option that says "Add address manually". **It's of vital importance to consider the following concepts**:

1. The value stored in `thread_count` is a **pointer**.
2. That pointer is for `rax`, **not for `rax+98`, which is the one we're interested in**.
3. To update the value, we'll have to **execute the script**.

So, how are we going to proceed? In the "Address" field, we'll type `[thread_count]+98`. By putting it in brackets, **we are treating it as a pointer and not a numerical value**. The `+98` following it is the offset to the value we want to see. At this point, this is equivalent to `[rax+00000098]`, but with variables we control.

![Picture](</assets/images/2021-11-07/(13).png>)

With that and a relatively decent description, it's ready. Even though it shows that **Cheat Engine currently has no idea what address we're telling it**, that's because we haven't executed the script yet, and it hasn't been reserved or initialized in memory. So, let's do that.

![Picture](</assets/images/2021-11-07/(14).png>)

_You can nest values and scripts by dragging them onto each other._

Now it's injected, but it hasn't been executed yet. The reason is that the part of the code where we injected it hasn't been executed yet. To make it execute in this case, **simply click "Start"**.

![Picture](</assets/images/2021-11-07/(15).png>)

_When clicking start with the script activated, our variable updates to the desired value._

This technique is extremely useful for efficiently debugging programs, and contrary to popular belief, **Cheat Engine isn't just for games**. Now that we have what we want, we can remove the first cell because it holds a static address, meaning the next time we start the program, it will look at the address it's currently assigned to, which means it won't give us the value we're looking for due to the aforementioned ASLR. On the other hand, the third cell is dynamic, meaning it will change and adapt the next time due to the script we created.

![Picture](</assets/images/2021-11-07/(16).png>)

_Even though it displays the same address, it's dynamic because its value is `[thread_count]+98`._

Now that our environment is set up, let's see if we can do something. First, let's save the cheat table so that we can open it later and verify that our script works flawlessly.

![Picture](</assets/images/2021-11-07/(17).png>)

*You can use* File > Save As *to save your cheat table for later.*

Now, we can close CE and the application in question, then reopen ILSpy for the part of the code with a bug.

```csharp
using System.Windows;
using sxesac.Classes.Helpers;
using sxesac.Classes.Internal;
using sxesac.Helpers;

private bool VerifyUserOptions(int taskNumber)
{
    if (taskNumber == 0)
    {
        System.Windows.Application.Current.Dispatcher.Invoke(() => MessageBox.Show("How do you want to get work done if you don't have any employees? Gotta hire some, no?\nYou can't have 0 threads.\n\nTo continue, specify at least 1 thread.", "No threads", MessageBoxButton.OK, MessageBoxImage.Exclamation));
        return false;
    }
    if (LoadedAccountsCollection.Count == 0)
    {
        MessageBox.Show("Your accounts list is empty.\n\nTo continue, add an accounts list.", "Accounts list is empty", MessageBoxButton.OK, MessageBoxImage.Exclamation);
        return false;
    }
    if (taskNumber > LoadedAccountsCollection.Count)
    {
        int count = LoadedAccountsCollection.Count;
        MessageBox.Show($"XXXXX lowered the threads number you specified ({taskNumber}) because you loaded {LoadedAccountsCollection.Count} accounts.\n\nXXXX will continue checking accounts with {count} threads.", "You can't have more threads than accounts", MessageBoxButton.OK, MessageBoxImage.Exclamation);
        taskNumber = count;
        return true;
    }
    if (ProxyHelper.LoadedProxiesCollection.Count == 0)
    {
        MessageBox.Show("Your proxies list is empty.\n\nTo continue, add a proxy list.", "Proxy list is empty", MessageBoxButton.OK, MessageBoxImage.Exclamation);
        return false;
    }
    if (taskNumber > ProxyHelper.LoadedProxiesCollection.Count && ProxyModifier.proxyMethod != 0)
    {
        MessageBox.Show("Your thread number is bigger than the proxies you loaded.\n\nTo continue, lower your threads list or load more proxies using the Stack-Add feature.", "Threads are bigger than proxy number", MessageBoxButton.OK, MessageBoxImage.Exclamation);
        return false;
    }
    if (taskNumber > 201 && !VerifyVersion.IsPaidVersion())
    {
        return false;
    }
    Cleanup();
    isChecking = true;
    return true;
}
```

I've highlighted the bug in bold. This bug occurs when the number of threads is greater than the number of loaded accounts. If this happens, the program reduces the number of threads to match the number of loaded accounts and exits the function early, returning a `true`. The code that comes after wouldn't be executed, making it similar to a broken access control or _Broken Access Control_ issue. Let's put it into practice to see if it works. To do this, we'll open the program alongside Cheat Engine to see if it's true that this number of threads gets reduced.

![Picture](</assets/images/2021-11-07/(18).png>)

_Note that in the second cell, we no longer have an address, but the value that will be considered._

To have everything ready, we just need to activate the script and click "Start" in the program. After that, we can load our combo list and some placeholder proxies. Then, all we need to do is set the number of threads to be greater than the number of loaded accounts.

![Picture](</assets/images/2021-11-07/(19).png>)

_Everything is ready to exploit the bug._

Now, click "Start". What we expect now is for the number of threads to change in CE and to be greater than the 201-thread limit.

![Picture](</assets/images/2021-11-07/(20).png>)

_That's the warning we were waiting for._

![Picture](</assets/images/2021-11-07/(21).png>)

_Proof that the number of threads has indeed changed, as we expected._

Here, we can see that we get the expected warning. I clicked "OK" and left it running for a few seconds to verify that the program doesn't crash or become corrupted, and then I clicked the "Stop" button to halt it. In the second image, we can see that the number has changed exactly to the number of loaded combos, and it's greater than the 201-thread limit. Finally, we've analyzed the code, conducted some tests, set up a simple analysis environment, and furthermore, found and exploited a _Broken Access Control_ vulnerability.

## Conclusion

While investigating programs and developing certain exploits can be extremely intimidating, **with the right tools, patience, and a willingness to learn, anything is achievable**. Before discovering this program, I had very little knowledge about exploitation and working in Windows environments, as I had mainly focused on Linux/Unix-based ones. I've learned much more than I thought, and I've gained an understanding of how a significant portion of the _.NET Core_ works, demonstrating that the ambition to learn is everything in this field.

As a final note, this post was written in Spanish by me. This post was originally uploaded at [AsturHackers' Website](https://blog.asturhackers.es/crackeando-el-cracker-una-vision-practica-del-reversing-en-windows)
