# StrangeVM

- reverse 


This challenge is pretty straight forward, no need of gdb (unless you love it too much), just the raw decompiled and some of your time.

## Chall structure

It consists of 2 files, vm (an elf32 executable) and code.pascal (some strange file containing some compiled stuff).
By decompling the vm file with bj it immediatly makes sense why it is called that way, the executable emulates a CPU by defining its own ISA (instruction set architecture), it takes code.pascal in input as some sort of bytecode, initializes an empty mem space, where the modfications of code.pascal will be stored and then will finally compare the resulting mem state with an harcoded target mem stae, if those two match, you win, the input (the real ctf flag), is correct!

```C
00401e4c    int64_t executeVM()

00401e4c    {
00401e4c        int32_t var_c = 0;
00401e4c        
00402192        while (true)
00402192        {
00402192            char result = *(uint8_t*)((int64_t)var_c + code);
00402192            
00402197            if (!result)
004021a0                return result;
004021a0            
00401e6d            int32_t var_c_1 = var_c + 1;
00401e78            int32_t rax_5 = (int32_t)*(uint8_t*)((int64_t)var_c + code);
00401e78            
00401e7e            if (rax_5 != 6)
00401e7e            {
00401e87                if (rax_5 == 5)
00401e90                {
004020bd                    readInt((int64_t)var_c_1 + code);
004020c5                    mem;
004020e1                    __isoc23_scanf("%c", 0);
004020e6                    var_c = var_c_1 + 4;
004020ea                    continue;
00401e90                }
00401e90                else if (rax_5 == 4)
00401ea2                {
00402066                    int32_t rax_56 = readInt((int64_t)var_c_1 + code);
004020a0                    *(uint8_t*)(mem + (int64_t)rax_56) = readByte(code + (int64_t)var_c_1 + 4);
004020a2                    var_c = var_c_1 + 5;
004020a6                    continue;
00401ea2                }
00401ea2                else if (rax_5 == 3)
00401eb4                {
00401fc5                    int32_t rax_39 = readInt((int64_t)var_c_1 + code);
00401fe4                    char rax_42 = readByte(code + (int64_t)var_c_1 + 4);
00401fe4                    
00401ff0                    if (!rax_42)
00401ff0                        break;
00401ff0                    
00402049                    *(uint8_t*)((int64_t)rax_39 + mem) = (int8_t)((int64_t)*(uint8_t*)((int64_t)rax_39 + mem) % (int32_t)rax_42);
0040204b                    var_c = var_c_1 + 5;
0040204f                    continue;
00401eb4                }
00401eb4                else if (rax_5 == 1)
00401ec6                {
00401ee4                    int32_t rax_9 = readInt((int64_t)var_c_1 + code);
00401f03                    char rax_12 = readByte(code + (int64_t)var_c_1 + 4);
00401f37                    *(uint8_t*)((int64_t)rax_9 + mem) += rax_12;
00401f39                    var_c = var_c_1 + 5;
00401f3d                    continue;
00401ec6                }
00401ec6                else if (rax_5 == 2)
00401ecb                {
00401f54                    int32_t rax_24 = readInt((int64_t)var_c_1 + code);
00401f73                    char rax_27 = readByte(code + (int64_t)var_c_1 + 4);
00401fa8                    *(uint8_t*)((int64_t)rax_24 + mem) -= rax_27;
00401faa                    var_c = var_c_1 + 5;
00401fae                    continue;
00401ecb                }
00401ecb                
0040215a                *(uint8_t*)((int64_t)var_c_1 + code);
00402174                _IO_fprintf(stderr, "Unknown operation code: %d\n", 0);
0040217e                exit(1);
0040217e                /* no return */
00401e7e            }
00401e7e            
00402101            int32_t rax_73 = readInt((int64_t)var_c_1 + code);
00402120            char rax_76 = readByte(code + (int64_t)var_c_1 + 4);
00402120            
0040213c            if (!*(uint8_t*)((int64_t)rax_73 + mem))
00402142                var_c_1 += (int32_t)rax_76;
00402142            
00402145            var_c = var_c_1 + 5;
00402192        }
00402192        
0040200b        _IO_fwrite("Division by zero error\n", 1, 0x17, stderr);
00402015        exit(1);
00402015        /* no return */
00401e4c    }

```
The fuction executeVM() checks the input file (code.pascal) byte by byte and interprets it based on its ISA by operating an instruction cycle fetch -> decode -> execute.

This particular set defines 6 istructions: JZ (opcode 6), USER INPUT (opcode 5), MOV (opcode 4), MOD (opcode 3), SUB (opcode 2), ADD (opcode 1).

By the if statements you can understand how each sequence of bytes of the byteocde gets interpreted by the vm:

-  JZ: [ 06 ] [ xx xx xx xx ] [ JJ ] -> this instruction is composed by 6 bytes, [ 06 ] = opcode, [ xx xx xx xx ] checks if this mem addr is 0, [ JJ ] offset shit where to jump
-  USER INPUT: [ 05 ] [ xx xx xx xx ] -> instruction composed by 5 bytes, [ 05 ] = opcode, [ xx xx xx xx ] mem addr where to store user input
-  MOV: [ 04 ] [ xx xx xx xx ] [ MM ] -> 6 bytes, [ 04 ] = opcode, [ xx xx xx xx ] mem addr where to write the value, [ MM ] value to write in mem
-  MOD: [ 03 ] [ xx xx xx xx ] [ DD ] -> 6 bytes, [ 03 ] = opcode, [ xx xx xx xx ] mem addr where to store the result, [ DD ] it is the value to divide by ( mem[ addr ] % divisor )  
-  SUB: [ 02 ] [ xx xx xx xx ] [ SS ] -> 6 bytes, [ 02 ] = opcode, [ xx xx xx xx ] same as MOD, [ SS ] value to subtract ( mem[ Address ] - value ) 
-  SUB: [ 01 ] [ xx xx xx xx ] [ AA ] -> 6 bytes, [ 01 ] = opcode, [ xx xx xx xx ] same as MOD, [ AA ] value to add ( mem[ Address ] + value )

