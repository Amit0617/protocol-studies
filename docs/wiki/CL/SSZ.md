# Simple Serialize (SSZ)

## Overview

Simple Serialize (SSZ) is a serialization and Merkleization scheme designed specifically for Ethereum's Beacon Chain. SSZ replaces the [RLP serialization](/docs/wiki/EL/RLP.md) used on the execution layer (EL) everywhere across the consensus layer (CL) except the [peer discovery protocol](https://github.com/ethereum/devp2p). Its development and adoption are aimed at enhancing the efficiency, security, and scalability of Ethereum's CL.

## How SSZ Works - Basic Types

Here’s how SSZ handles the serialization and deserialization of the basic types:

```mermaid
flowchart TD
    A[Start Serialization] --> B[Choose Data Type]
    B --> C[Unsigned Integer]
    B --> D[Boolean]
    
    C --> E[Convert Integer to \nLittle-Endian Byte Array]
    E --> F[Serialized Output for Integer]
    
    D --> G["Convert Boolean to Byte \n(True to 0x01, False to 0x00)"]
    G --> H[Serialized Output for Boolean]
    
    classDef startEnd fill:#f9f,stroke:#333,stroke-width:4px;
    class A startEnd;
    classDef process fill:#ccf,stroke:#f66,stroke-width:2px;
    class B,C,D,E,G process;
    classDef output fill:#cfc,stroke:#393,stroke-width:2px;
    class F,H output;
```

_Figure: Serialization Process for Basic Types._


```mermaid
flowchart TD
    A[Start Deserialization] --> B[Determine Data Type]
    B --> C[Unsigned Integer]
    B --> D[Boolean]
    
    C --> E[Read Little-Endian Byte Array]
    E --> F[Reconstruct Original Integer Value]
    F --> G[Deserialized Integer Output]
    
    D --> H[Read Byte]
    H --> I["Translate Byte to Boolean \n(0x01 to True, 0x00 to False)"]
    I --> J[Deserialized Boolean Output]
    
    classDef startEnd fill:#f9f,stroke:#333,stroke-width:4px;
    class A startEnd;
    classDef process fill:#ccf,stroke:#f66,stroke-width:2px;
    class B,C,D,E,H,I process;
    classDef output fill:#cfc,stroke:#393,stroke-width:2px;
    class G,J output;
```

_Figure: Deserialization Process for Basic Types._

### Unsigned Integers

Unsigned integers (`uintN`) in SSZ are denoted where `N` can be any of 8, 16, 32, 64, 128, or 256 bits. These integers are serialized directly to their little-endian byte representation, which is a form well-suited for most modern computer architectures and facilitates easier manipulation at the byte level.

**Serialization Process for Unsigned Integers:**

1. **Input**: Take an unsigned integer of type `uintN`.
2. **Convert to Bytes**: Convert the integer into a byte array of length `N/8`. For instance, `uint16` represents 2 bytes.
3. **Apply Little-Endian Format**: Arrange the bytes in little-endian order, where the least significant byte is stored first.
4. **Output**: The resulting byte array is the serialized form of the integer.

**Example:**
- Integer `1025` as `uint16` would be serialized to `01 04` in hexadecimal. First, convert `1025` to hex which gives `0x0401`. In little-endian format, the least significant byte (LSB) comes first. So, `0x0401` in little-endian is `01 04`. The byte array `[01, 04]` is the serialized output.

**Deserialization Process for Unsigned Integers:**

1. **Input**: Read the byte array representing a serialized `uintN`.
2. **Read Little-Endian Bytes**: Interpret the bytes in little-endian order to reconstruct the integer value.
3. **Output**: Convert the byte array back into the integer.

**Example:**
- Byte array `01 04` (in hex) is deserialized to the integer `1025`. Read the first byte `01` as the lower part and `04` as the higher part of the integer. It translates back to `0401` in hex when reassembled in big-endian format for human readability, which equals 1025 in decimal.

### Booleans

Booleans in SSZ are quite straightforward, with each boolean represented as a single byte.

**Serialization Process for Booleans:**

