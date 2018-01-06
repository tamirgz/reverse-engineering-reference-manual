# .encodings

## *<p align='center'> String Encoding </p>*
* __ASCII__: string encoding that maps a byte to an English character, a special character, or a number
  * [ASCII Table](http://www.asciitable.com/)
  * Out of the 128 characters defined in ASCII, only 95 of them are human-readable
  * ASCII used 7 bits only, but the extra bit is still not enough to encode all the other languages
* __Unicode__: Various encoding schemes were invented but none covered every languages until Unicode came along
  * [Unicode Character Table](https://unicode-table.com/en/#control-character)
  * Unicode is a large table mapping every character to a unique numbers (code point) 
  * First 256 code points maps 1:1 to ASCII  
  * Different UTF encodings (e.g. UTF-8, UTF-16) use different amount of bytes to encode those code points
* __Cause Of Garbled Text__: reading a byte sequence using the wrong encoding scheme
  
#
## *<p align='center'> Data Encoding </p>*
* __Definition__: all forms of content modification for the purpose of hiding intent
* __Caesar Cipher__: formed by shifting the letters of alphabet #’s characters to the left or right
* __Single-Byte XOR Encoding__: modifies each byte of plaintext by performing a logical XOR operation with a static byte value
  * __Identifying XOR Loop__: looks for a small loop that contains the XOR function (where it is xor-ing a register and a constant or a register with another register)
  * __Single-byte XOR's Weakness__: if there are many null bytes then key will be easy to figure out since XOR-ing nulls with the key reveals the key. 
  * __Solutions To Single-Byte XOR Encoding's Weakness__: 
    + Null-preserving single-byte XOR encoding: if plaintext is NULL or key itself, then it will not be encoded via XOR
    + Generates the keystream used to XOR the data using a pseudorandom number generator 
      * __Blum Blum Shub PRNG__: generic form: Value<sup>i+1</sup> = (Value<sup>i</sup> * Value<sup>i</sup>) % M. M is a constant  that is the product of 2 large primes and an initial V needs to be given. Actual key being xor-ed with the data is the lowest byte of current PRNG value
* __Other Simple Encoding Scheme__:
  + __ADD, SUB__
  + __ROL, ROR__: Instructions rotate the bits within a byte right or left
  + __Multibyte__: XOR key is multibyte
  + __Chained or Loopback__: Use content itself as part of the key
    * the original key is applied at one side of the plaintext and the encoded output character is used as the key for the next character
* __Data Encoding Example (Base64)__:
  + Encodes binary data into character set of 64 ASCII characters
  * Most common character set is MIME’s Base64, whose table consists of A-Z, a-z, and 0-9 for the first 62 values and + / for the last 2 values
  * Base64 operates every 3 bytes (24 bits). For every 6 bits, it indexes the table with 64 characters. The encoded value is the character that is indexed with the 6 bits 
  * One padding character may be presented at the end of the encoded string (typically =) since Base64 operates every 3 bytes
  * Easy to develop a custom substitution cipher using Base64 since the only item that needs to be changed is the indexing string table of 64 characters
  * Example:

<div align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/encodings/Data_Encoding/base64_conversion.png"> 
<p align='center'><sub><strong>base64 conversion</strong></sub></p>
</div>