## Solution idea 

Defined the flux of execution of the program you have a bunch of ways to solve this challenge, you can reverse the vm logic by running the vm on the bytecode, taking note of each operation and then running it backwards by inverting math operations, that you understand being a pretty long way to go so i thought of a better one.

The idea is that you can simply discover the input flag by running the vm on a dummy input (all zeros) so you can discover all bytecode modifications. 

To be more clear:
Imagine you have a scale that got a weighting error of 10 kgs, every time someone gets on the scale it adds 10 kgs to the actual weight. You dont know what is the error but you feel that the final weight is wrong what would you do to discover how many kilos the scale is adding?
Simple: you put 0 kgs on it (leave it empty) and you'll see that the scale is adding 10 kgs, so to get the right weight now you can just make final weight - 10 and you get the expected output.

Here we approach the challenge the same way:

$\text{weight} + \text{modification} = \text{final weight}$

$\text{input} + \text{modifications} = \text{output}$

$\text{flag} + \text{modifications} = \text{target memory state}$

we dont know flag but we know final memory state (hardcded in the binary for the last check). if we put 0 as dummy input we will get the byecode modifications.



$0 + \text{modifications} = \text{result memory state}$

so we see that target_memory will be exactly the modifications so:

$flag  + \text{result memory state} = \text{target memory state}$

$flag = \text{target memory state} - \text{result memory state}$

So to solve the challenge we emulate the vm throught a python script, giving as input all zeros. The resulting memory will be the modifications and so we will able to tell that the flag is equal to the target memory (hardcoded in bynary) - the result of vm emulation on all zeros.

The exploit for this challenge will be:

```Python
#check flag hardoced in the binary
final_flag_mem_check = [0x56, 0x4c, 0x75, 0x5c, 0x38, 0x6d, 0x39, 0x58, 0x6c, 0x28, 0x3e, 0x57, 0x7b, 0x5f, 0x3f, 0x54, 0x44, 0x5b, 0x71, 0x20, 0x82, 0x1b, 0x8b, 0x50, 0x80, 0x46, 0x7e, 0x15, 0x8a, 0x57, 0x7d, 0x5a, 0x50, 0x54, 0x81, 0x51, 0x8c, 0x0c, 0x94, 0x44, 0x00]


with open("code.pascal", "rb") as f:
    code = f.read()

zeroes_input = [0] * len(final_flag_mem_check)

mem =[0] * 1024
i = 0
user_input_i = 0
while (i < len(code)):  
    
    if (code[i] == 6):  #[ 06 ] [ AA BB CC DD ] [ JJ ] sarebbe un JZ
        mem_write_addr = int.from_bytes(code[i+1:i+5], 'little')
        bytes_to_jump = code[i+5]
        
        #Current Position + 6 (Instruction Size) + 12 (Offset)
        i+=6

        if (mem[mem_write_addr] == 0):
            i += bytes_to_jump      
        continue
    if(code[i] == 5):   #[ 05 ] [ AA BB CC DD ] example instruction read user input (5 bytes)
        mem_write_addr = int.from_bytes(code[i+1:i+5], 'little')        
        
        element = 0
        if (user_input_i < len(zeroes_input)):      
            element = zeroes_input[user_input_i]    
            user_input_i += 1
        mem[mem_write_addr] = element
        i += 5
        continue
    if(code[i] == 4):   #[ 04 ] [ AA BB CC DD ] [ VV ] 6 bytes MOV
        mem_write_addr = int.from_bytes(code[i+1:i+5], 'little')
        element = code[i+5]
        mem[mem_write_addr] = element
        i += 6
        continue
    if(code[i] == 3):   #[ 03 ] [ AA BB CC DD ] [ VV ] 6 bytes MOD
        mem_write_addr = int.from_bytes(code[i+1:i+5], 'little')
        element = code[i+5]
        mem[mem_write_addr] %= element      
        i += 6
        continue
    if(code[i] == 2):   #[ 02 ] [ AA BB CC DD ] [ VV ] 6 bytes SUB.
        mem_write_addr = int.from_bytes(code[i+1:i+5], 'little')
        element = code[i+5]
        mem[mem_write_addr] = (mem[mem_write_addr] - element) & 0xFF
        i += 6
        continue
    if(code[i] == 1):   #[ 01 ] [ AA BB CC DD ] [ VV ] 6 bytes ADD.
        mem_write_addr = int.from_bytes(code[i+1:i+5], 'little')
        element = code[i+5]
        mem[mem_write_addr] = (mem[mem_write_addr] + element) & 0xFF
        i += 6
        continue
    if (code[i] == 0): 
        break

    i += 1


# 0 + Modification = result_mem 
# Flag + result_mem = Target_mem
# Flag = target_mem - result_mem
flag = ""
for i in range(len(final_flag_mem_check) - 1):

    target_mem_value = final_flag_mem_check[i]
    target_result_value = mem[i]

    diff = (target_mem_value - target_result_value) & 0xFF
    flag+= chr(diff)

print(flag)
```

that returns us the correct flag: `VMs_4r3_d14bol1c4l_3n0ugh_d0nt_y0u_th1nk`