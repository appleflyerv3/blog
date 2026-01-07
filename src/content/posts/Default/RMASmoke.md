---
title: "Leaking the Google Cr50 BootROM"
date: "2026-01-07"
description: "Using a buffer overflow to leak the Google Cr50 ROM!"
published: 2026-01-07
---

## Code execution within the Cr50
HavenOverflow has managed to leak the Cr50 BootROM by exploiting a buffer overflow in the TPM2 task \
(this vulnerability was found by writable, codenamed RMASmoke, <https://issuetracker.google.com/issues/324336238>). \
The vulnerability involved creating an NVMEM space of more than 1024 bytes, and then proceeding to read from it. Reading back from this NVMEM space of more than 1024 bytes caused the out buffer of 1024 bytes in the TPM2_NV_Read function to overflow into the saved registers on the stack and other stack contents, resulting in us being able to override the stack, giving us control over the pc(program counter) and all the CPU registers in the Cr50. (thanks writable, aka unretained, aka profile_encryption for the RMASmoke vulnerability!!)

Although we could only jump to instructions that already exist in the Cr50 RW, and not write and execute our own instructions, using a technique called a ROP chain, which involves using existing instructions, this basically gave us code execution as the instructions that already exist in the Cr50 RW was sufficient for us to:
- write to registers(full control over things like the CRYPTO, USB, UART, I2C, FLASH, KEYMGR, FUSE, WATCHDOG peripherals)
- read from regsisters
- read from allowed memory regions
- print to the console

## RMASmoke has been patched though...
Although RMASmoke has been patched since Cr50 RW version 0.5.230, we were able to find chromebooks which a Cr50 RW that was below this version. Therefore, we were able to abuse RMASmoke on the affected chromebooks.

We have made a ROP chain for Cr50 RW versions 0.5.153 and 0.5.120. If you want a ROP chain for your Cr50 version for some reason, please dm appleflyer on discord.

## HIDE_ROM exists, but HIDE_ROM does not work
Google has thought about such a case where if one had code execution over the Cr50, one may be able to leak the BootROM. Therefore, the developers created the HIDE_ROM register to lock down the BootROM memory region once the code in the RO completes execution to prevent reading from the BootROM.
Initially, we thought that this would stop us from reading from the BootROM memory region. But...
Attempts to read from the address 0x0 with RMASmoke did not result in a force shutdown by the CPU or any halt whatsoever, neither did it return just 0s or 1s. Instead, the actual code in the BootROM memory region was printed out.

For some reason, HIDE_ROM was not working as intended. Therefore, it allowed us to leak the BootROM. This issue is also unfixable as GLOBALSEC's logic cannot be changed.

There are many possibilites why HIDE_ROM did not work:
- HIDE_ROM was just to stop execution from the BootROM, not to prevent reads on the BootROM memory region, although that would defeat the purpose of "hiding the BootROM", as suggested by the name HIDE_ROM.
- the name HIDE_ROM given to the register 0x400940d0 was not even anything to do with the BootROM, it could also just have been a dummy register, or it could have to do with something else completely.
- it could also have been some scare or lie by google to prevent people from trying to leak the BootROM.

To this day, we are unsure why HIDE_ROM does not work, or what the purpose of HIDE_ROM was.

## Leaking the BootROM
With HIDE_ROM not working properly, and with us having code execution over the Cr50, we could disable the Cr50's WATCHDOG with a few register writes.
Disabling the WATCHDOG gave us enough time for our code to run.

We were then able to successfully write and execute 2 ROP chains.
- a ROP chain to bump our stack pointer to another region where we had more space to execute more code
- a ROP chain to read from the BootROM and leak the BootROM bytes

We were able to call the `printf` function in the Cr50 to print out the BootROM at the address 0x0 as hex over the USB console(via a SuzyQable).

## BootROM source code
After leaking the BootROM, HavenOverflow has decompiled the BootROM entirely. The BootROM is now publically available and compilable.

Original leaked BootROM: <https://github.com/HavenOverflow/gscemu/raw/refs/heads/main/fw/rom/haven.rom> \
C source code: <https://github.com/HavenOverflow/Cr50/tree/main/chip/haven/rom> \
C source compiled BootROM: <https://github.com/HavenOverflow/gscemu/raw/refs/heads/main/fw/rom/haven_C_source.rom>

## Other purposes of RMASmoke
Other than leaking the BootROM, we could also test behavior of peripherals within the Cr50, such as
- WATCHDOG register behavior
- FUSE register values

---

## credits:
- hannah: the idea to abuse RMASmoke to leak the BootROM \
          rewriting the RMASmoke PoC written by writable originally: making an NVMEM space larger than 1024 bytes and reading from that NVMEM space

- writable: finding the buffer overflow in the Cr50 TPM2 task

- appleflyer: verifying we had control over the stack and pc(program counter) in the Cr50 with RMASmoke \
              writing the ROP chain to leak the BootROM \
              buying an RMASmoke vulnerable chromebook(it was 75 USD btw) and running RMASmoke \
              writing this writeup :D
