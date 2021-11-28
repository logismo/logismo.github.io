---
title: "picoCTF 2021: Transformation Write-up | Reverse Engineering"
date: 2021-11-28
categories: [picoCTF]
tags: [picoCTF, reverse engineering, python, unicode, bitwise operations]
---

## Description
> I wonder what this really is... enc
>
> `''.join([chr((ord(flag[i]) << 8) + ord(flag[i + 1])) for i in range(0, len(flag), 2)])`

## Solution
This challenge provides a snippet whose syntax should be recognizable if you're familiar with Python, and a text file named `enc` with the following contents:
```bash
$ file enc
 enc: Unicode text, UTF-8 text, with no line terminators
$ cat enc
 灩捯䍔䙻ㄶ形楴獟楮獴㌴摟潦弸弲㘶㠴挲ぽ
```

The Python snippet has a variable named `flag`, so assuming that the contents of `enc` is the flag encoded by the snippet, we can try to run the operations of the snippet in reverse in order to decode the flag.

The first part of the operation:
```python
ord(flag[i]) << 8 
```
takes the first character in the flag, runs it through the [`ord()` built-in Python function](https://docs.python.org/3.4/library/functions.html?highlight=ord#ord){:target="_blank"} converting it to an integer Unicode code. Then, a [bitwise left shift operation](https://wiki.python.org/moin/BitwiseOperators){:target="_blank"} by 8 bits ( `<< 8` ) is performed. 

The Unicode code of the second character gets added to this value:
```python
+ ord(flag[i + 1])
```
which is finally converted back to a character through  [`chr()`](https://docs.python.org/3.4/library/functions.html?highlight=#chr){:target="_blank"}.

In sum, we can think of this as every two characters of the flag being encoded into a single character in the `enc` string.

So by doing the inverse of the first operation – shifting the Unicode value of our first encoded character 8 bits to the right – we get the following: 

```python
>>> chr(ord('灩') >> 8)
'p'
 ```
Knowing that the flag follows the format `picoCTF{FLAG}` , it looks like we're on the right track.

As for the second operation, subtracting the first decoded character, left shifted by eight bits, from the encoded character should give us the second decoded character:

```python
>>> chr(ord('灩') - (ord('p') << 8))
'i'
```

Repeating these steps for all the characters in the encoded string gives us:
```python
enc = '灩捯䍔䙻ㄶ形楴獟楮獴㌴摟潦弸弲㘶㠴挲ぽ'
flag = ''

for i in range(len(enc)):
    flag += chr(ord(enc[i]) >> 8)
    flag += chr(ord(enc[i]) - (ord(flag[-1]) << 8))

print(flag)
# output: picoCTF{16_bits_inst34d_of_8_26684c20}
```
And that's how you reverse engineer the encoding.

## In-depth Explanation

The Python snippet used to encode the flag is essentially an algorithm for encoding [Unicode](https://docs.python.org/3/howto/unicode.html){:target="_blank"} UTF-8 (the default encoding) strings into UTF-16 (Big Endian). 

Like the flag itself hints at (`16_bits_inst34d_of_8`). An *a posteriori* hint, if you will.

We can see what's happening under the hood by printing a character's value in binary:
```python
>>> bin(ord('p'))
'0b1110000'
```

Left-shifted by 8 bits:

```python
>>> bin(ord('p') << 8)
'0b111000000000000'
```

Plus the value of the next character:

```python
>>> bin((ord('p') << 8) + ord('i'))
'0b111000001101001'
```

So we can think of left-shifting as padding the letter "p" with 8 empty bits, which are then filled by the second letter in the flag, and so forth for all characters. This is why right-shifting by 8 bits gives us the original first character (since we remove the padding being filled by the second character), and the logic for the rest of the reverse engineering follows from there. Subtracting the value of `'ord('p') << 8'` zeroes out the leading 8 bits and we're left with "i".

Since the default encoding is UTF-8 and the `enc` string is UTF-16, the characters are rendered incorrectly (as multiple *kanji*).

Therefore, an alternate way to solve this challenge is to simply:

## Alternate Solution

Encode the `enc` string in UTF-16BE:

```python
>>> '灩捯䍔䙻ㄶ形楴獟楮獴㌴摟潦弸弲㘶㠴挲ぽ'.encode('utf-16be')
b'picoCTF{16_bits_inst34d_of_8_26684c20}'
```
