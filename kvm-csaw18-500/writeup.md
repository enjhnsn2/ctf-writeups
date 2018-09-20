Played with sigpwny

we're given a 64 bit elf. When run, it doesn't really do much, but if you ltrace you can see that it attempts to access /dev/kvm.
For anyone not familiar, kvm is the builtin hypervisor capabilities in linux (kernel virtual machine).

You can see that the binary creates a virtual machine and a vcpu, then copies 0x1380 bytes into the vm. 
We then dd this code into a second file for examination.
The rest of the outer binary appears to pass stdin/stdout through to the vm, and does *something*m when the vm halts (then reruns it).

The inner binary is obviously an exectable, but both IDA and Binary ninja botch the dissassembly. After some examining of the hex dump in binary ninja, we can recover a few more function fragments.
<picture>
We see that the 
The *something* that the outer binary does now becomes clear, when the inner binary halts and sets rax to a magic number, 
the outer binary checks this magic value, does a lookup in a table, then changes the inner binaries rip to that value. 
<picture of table>
In this way,
the inner binary is able to push its control flow through the outer binary.

After manually deobfuscating the control flow of the inner binary by changing the halts to the corresponding jumps, the binary becomes much simpler; stage 1 complete. It is worthwhile to note how easy Binary Ninja made this pathcing process with its patch->assemble function that allows you to directly insert assembly instructions into the binary. 
<picture of inner binary>

We now see that it is comparing our input string against a constant string in memory after performing some bit shifting. Its keygen time!
We see that it is performing lookups in a constant data structure where each item seems to have 3 fields, and two variants of the data structure
<picture of binary tree>
Variant  1:
Field 1: ff
Field 2: ptr into data structure
Field 3: ptr into data structure

Variant 2:
Field 1: character
Field 2: null
Field 3: null

With a bit of thinking, we realize this is a binary tree, which makes this function
<picture of function>
a preorder traversal.

We can see that each character in our input string is being searched for in this tree, and some bits are being stored to a result
that is then compared against the magic constant string to see if we win.
We discover that another one of the functions just adds a 1 or 0 to our result bitstring.
Which means that for any given character in our input string, its mapping become the path needed to take in the binary tree to find the character,
0 for left, 1 for right; kind of like an bad huffman encoding.

After building up a map between input characters and their resulting bit strings, we bruteforce decoded all possible strings in the data section of the binary.
This gives us the flag!