1. **Input**: Take a boolean value (`True` or `False`).
2. **Convert to Byte**: 
   - If the boolean is `True`, serialize it as `01` (in hex).
   - If the boolean is `False`, serialize it as `00`.
3. **Output**: The resulting single byte is the serialized form of the boolean.

**Example:**
- `True` becomes `01`.
- `False` becomes `00`.

**Deserialization Process for Booleans:**

1. **Input**: Read a single byte.
2. **Interpret the Byte**: 
   - A byte of `01` indicates `True`.
   - A byte of `00` indicates `False`.
3. **Output**: The boolean value corresponding to the byte.

**Example:**
- Byte `01` is deserialized to `True`.
- Byte `00` is deserialized to `False`.

We can run SSZ serialization and deserialization commands using the python Eth2 spec as per the [instructions](https://eth2book.info/capella/appendices/running/) and verify the above byte arrays.

```python
>>> from eth2spec.utils.ssz.ssz_typing import uint64, boolean
# Serializing 
>>> uint64(1025).encode_bytes().hex()
'0104000000000000'
>>> boolean(True).encode_bytes().hex()
'01'
>>> boolean(False).encode_bytes().hex()
'00' 

# Deserializing 
>>> print(uint64.decode_bytes(bytes.fromhex('0104000000000000')))
1025
>>> print(boolean.decode_bytes(bytes.fromhex('01')))
1
>>> print(boolean.decode_bytes(bytes.fromhex('00')))
0
```

## How SSZ Works - Composite Types

### Vectors

Vectors in SSZ are used to handle fixed-length collections of homogeneous elements. Here’s a detailed breakdown of how SSZ handles the serialization and deserialization of vectors.

**SSZ Serialization for Vectors**

```mermaid
flowchart TD
    A[Start Serialization] --> B[Define Vector with Type and Length]
    B --> C[Serialize Each Element]
    C --> D["Convert Each Element to \nByte Array (Little-Endian)"]
    D --> E[Concatenate All Byte Arrays]
    E --> F[Output Serialized Vector]
    
    classDef startEnd fill:#f9f,stroke:#333,stroke-width:4px;
    class A startEnd;
    classDef process fill:#ccf,stroke:#f66,stroke-width:2px;
    class B,C,D,E process;
    classDef output fill:#cfc,stroke:#393,stroke-width:2px;
    class F output;
```

_Figure: SSZ Serialization for Vectors._


1. **Fixed-Length Definition**: Vectors are defined with a specific length and type of elements they can hold, such as `Vector[uint64, 4]` for a vector containing four 64-bit unsigned integers.

2. **Element Serialization**:
   - Each element in the vector is serialized independently according to its type. 
   - For basic types like integers or booleans, this means converting each element to its byte representation. 
   - If the elements are composite types, each element is serialized according to its specific serialization rules.

3. **Concatenation**:
   - The serialized outputs of each element are concatenated in the order they appear in the vector. 
   - Since the length of the vector and the size of each element are known and fixed, no additional metadata (like length prefixes) is needed in the serialized output.

**Example:**
For a `Vector[uint64, 3]` with the elements `[256, 512, 768]`, each element is 64 bits or 8 bytes long. The serialization would proceed as follows:

1. **Convert Each Integer to Little-Endian Byte Array**:
   - `256` as `uint64` becomes `00 01 00 00 00 00 00 00`.
   - `512` as `uint64` becomes `00 02 00 00 00 00 00 00`.
   - `768` as `uint64` becomes `00 03 00 00 00 00 00 00`.

2. **Concatenate These Byte Arrays**:
   - The resulting concatenated byte array will be `00 01 00 00 00 00 00 00 00 02 00 00 00 00 00 00 00 03 00 00 00 00 00 00`.

**Serialized Output**:
   - `00 01 00 00 00 00 00 00 00 02 00 00 00 00 00 00 00 03 00 00 00 00 00 00`.


**SSZ Deserialization for Vectors**

```mermaid
flowchart TD
    A[Start Deserialization] --> B[Receive Serialized Byte Stream]
    B --> C[Identify and Split Byte Stream \nBased on Element Size]
    C --> D[Deserialize Each Byte Segment\n to Its Original Type]
    D --> E[Reassemble Elements into Vector]
    E --> F[Output Deserialized Vector]
    
    classDef startEnd fill:#f9f,stroke:#333,stroke-width:4px;
    class A startEnd;
    classDef process fill:#ccf,stroke:#f66,stroke-width:2px;
    class B,C,D,E process;
    classDef output fill:#cfc,stroke:#393,stroke-width:2px;
    class F output;
```

_Figure: SSZ Deserialization for Vectors._


1. **Fixed-Length Utilization**:
   - The deserializer uses the predefined length and type of the vector to parse the serialized data.
   - It knows exactly how many bytes each element takes and how many elements are in the vector.

2. **Element Deserialization**:
   - The byte stream is split into segments corresponding to the size of each element.
   - Each segment is deserialized independently according to the type of elements in the vector.

3. **Reconstruction**:
   - The elements are reconstructed into their original form (e.g., converting byte arrays back into integers or other specified types).
   - These elements are then aggregated to reform the original vector.

**Example:**
Given the serialized data for a `Vector[uint64, 3]`:
- Serialized Byte Array: `00 01 00 00 00 00 00 00 00 02 00 00 00 00 00 00 00 03 00 00 00 00 00 00`.

1. **Parse the Data into Segments**:
   - Each segment consists of 8 bytes.
   - First segment: `00 01 00 00 00 00 00 00` → Represents the integer 256.
   - Second segment: `00 02 00 00 00 00 00 00` → Represents the integer 512.
   - Third segment: `00 03 00 00 00 00 00 00` → Represents the integer 768.

2. **Convert Each Segment from a Little-Endian Byte Array Back to an Integer**:
   - Using little-endian format, each byte array is read and converted back into the respective `uint64` integer.

3. **Reconstruction**:
   - The reconstructed vector is `[256, 512, 768]`.

We can run and verify it in python like below:

```python
>>> from eth2spec.utils.ssz.ssz_typing import uint8, uint16, Vector
>>> Vector[uint16, 3](256, 512, 768).encode_bytes().hex()
'000100000000000000020000000000000003000000000000'
>>> print(Vector[uint64, 3].decode_bytes(bytes.fromhex('000100000000000000020000000000000003000000000000')))
Vector[uint64, 3]<<len=3>>(256, 512, 768)
>>> 

```

### Lists

Lists in SSZ are crucial for managing variable-length collections of homogeneous elements within a specified maximum length (`N`). This flexibility allows for dynamic management of the data structures such as transaction sets or variable state components, adapting to the changing needs of the network.

**SSZ Serialization for Lists**

```mermaid
flowchart TD
    A[Start Serialization] --> B[Define List with Type and Max Length]
    B --> C[Serialize Each Element]
    C --> D["Convert Each Element to \nByte Array (Little-Endian)"]
    D --> E[Concatenate All Byte Arrays]
    E --> F[Optional: Include Length Metadata]
    F --> G[Output Serialized List]
    
    classDef startEnd fill:#f9f,stroke:#333,stroke-width:4px;
    class A startEnd;
    classDef process fill:#ccf,stroke:#f66,stroke-width:2px;
    class B,C,D,E,F process;
    classDef output fill:#cfc,stroke:#393,stroke-width:2px;
    class G output;
```

_Figure: SSZ Serialization for Lists._

1. **Define the List**: Lists in SSZ are defined with a specific element type and a maximum length, noted as `List[type, N]`. This definition not only constrains the list's maximum capacity but also informs how elements should be serialized.

2. **Element Serialization**:
   - Each element in the list is serialized based on its type. For `uint64` elements, the serialization process involves converting each integer into a byte array.

3. **Concatenate Serialized Elements**:
   - The outputs of the serialized elements are concatenated sequentially. The total length of the serialized data varies depending on the number of elements present at the time of serialization.

4. **Include Length Metadata (Optional)**:
   - Depending on the implementation requirements, the length of the list might be explicitly included at the start of the serialized data to aid in parsing and validation during deserialization.

**Example**:
For a `List[uint64, 5]` containing the elements `[1024, 2048, 3072]`, the serialization process would involve:
- Converting each integer to a byte array in little-endian format: `00 04 00 00 00 00 00 00`, `00 08 00 00 00 00 00 00`, `00 0C 00 00 00 00 00 00`.
- Concatenating these arrays results in: `00 04 00 00 00 00 00 00 00 08 00 00 00 00 00 00 00 0C 00 00 00 00 00 00`.

**SSZ Deserialization for Lists**

```mermaid
flowchart TD
    A[Start Deserialization] --> B[Receive Serialized Byte Stream]
    B --> C["Identify and Split Byte Stream Based \n on Element Size (8 bytes for uint64)"]
    C --> D[Deserialize Each Byte Segment to uint64]
    D --> E[Reassemble Elements into List]
    E --> F[Output Deserialized List]
    
    classDef startEnd fill:#f9f,stroke:#333,stroke-width:4px;
    class A startEnd;
    classDef process fill:#ccf,stroke:#f66,stroke-width:2px;
    class B,C,D,E process;
    classDef output fill:#cfc,stroke:#393,stroke-width:2px;
    class F output;
```

_Figure: SSZ Deserialization for Lists._

1. **Receive Serialized Data**: The serialized byte stream for the list is the input, containing sequences of byte arrays for each element.

2. **Parse and Deserialize Each Element**:
   - Based on the element type, say `uint64`, parse the serialized stream into 8-byte segments.
   - Convert each byte array from little-endian format back into a `uint64`.

3. **Reassemble the List**:
   - The deserialized elements are reassembled to recreate the original list.

**Example**:
Given the serialized data `00 04 00 00 00 00 00 00 00 08 00 00 00 00 00 00 00 0C 00 00 00 00 00 00` for a `List[uint64, 5]`:
- Split the data into segments of 8 bytes: `00 04 00 00 00 00 00 00`, `00 08 00 00 00 00 00 00`, `00 0C 00 00 00 00 00 00`.
- Convert each segment from little-endian to integers: `1024`, `2048`, `3072`.
- The reconstructed list is `[1024, 2048, 3072]`.

We can run and verify the SSZ for the above example as below:

```python
>>> from eth2spec.utils.ssz.ssz_typing import uint8, List, Vector
>>> List[uint64, 5](1024, 2048, 3072).encode_bytes().hex()
'00040000000000000008000000000000000c000000000000'
>>> Vector[uint64, 3](1024, 2048, 3072).encode_bytes().hex()
'00040000000000000008000000000000000c000000000000'
>>> print(List[uint64, 5].decode_bytes(bytes.fromhex('00040000000000000008000000000000000c000000000000')))
List[uint64, 5]<<len=3>>(1024, 2048, 3072)
>>> 
```

Lists are variable sized objects in SSZ they are encoded differently from fixed sized vectors when contained within another object, so there is a small overhead. For example, below `Alice` and `Bob` objects have different encoding.

```python
>>> from eth2spec.utils.ssz.ssz_typing import uint8, Vector, List, Container
>>> class Alice(Container):
...     x: List[uint8, 3] # Variable sized
>>> class Bob(Container):
...     x: Vector[uint8, 3] # Fixed sized
>>> Alice(x = [1, 2, 3]).encode_bytes().hex()
'04000000010203'
>>> Bob(x = [1, 2, 3]).encode_bytes().hex()
'010203'
>>> 
```

### Bitvectors

Bitvectors in SSZ are used to manage fixed-length sequences of boolean values, typically represented as bits. This data structure is particularly efficient for compactly storing binary data or flags, which are common in Ethereum applications for indicating state conditions, permissions, or other binary settings.

**SSZ Serialization for Bitvectors**

```mermaid
flowchart TD
    A[Start Serialization] --> B[Define Bitvector of Size N]
    B --> C[Pack Bits into Bytes]
    C --> D[Bits from LSB to\n MSB within each byte]
    D --> E[Add Padding if N % 8 != 0]
    E --> F[Output Serialized Byte Array]
    
    classDef startEnd fill:#f9f,stroke:#333,stroke-width:4px;
    class A startEnd;
    classDef process fill:#ccf,stroke:#f66,stroke-width:2px;
    class B,C,D,E process;
    classDef output fill:#cfc,stroke:#393,stroke-width:2px;
    class F output;
```

_Figure: SSZ Serialization for Bitvectors._

1. **Define the Bitvector**: A bitvector in SSZ is defined by its length `N`, which specifies the number of bits. For example, `Bitvector[256]` means a bitvector that contains 256 bits.

2. **Convert Bits to Bytes**:
   - Each bit in the bitvector represents a boolean value, where `0` corresponds to `False` and `1` to `True`.
   - These bits are packed into bytes, with the least significant bit (LSB) first within each byte. This means the first bit in the bitvector corresponds to the LSB of the first byte.

3. **Byte Array Formation**:
   - The bits are serialized into a byte array by packing 8 bits into each byte until all bits are accounted for.
   - If `N` is not a multiple of 8, the last byte will contain fewer than 8 bits of data, padded with zeros at the most significant bit positions.

**Example**:
For a `Bitvector[10]` with the pattern `1011010010`:
- The first 8 bits (`10110100`) form the first byte.
- The remaining 2 bits (`10`) are padded with six zeros to form the second byte: `10000000`.
- The serialized output is `B4 80` in hexadecimal.

**SSZ Deserialization for Bitvectors**

```mermaid
flowchart TD
    A[Start Deserialization] --> B[Receive Serialized Byte Array]
    B --> C[Read Each Byte]
    C --> D[Convert Bytes to Bits]
    D --> E[Respect LSB to MSB \nOrder in Each Byte]
    E --> F[Discard Padding if Present]
    F --> G[Reconstruct Bitvector]
    G --> H[Output Deserialized Bitvector]
    
    classDef startEnd fill:#f9f,stroke:#333,stroke-width:4px;
    class A startEnd;
    classDef process fill:#ccf,stroke:#f66,stroke-width:2px;
    class B,C,D,E,F,G process;
    classDef output fill:#cfc,stroke:#393,stroke-width:2px;
    class H output;
```

_Figure: SSZ Deserialization for Bitvectors._

1. **Read Serialized Byte Array**: Start with a byte array that encodes the bitvector.

2. **Extract Bits from Bytes**:
   - Convert each byte back into bits. Remember, the bits are stored LSB first within each byte.
   - If the bitvector's length `N` is not a multiple of 8, discard the extraneous padding bits in the final byte.

3. **Reconstruct the Bitvector**:
   - Reassemble the extracted bits into the original bitvector format, adhering to the specified length `N`.

**Example**:
Given the serialized data `B4 80` for a `Bitvector[10]`:
- Convert `B4` (`10110100` in binary) and `80` (`10000000` in binary) back into bits.
- Extract the first 10 bits from the binary sequence: `1011010010`.
- The reconstructed bitvector is `1011010010`.

You can run and verify it in python as below:

```python
>>> from eth2spec.utils.ssz.ssz_typing import Bitvector
>>> Bitvector[8](0,0,1,0,1,1,0,1).encode_bytes().hex()
'b4'
>>> Bitvector[8](0,0,0,0,0,0,0,1).encode_bytes().hex()
'80'
```

Note that, functionally we could use either `Vector[boolean, N]` or `Bitvector[N]` to represent a list of bits. However, the latter will have a serialization up to eight times shorter in practice since the former will use a whole byte per bit.

```python
>>> from eth2spec.utils.ssz.ssz_typing import Vector, Bitvector, boolean
>>> Bitvector[5](1,0,1,0,1).encode_bytes().hex()
'15'
>>> Vector[boolean,5](1,0,1,0,1).encode_bytes().hex()
'0100010001'
```

### Bitlists

Bitlists in SSZ are similar to bitvectors but are designed to handle variable-length sequences of boolean values with a specified maximum length (`N`). 

**SSZ Serialization for Bitlists**

```mermaid
flowchart TD
    A[Start Serialization] --> B[Define Bitlist of Size N]
    B --> C[Pack Bits into Bytes]
    C --> D[Add Sentinel Bit]
    D --> E[Pad Final Byte if Necessary]
    E --> F[Output Serialized Byte Array]
    
    classDef startEnd fill:#f9f,stroke:#333,stroke-width:4px;
    class A startEnd;
    classDef process fill:#ccf,stroke:#f66,stroke-width:2px;
    class B,C,D,E process;
    classDef output fill:#cfc,stroke:#393,stroke-width:2px;
    class F output;
```

_Figure: SSZ Serialization for Bitlists._


1. **Define the Bitlist**: A bitlist is defined by its maximum length `N`, which determines the upper bound of bits that can be included. The actual number of bits, however, can be less than `N`.

2. **Pack Bits into Bytes**:
   - Each bit in the bitlist represents a boolean value, where `0` corresponds to `False` and `1` to `True`.
   - These bits are serialized into a byte array, packed from the LSB to the MSB within each byte, similar to bitvectors.

3. **Add Sentinel Bit**:
   - To mark the end of the bitlist and distinguish its actual length from its maximum capacity, a sentinel bit (`1`) is added to the end of the bit sequence. This is crucial to ensure that the deserialization process accurately identifies the length of the bitlist.

4. **Byte Array Formation and Padding**:
   - After including the sentinel bit, the bits are packed into bytes, with any necessary padding applied to the last byte to complete it if the total number of bits (including the sentinel) does not divide evenly by 8.


**SSZ Deserialization for Bitlists**

```mermaid
flowchart TD
    A[Start Deserialization] --> B[Receive Serialized Byte Array]
    B --> C[Convert Bytes to Bits]
    C --> D[Identify and Remove Sentinel Bit]
    D --> E[Remove Padding Bits]
    E --> F[Reconstruct Original Bitlist]
    F --> G[Output Deserialized Bitlist]
    
    classDef startEnd fill:#f9f,stroke:#333,stroke-width:4px;
    class A startEnd;
    classDef process fill:#ccf,stroke:#f66,stroke-width:2px;
    class B,C,D,E,F process;
    classDef output fill:#cfc,stroke:#393,stroke-width:2px;
    class G output;
```

_Figure: SSZ Deserialization for Bitlists._

1. **Receive Serialized Byte Array**: Start with the byte array that encodes the bitlist, including the sentinel bit.

2. **Extract Bits from Bytes**:
   - Convert each byte back into bits, respecting the order (LSB to MSB).
   - Continue this process for each byte in the serialized data.

3. **Identify and Remove the Sentinel Bit**:
   - As bits are extracted, locate the first `1` (sentinel bit) from the end of the bit sequence to determine the actual end of the bitlist data.
   - All bits following the sentinel bit are disregarded as padding.

4. **Reconstruct the Bitlist**:
   - Reassemble the extracted bits (excluding the sentinel bit and any padding) into the original bitlist format.

You can run the encoding of Bitlist like below:

```python
>>> from eth2spec.utils.ssz.ssz_typing import Bitlist
>>> Bitlist[100](0,0,0).encode_bytes().hex()
'08'
```

As a consequence of the sentinel, we require an extra byte to serialize a bitlist if its actual length is a multiple of eight (irrespective of the maximum length `N`). This is not the case for a bitvector.

```python
>>> Bitlist[8](0,0,0,0,0,0,0,0).encode_bytes().hex()
'0001'
>>> Bitvector[8](0,0,0,0,0,0,0,0).encode_bytes().hex()
'00'
```


## Fixed VS Variable Length Types


## SSZ Tools

## Resources
- [Simple serialize](https://ethereum.org/en/developers/docs/data-structures-and-encoding/ssz/)
- [SSZ specs](https://github.com/ethereum/consensus-specs/blob/dev/ssz/simple-serialize.md)
- [eth2book - SSZ](https://eth2book.info/capella/part2/building_blocks/ssz/#ssz-simple-serialize)
- [Go Lessons from Writing a Serialization Library for Ethereum](https://rauljordan.com/go-lessons-from-writing-a-serialization-library-for-ethereum/)
- [Interactive SSZ serializer/deserializer](https://www.ssz.dev/)