# CryptOfTheUndead Writeup

## Challenge Overview

**Difficulty:** Very Easy

This challenge involved analyzing and defeating a file encryption program called the "zombifier" that uses ChaCha20 encryption with a hardcoded key.

---

## Initial Analysis: String Extraction

I began by using `strings` to extract readable data from the binary, which revealed several important clues:

```
error: that which is undead may not be encrypted
BRAAAAAAAAAAAAAAAAAAAAAAAAAINS!!
successfully zombified your file!
chacha20_block_next
GCC: (GNU) 14.2.1 20240805
GCC: (GNU) 14.2.1 20240910
main.c
main.cold
```

### Key Findings

From these strings, I determined that:
- The program was written in CPP and compiled with GCC 14.2.1
- ChaCha20 encryption** was being used
- A hardcoded encryption key was present: `BRAAAAAAAAAAAAAAAAAAAAAAAAAINS!!`
- Files receive a `.undead` extension after encryption

---

## Reverse Engineering with Ghidra

I opened the binary in **Ghidra** to analyze the `main` function and understand the encryption logic:

```c
undefined4 main(int param_1,undefined8 *param_2)
{
  int iVar1;
  size_t sVar2;
  char *__dest;
  char *pcVar3;
  undefined4 uVar4;
  long in_FS_OFFSET;
  void *local_40;
  size_t local_38;
  long local_30;

  local_30 = *(long *)(in_FS_OFFSET + 0x28);
  if (param_1 < 2) {
    pcVar3 = "crypt";
    if (param_1 == 1) {
      pcVar3 = (char *)*param_2;
    }
    uVar4 = 1;
    printf("Usage: %s file_to_encrypt\n",pcVar3);
  }
  else {
    pcVar3 = (char *)param_2[1];
    iVar1 = ends_with(pcVar3,".undead");
    if (iVar1 == 0) {
      sVar2 = strlen(pcVar3);
      __dest = malloc(sVar2 + 9);
      strncpy(__dest,pcVar3,sVar2 + 9);
      sVar2 = strlen(__dest);
      builtin_strncpy(__dest + sVar2,".undead",8);
      iVar1 = read_file(pcVar3,&local_38,&local_40);
      if (iVar1 == 0) {
        encrypt_buf(local_40,local_38,"BRAAAAAAAAAAAAAAAAAAAAAAAAAINS!!");
        iVar1 = rename(pcVar3,__dest);
        if (iVar1 == 0) {
          iVar1 = open(__dest,0x201);
          if (iVar1 < 0) {
            uVar4 = 5;
            perror("error opening new file");
          }
          else {
            write(iVar1,local_40,local_38);
            close(iVar1);
            puts("successfully zombified your file!");
            uVar4 = 0;
          }
        }
        else {
          uVar4 = 4;
          perror("error renaming file");
        }
      }
      else {
        uVar4 = 3;
        perror("error reading file");
      }
    }
    else {
      uVar4 = 2;
      puts("error: that which is undead may not be encrypted");
    }
  }
  if (local_30 == *(long *)(in_FS_OFFSET + 0x28)) {
    return uVar4;
  }
                    /* WARNING: Subroutine does not return */
  __stack_chk_fail();
}
```

### Understanding the Encryption Process

The pseudo-code reveals the encryption workflow:

1. Read the input file into a buffer
2. Check if the file is already encrypted (has `.undead` extension)—if so, reject it
3. Encrypt the buffer using the hardcoded key `"BRAAAAAAAAAAAAAAAAAAAAAAAAAINS!!"`
4. Rename the original file by appending `.undead` extension
5. Write the encrypted data to the new file

The vulnerability is pretty trivial: The key is hardcoded, meaning it can be easily recovered. 

---

## Building a Decryption Tool

Rather than manually recreate the decryption logic, I leveraged an LLM to quickly generate a Python decryption tool. This is a practical approach when time is critical in real penetration testing or incident response scenarios.

### Decryption Script

```python
#!/usr/bin/env python3
import sys
import argparse
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms
from cryptography.hazmat.backends import default_backend

def decrypt_file(encrypted_file, output_file=None):
    """
    Decrypt a .undead file that was encrypted with ChaCha20.

    Args:
        encrypted_file: Path to the .undead encrypted file
        output_file: Path to write decrypted data (defaults to filename without .undead)
    """

    # Hardcoded key from the binary
    KEY = b"BRAAAAAAAAAAAAAAAAAAAAAAAAAINS!!"

    # ChaCha20 requires a 16-byte nonce
    NONCE = bytes(16)

    try:
        # Read encrypted file
        with open(encrypted_file, 'rb') as f:
            ciphertext = f.read()

        if not ciphertext:
            print(f"Error: {encrypted_file} is empty")
            return False

        # Determine output filename
        if output_file is None:
            if encrypted_file.endswith('.undead'):
                output_file = encrypted_file[:-7]  # Remove .undead extension
            else:
                output_file = encrypted_file + '.decrypted'

        # Decrypt using ChaCha20
        cipher = Cipher(
            algorithms.ChaCha20(KEY, NONCE),
            None,
            backend=default_backend()
        )
        decryptor = cipher.decryptor()
        plaintext = decryptor.update(ciphertext) + decryptor.finalize()

        # Write decrypted data
        with open(output_file, 'wb') as f:
            f.write(plaintext)

        print(f"✓ Successfully decrypted: {encrypted_file}")
        print(f"✓ Output written to: {output_file}")
        return True

    except FileNotFoundError:
        print(f"Error: File not found: {encrypted_file}")
        return False
    except Exception as e:
        print(f"Error during decryption: {e}")
        return False

def main():
    parser = argparse.ArgumentParser(
        description='Decrypt files encrypted with the ChaCha20 "zombifier"'
    )
    parser.add_argument('encrypted_file', help='Path to .undead encrypted file')
    parser.add_argument('-o', '--output', help='Output file path (optional)')

    args = parser.parse_args()

    success = decrypt_file(args.encrypted_file, args.output)
    sys.exit(0 if success else 1)

if __name__ == '__main__':
    main()
```

### How It Works

The decryption tool is basic and fairly straightforward:

1. Reads the encrypted file
2. Uses the hardcoded key and a zero-initialized 16-byte nonce
3. Applies ChaCha20 decryption using the `cryptography` library
4. Writes the plaintext to the output file

---

## Key Vulnerabilities

This challenge demonstrates 2 highly critical security flaws:

It has a hardcoded encryption key, meaning anyone with basic reverse engineering knowledge can obtain the key and decrypt the file.

There is no key derivation, meaning each file uses identical encryption parameters.

---

## Solution

After building the decryption tool, I simply ran it against the encrypted flag file to recover the plaintext and complete the challenge.
