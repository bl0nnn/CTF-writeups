# Nucleus

- reverse 

This challenge gives out just a windows executable "nucleus21.exe". As any reverse challenge the first thing to do is running the executable. Given the fact that I was solving this from a Mac I had to use [Wine](https://www.winehq.org), a compatibility layer to run Windows executables on posix-compliant OSs. To run simple executables like this one I always prefer to proceed this way, wine simply translates Windows APIs to Posix calls on the fly, no performance or memory to be worring about (If you want to use it too just remember that Wine was made for intel processors so you'll need to use Rosetta if you want to run it on you apple silicon mac).
That said, by running the executable I noticed something strange: it created a second identical file with the same name (nucleus{XX}.exe) but with the number on in the name incremented by one (e.g. nucleus21.exe if ran creates nucleus22.exe).  mmmh self riplicant code?? Lets disassemble!

## Analysis
I disassembled the executable with my preferred disassembler and started analyzing the code.
By looking at the _main_ function my doubts were almost immediatly confirmed:

```C
140001360    int64_t main()

140001360    {
140001360        void var_258;
140001374        int64_t rax_1 = __security_cookie ^ &var_258;
140001391        void var_128;
140001391        uint32_t rax_2 = GetModuleFileNameA(nullptr, &var_128, 0x104);
14000139b        char var_238[0x110];
14000139b        
14000139b        if (rax_2)
14000139b        {
1400013ac            int64_t r9_1 = strcpy_s(&var_238, 0x104, &var_128);
1400013bd            char* rcx_2 = &var_238[(int64_t)(rax_2 - 5)];
1400013c0            char rdx_1 = *(uint8_t*)rcx_2;
1400013c0            
1400013c8            if (rdx_1 - 0x30 <= 9)
1400013c8            {
1400013cd                if (rdx_1 != 0x39)
1400013e4                    *(uint8_t*)rcx_2 = rdx_1 + 1;
1400013cd                else
1400013db                    sub_140001070(rcx_2, (int64_t)(0x104 - (rax_2 - 5)), "10.exe", r9_1);
1400013db                
1400013f6                CopyFileA(&var_128, &var_238, 0);
1400013c8            }
14000139b        }
14000139b        
140001401        sub_1400010d0(&var_238);
140001413        __security_check_cookie(rax_1 ^ &var_258);
140001428        return 0;
140001360    }

```

The function `GetModuleFileNameA` is a standard windows api call that returns the length of the path of the current runnnig executable that called it + it puts in var_128 the actual path it reads.
That length is important beacause it is later used to check the the fifth (starting from the last) char of the path so in this case this char will be the one exactly before the extension (e.g. \Desktop\File1.exe has length 18, `char* rcx_2 = &var_238[(18 - 5)];` -> "1")
So `char* rcx_2 = &var_238[(int64_t)(rax_2 - 5)];` simply reads the path at that certain offset and saves that char in rcx_2. Note how the the second if (`if (rdx_1 - 0x30 <= 9)`) checks if the char is a number between 0 and 9, only those are accepted.
If the number is between 0 and 8 it simpy changes the file name incrementing the numeber (here is why i was seeing the new file generated with the incremented last number), if it is 9 it calls `sub_14000107` that simply converts the end of the string to 10.exe (e.g., changing file9.exe to file10.exe), to go over 9.
It finally copies the current executable inside the newly created file and calls `sub_1400010d0()` on the new named file.


`sub_1400010d0()` is a pretty long function but the important thing it does are 4:
1. It takes the path of the currently running executable, opens it and calculate its total size. Allocate memory into the variable lpBuffer and pastes the content of the .exe file into it.
    ```C
    GetModuleFileNameA(nullptr, &var_138, 0x104)
    HANDLE rax_4 = CreateFileA(&var_138, 0x80000000, FILE_SHARE_READ, nullptr, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, nullptr);
    uint32_t rax_6 = GetFileSize(rax_4, nullptr);
    uint8_t* lpBuffer = malloc((uint64_t)rax_6);
    ReadFile(rax_4, lpBuffer, rax_6, &numberOfBytesRead, var_168_1)
    ```
2. It waits for user input. 
    ```C
    int32_t rax_9 = getchar();
    char key = (uint8_t)rax_9;

    if (rax_9 != 0xa && rax_9 != 0xffffffff)
    {
        int32_t i;

        do
        {
            i = getchar();

            if (i == 0xa)
                break;
        } while (i != 0xffffffff);
    }
    ```
    and just takes the first char of whatever the user has inputted (the do while in fact discards all the other chars)
3. All good until this thing right here:
    ```C
        uint128_t zmm1 = (uint128_t)(int32_t)key;
        zmm1 = _mm_unpacklo_epi8(zmm1, (uint64_t)zmm1);
        zmm1 = _mm_shuffle_epi32(
        _mm_unpacklo_epi16(zmm1, (uint64_t)zmm1), 0);

        if (rsi_2){
            if (rsi_2 >= 0x40)
            {
                int32_t r8_2 = 0x20;

                do
                {
                    uint64_t i_3 = (uint64_t)i_1;
                    i_1 += 0x40;
                    *(uint128_t*)(lpBuffer + i_3) ^= zmm1;
                    
                    int128_t* rax_11 = (uint64_t)(r8_2 - 0x10);
                    *(uint128_t*)((char*)rax_11 + lpBuffer) ^= zmm1;
                    
                    int128_t* rax_12 = (uint64_t)r8_2;
                    *(uint128_t*)((char*)rax_12 + lpBuffer) ^= zmm1;
                    
                    int128_t* rax_13 = (uint64_t)(r8_2 + 0x10);
                    r8_2 += 0x40;
                    *(uint128_t*)((char*)rax_13 + lpBuffer) ^= zmm1;

                } while (i_1 < (rsi_2 & 0xffffffc0));
            }
        }
    ```
    This got me confused for some time. It saves the user inserted key in zmm1 and calls 3 different functions on that key. Shut out to Gemini and Intel Intrinsics Guide that helped me understand what all this mess was.
    In substance what this chunk of code wants to do is xorring the full .exe code of the current running executable. Normally a CPU processes one chunk of data at a time so if you want for example to xor 100k bytes file you should do 100k iterations, xoring each byte with the key, resulting in 100k separate CPU cycles (one for each byte) to avoid this intel exports a set of functions to optimize this compilation time.
    So what this does is just xorring the immense payload generated by the running .exe not one byte at a time but 64 bytes at a time. (e.g. user input: "a" it becomes "a"*64).

4. Finally it takes the generated payload and puts it in the new generated file (the one with the number incremented), specifically, through the api UpdateResourceA() so inside the RC_DATA section (0xa) at ID (101). 

## Solution idea
So now what this chall do is clear: it replicates the proram encrypting it with a xor function. The idea is to get all the input keys the creator of the challenge inserted to get the flag, 21 files means 21 chars. To do that we can simply take nucleus21, dump the rcdata section, get the key by xorring a certain value with the first values of the paylaod (that in windows executable is always M, MZ to be exact) and xor it with the values at ID 101.

The exploit is just a simple pe parser (using the pefile library), creating at each iteration a nucleusXX.file with the dumped pe from the parsed executable, extract the key char at section 101 and do that till nucleus0.exe

```Python
import pefile
import os

flag = []
current_exe = "nucleus21.exe"
for i in range(21, 0, -1):

    pe = pefile.PE(current_exe)
    for entry in pe.DIRECTORY_ENTRY_RESOURCE.entries:
        if entry.struct.Id == 10:
            for resource_id in entry.directory.entries:
                if resource_id.struct.Id == 0x65:
                    data_rva = resource_id.directory.entries[0].data.struct.OffsetToData
                    size = resource_id.directory.entries[0].data.struct.Size        
                    payload_bytes = pe.get_memory_mapped_image()[data_rva:data_rva+size]
                    break
    
    key_value = payload_bytes[0] ^ 0x4D
    flag.append(chr(key_value))
    decrypted_payload = ""
    decrypted_payload = bytearray(value ^ key_value for value in payload_bytes)

    next_exe = f"nucleus{i-1}.exe"

    with open(next_exe, 'wb') as f:
        f.write(decrypted_payload) 
    
    current_exe = next_exe

print(flag)   
ordered_flag=""
for i in range(len(flag) - 1, -1, -1):
    ordered_flag += flag[i]

print(ordered_flag)
```

which prints eventually the right flag: gigem{RCD4Ta_i5_N3aT}