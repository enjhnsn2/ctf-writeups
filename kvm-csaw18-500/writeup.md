We're given a 64 bit elf. When run, it doesn't really do much but take input from stdin, but if you ltrace it, you can see that it attempts to access /dev/kvm. For anyone not familiar, kvm is the builtin hypervisor in linux (kernel virtual machine), so the binary is very likely spinning up a vm. After some examination, you can see that the binary creates a virtual machine and a vcpu, then copies 0x1380 bytes into the vm. 
![](copied_region.png?raw=true)  
We then dd this code into a second file for examination. The rest of the outer binary appears to pass stdin/stdout through to the vm, and does *something* when the vm halts (then reruns the vm). The inner binary is obviously an executable, but both IDA and Binary ninja botch the dissassembly. After some examining of the hex dump in binary ninja, we can recover a few more function fragments.
![](broken_cfg.png?raw=true)
![](broken_cfg2.png?raw=true)

We see that the function fragments often end in setting rax to a magic value, then halting the vm. After some examination of the outer binary, it appears that when the vm halts, it reads in the magic value in rax, does a lookup in a static table, then changes the inner binary's rip to the value found in the table.
In this way, the inner binary is able to push its control flow through the outer binary, obfuscating its execution. After manually deobfuscating the control flow of the inner binary by changing the halts to the corresponding jumps, the binary becomes much simpler; stage 1 complete. (It is worthwhile to note how easy Binary Ninja made this pathcing process with its patch->assemble function that allows you to directly insert assembly instructions into the binary).  

We now see that it is comparing our input string against a constant string in memory after performing some bit shifting.   
Its keygen time!
We see that it is performing lookups in a constant data structure where each item seems to have 3 fields, and two variants of the data structure  
![](bin_tree.png?raw=true)  
Variant  1:  
Field 1: ff  
Field 2: ptr into data structure  
Field 3: ptr into data structure  

Variant 2:  
Field 1: character  
Field 2: null  
Field 3: null  

With a bit of thinking, we realize this is a binary tree, which makes this function a preorder traversal.
![](preorder_search.png?raw=true)  

We can see that each character in our input string is being searched for in this tree, and some bits are being stored to a result that is then compared against the magic constant string to see if we win. We discover that another one of the functions just adds a 1 or 0 to our result bitstring, which means that for any given character in our input string, its mapping becomes the path needed to take in the binary tree to find the character, 0 for left, 1 for right; kind of like a bad huffman encoding. After building up a map between input characters and their resulting bit strings, we bruteforce decoded all possible strings in the data section of the binary.
This gives us the flag!  
flag{who would win? 1000 ctf teams or 1 obfuscat3d boi?}



