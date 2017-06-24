---
layout: post
title:  "Creating a disassembler plugin for Binary Ninja"
date:   2017-06-23 22:30:47 +1000
categories: jekyll update
---
# Intro
Recently i've been researching USB flash drives and the [BadUSB](https://opensource.srlabs.de/projects/badusb) class of attacks. 
I initially have been using the IDAPro disassembler for this research which provides a lot of functionality out of the box but costs $$$. I'd recently purchased a copy of [Binary Ninja](https://binary.ninja/) as it offers a heap of functionality for a lot less money, looks good and provides an integrated API for creating your own plugins.
Another thing that Binary Ninja offers is a [community repository](https://github.com/Vector35/community-plugins) for Plugins.
Since the 8051 is not supported out-of-the-box by Binja; i decided to wack together a disassembler and lifter for it. This is a great architecture to do this for as it is incredibly simple and straightforward.

## About the 8051
The 8051 originates from 1980 as one of Intels early microcontrollers. It most likely is the most used architecture in the world given its prevalence in basic embedded systems, i'd guess that at least 90% of all usb flash drives has an 8051 based controller IC.
Why it has become so prevalent is due to it being simple, well-supported, cheap and patent incumbered.
[8051 History](http://www.efton.sk/t0t1/history8051.pdf)

# Researching the architecture
Obtain all the references you can to the architecture you are implemented.
You are mainly looking for details of the 

[Keil 8051 Instruction Set Manual](http://www.keil.com/support/man/docs/is51/)
[8051 Instruction Set](https://www.win.tue.nl/~aeb/comp/8051/set8051.html)

#Define the registers of chip and the widths.
```
Registers = [
    'SP',       #stack pointer 0x81
    'DPTR',       #data pointer  0x82-83
    'PWS',      #Program Status Word
    'A',        #accumulator A 0xE0
    'B',        #B register 0xF0
    'R0',       #Reg Bank 0,1,2,3
    'R1',
    'R2',
    'R3',
    'R4',
    'R5',
    'R6',
    'R7'
]
```
#Define the addressing modes of the device*
Here we are defining how operands are addressed.
Most processors will use these basic addressing modes:
+Register
+Immediate
+Memory

These effect how the disassembler will render and translate instructions. The addressing mode tells the processor how to access and write data for that operation. This can be simple memory and register accesses (MOV Reg, Memory) , to modes that perform indirect memory accesses and shift data (e.g LDMIA R0!, { R5-R8 }).
Each processor will support a mix of different modes depending on its architecture and memory layout. These will also be syntaxed differently depending on the chip, and the assembler syntax (e.g intel or ATT syntax etc)

For the 8051 i identified the following modes
##IMMEDIATE_MODE - MOV A,#20h
Instructions that use immediate data that is encoded in the raw binary.
the "#" indicates that the 0x20 value is to be written to register 'A'. here the 0x20 is encoded in the raw binary (e520 = MOV A, #20h).

##REGISTER_MODE = 1 #Register Addressing MOV A, R0
Register-Register transfer modes. Here we are just using the internal registers to perform operations.

##DIRECT_MODE = 2 #Direct Addressing   MOV A,30h
Direct addressing mode - Here we directly accessing memory of the device. 
In this example the value at memory location 0x30 is moved into register 'A'.

##REG_INDIRECT_MODE = 3   #Indirect Addressing MOV A,@R0
Here we are starting to read data indirectly, or to put it another way use pointers.
We have a value in register 'R0' which represents an address; and we want the value at an address to be put into register 'A'.
i.e.
`A = R0[0]
The "@" symbol in this case can be thought of as the "*" operator in C

##CODE_MODE = 5       #Code Indirect   MOVC A,@A+DPTR
Another mode is offset indirect addressessing. 
Here we add the values in the 'A' register and the 'DPTR' to get the memory address to read.
We then copy the value in this address to register 'A'.
so in pseudo C:
A = *(A+DPTR)

Below is the table created - many of these are present for assisting with the formatting of the disassembly
```
IMMEDIATE_MODE = 0  #Immediate Addressing    MOV A,#20h
REGISTER_MODE = 1 #Register Addressing MOV A, R0
DIRECT_MODE = 2 #Direct Addressing   MOV A,30h
REG_INDIRECT_MODE = 3   #Indirect Addressing MOV A,@R0
INDEXED_MODE = 4   #External Direct MOVX A,@DPTR
CODE_MODE = 5       #Code Indirect   MOVC A,@A+DPTR
BIT_ADDRESS_MODE = 6 #SETB bit_addr
BIT_INDEXED_MODE = 7 #jnb ACC.0, code_addr
BIT_CLEAR_MODE = 8 #MOV bit addr,C
IMMEDIATE_OFFSET_MODE = 9 #CJNE A,#data,reladdr
DIRECT_OFFSET_MODE = 10 #CJNE A,imm_data,reladdr
```
#Step 4: Define the Operand Tokens
Now we have an idea of what each mode looks like lets get to telling Binary Ninja how to render them.
We define a Map for each mode - which is addressed using the constants we made above
Each entry is defined as a standard lambda function with the 3 variables provided when we call it. (this allows for code reuse later)
Here are where all the InstructionTokens are defined https://api.binary.ninja/binaryninja.function.InstructionTextToken.html

The ones we are utilising here are
RegisterToken(reg) - Token represents a register.
TextToken(text) - Treat the token as a text value
PossibleAddressToken(text, address) - Token is most likely an address so Binary Ninja will render it and link to memory (so you can double click and go to that address)

So for Immediate addressing we translate "#20h" into:
```
lambda value, _, _ : [ #IMMEDIATE
        InstructionTextToken(TextToken,'#'), # "#"
        InstructionTextToken(PossibleAddressToken, hex(value), value) # tell binja its an address and print the hex value
        #InstructionTextToken(InstructionTextTokenType.TextToken,'h') # append an "h" for hex
    ],
Register:
lambda reg, _, _: [ #REGISTER
        InstructionTextToken(RegisterToken,reg)
    ],
Direct:
lambda addr, _, _: [ #DIRECT
        InstructionTextToken(PossibleAddressToken, '0x{:02X}'.format(addr), addr) #format memory as a hex value with 0x appended
    ],

Indirect:
lambda reg, _, _: [ #INDIRECT
        InstructionTextToken(TextToken,'@'),        #append a "@"
        InstructionTextToken(RegisterToken, reg)    #and the register
    ],

Indirect offset:
lambda reg, value, addr: [ #code mode
        InstructionTextToken(TextToken, '@'),               #append a "@"
        InstructionTextToken(RegisterToken, reg),           #and the register
        InstructionTextToken(OperandSeperatorToken, '+'),   #append a "+"
        InstructionTextToken(RegisterToken, value)          #and the register
    ],
```
Heres the finalised Map:
```
OperandTokenGen = [
    lambda reg, value, addr : [ #IMMEDIATE
        InstructionTextToken(TextToken,'#'),
        InstructionTextToken(PossibleAddressToken, hex(reg), reg)
        #InstructionTextToken(InstructionTextTokenType.TextToken,'h')
    ],
    lambda reg, value, addr: [ #REGISTER
        InstructionTextToken(RegisterToken,reg)
    ],
    lambda reg, value, addr: [ #DIRECT
        InstructionTextToken(PossibleAddressToken, '0x{:02X}'.format(reg), reg)
    ],
    lambda reg, value, addr: [ #INDIRECT
        InstructionTextToken(TextToken,'@'),
        InstructionTextToken(RegisterToken, reg)
    ],
    lambda reg, value, addr: [ #INDEXED
        InstructionTextToken(TextToken,'@'),
        InstructionTextToken(RegisterToken, reg)
    ],
    lambda reg, value, addr: [ #code mode
        InstructionTextToken(TextToken, '@'),
        InstructionTextToken(RegisterToken, reg),
        InstructionTextToken(OperandSeperatorToken, '+'),
        InstructionTextToken(RegisterToken, value)
    ],
    lambda reg, value, addr: [ #bit address mode
        InstructionTextToken(PossibleAddressToken, convertBitField(reg), reg)
    ],
    lambda reg, value, addr: [ #jnb ACC.0, code_addr
        InstructionTextToken(RegisterToken, reglookup[reg]),
        InstructionTextToken(TextToken,'.'),
        InstructionTextToken(TextToken, '{}'.format(1 >> value))
    ],
    lambda reg, value, addr: [ # BIT_CLEAR_MODE "C"
        InstructionTextToken(TextToken,'C'),
    ],
    lambda reg, value, addr: [ #IMMEDIATE_OFFSET_MODE = 9 CJNE A,#data,reladdr
        InstructionTextToken(TextToken, '#'),
        InstructionTextToken(IntegerToken,hex(reg), reg),
        #InstructionTextToken(PossibleAddressToken,hex(reg), reg),
        InstructionTextToken(TextToken, ','),
        InstructionTextToken(PossibleAddressToken,hex(value), value)
    ],
    lambda reg, value, addr: [ #DIRECT_OFFSET_MODE = 9 CJNE A,#data,reladdr
        InstructionTextToken(PossibleAddressToken, hex(reg), reg),
        InstructionTextToken(TextToken, ','),
        InstructionTextToken(PossibleAddressToken, hex(value), value)
    ]
]
```
#Implementing the architecture
Now we are ready to glue everything together for the disassembler!
Here we use the [Architecture class](https://api.binary.ninja/binaryninja.architecture-module.html) to define a new architecture.
```
class i8051(Architecture):
    name = 'i8051' #name of architecture 
    address_size = 2 #bus size - in an 8051 we have a 16bit bus
    default_int_size = 1 #int size - 8051 ints are 8 bits 
    max_instr_length = 3 #max length of an instruction for 8051 is 3 bytes
```
next we define our register Map.
```
RegisterInfo(name, length in bytes)
    regs = {
        'A': RegisterInfo(  'A', 1),
        'B': RegisterInfo(  'B', 1),
        'DPTR': RegisterInfo( 'DPTR', 2),
        'PSW': RegisterInfo('PSW', 1),
        'R0': RegisterInfo( 'R0', 1),
        'R1': RegisterInfo( 'R1', 1),
        'R2': RegisterInfo( 'R2', 1),
        'R3': RegisterInfo( 'R3', 1),
        'R4': RegisterInfo( 'R4', 1),
        'R5': RegisterInfo( 'R5', 1),
        'R6': RegisterInfo( 'R6', 1),
        'R7': RegisterInfo( 'R7', 1),
        'SP': RegisterInfo( 'SP', 2)
    }
```
Then the function that handles the decoding of instructions:
```
def decode_instruction(self, data, addr):
    error_value = (None, None, None, None, None, None, None, None, None)
    instruction = struct.unpack('<B', data[0])[0] #we unpack each instruction, each 8051 instruction is 1 byte long
    #and start processing them
    if instruction == 0x00: 
        return 'NOP', None, None, None, None, None, 1, None, None
    elif instruction == 0x01: #so here is the AJMP instruction
            (high_address,low_address) = struct.unpack('>BB', data[0:2]) #its 2 bytes long, with the address packed into both bytes
            direct_addr = ((high_address & 0xe0 >> 5) << 8) | low_address #we decode the address
            return_addr = ((addr + 2) & 0xF800) + direct_addr
            return 'AJMP', 1, None, DIRECT_MODE, None, return_addr, 2, None, None #then return the result instr, width, src_op, dst_op, src, dst,length, src_val, dst_val
```    
here is the code to decode a MOV A, #30h instruction (0x7430)
```
elif instruction == 0x74:
    imm_data = struct.unpack('>B', data[1])[0]
    return 'MOV',   1, IMMEDIATE_MODE, REGISTER_MODE, imm_data, 'A', 2, None, None
```
So the source is the immediate value 30 - IMMEDIATE MODE
and dest is register A - REGISTER MODE
width is 1 byte (single byte registers)
length is 2 bytes.

So we do this for all supported instructions (256 on an 8051)

Next up is creating the text for the disassembled instruction
```
def perform_get_instruction_text(self, data, addr):
        #decode the instruction
        (instr, width, src_operand, dst_operand, src, dst, length, src_value, dst_value) = self.decode_instruction(data, addr)
        if instr is None:
            return None
        #create the token list to return
        tokens = []

        #create the instruction text token
        instruction_text = instr
        tokens = [
            InstructionTextToken(InstructionTextTokenType.TextToken, '{:7s}'.format(instruction_text)) #format to pad with 7 spaces
        ]
        #add each token from the operand list we created before
        if dst_operand != None: #get the first token and add (destination)
            tokens += OperandTokenGen[dst_operand](dst, dst_value, addr)
        if dst_operand != None and src_operand != None: #if theres a src then add a ','
            tokens += [InstructionTextToken(TextToken, ',')]
        if src_operand != None: #append the src token
            tokens += OperandTokenGen[src_operand](src, src_value, addr)

        return tokens, length #return the token list
```
