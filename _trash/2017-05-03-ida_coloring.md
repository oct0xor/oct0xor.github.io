---
layout: page
title: Advanced Ida Pro Instruction Highlighting
---

<div align="center">
    <img src ="/assets/ida_color0.png"/>
</div>

> The actions described here can be considered as violation of EULA, provided for educational purposes only.

Ida Pro's adjustable color scheme is a real life saver. But it doesnt allow you to specify color for specific instructions. With right configuration it allows you to quickly see function calls in your code, but it doesnt work with indirect branches.

I was living fine without it, but missed this feature from OllyDbg.

There is workaround to use idc function `SetColor` to accomplish this. But to be honest it sucks. When you already have enough of colors your graph becomes a mess.

Thats why I took a quick look at how Ida Pro does coloring.

Standard color scheme is stored inside executable and has `IDACLR` magic. It gets copied in memory and used from there or loaded from other places. This information was enough to find how its handled next.

`get_color_sub_539020` - Gets RGB values from color scheme by id and returns constructed QColor.

<div align="center">
    <img src ="/assets/ida_color5.png"/>
</div>

Then its used in function that does coloring of all text in graph view.

<div align="center">
    <img src ="/assets/ida_color6.png"/>
</div>

Pointer to structure that holds current processed text is stored on stack. So actually we can hook `get_color_sub_539020`, check id (mnemonic = 5) compare text of processed mnemonic with our own and set another color.

Thats done with a few lines of asm:

```
main:

    cmp     cl, 5                       # check if id == 5 (mnemonic)
    jne     done

    mov     eax, [esp + 0x50]           # get ptr to stru that holds text from stack
    mov     eax, [eax + 1]              # get dword with mnemonic

    cmp     eax, 0x6C6C6163             # if dword == "call"
    jne     done

    mov     cl, 0x17                    # id of new color
    movzx   eax, WORD PTR [0x6C3378]    # instruction that we hooked
    jmp     0x00539032                  # return from payload

done:

    movzx   eax, WORD PTR [0x6C3378]
    jmp     0x00539032
```

and it does this:

<div align="center">
    <img src ="/assets/ida_color7.png"/>
</div>

My python script that adds this hook can be found [here](https://github.com/oct0xor/ida_pro_graph_styling/blob/master/inject_66.py).

(Important note: you might need to fix it for your IDA, all binaries are different its part of watermarking.)

Ok, so it will work for x86 and some others, what about all architectures? Actually its still easy to do with little more complicated payload.

From idp.hpp:

```
//-----------------------------------------------------------------------
/// Internal representation of processor instructions.
/// Definition of all internal instructions are kept in special arrays.
/// One of such arrays describes instruction names and features.
struct instruc_t
{
  const char *name;       ///< instruction name
  uint32 feature;         ///< combination of \ref CF_
/// \defgroup CF_ Instruction feature bits
/// Used by instruc_t::feature
//@{
#define CF_STOP 0x00001   ///< Instruction doesn't pass execution to the
                          ///< next instruction
#define CF_CALL 0x00002   ///< CALL instruction (should make a procedure here)
#define CF_CHG1 0x00004   ///< The instruction modifies the first operand
#define CF_CHG2 0x00008   ///< The instruction modifies the second operand
#define CF_CHG3 0x00010   ///< The instruction modifies the third operand
#define CF_CHG4 0x00020   ///< The instruction modifies 4 operand
#define CF_CHG5 0x00040   ///< The instruction modifies 5 operand
#define CF_CHG6 0x00080   ///< The instruction modifies 6 operand
#define CF_USE1 0x00100   ///< The instruction uses value of the first operand
#define CF_USE2 0x00200   ///< The instruction uses value of the second operand
#define CF_USE3 0x00400   ///< The instruction uses value of the third operand
#define CF_USE4 0x00800   ///< The instruction uses value of the 4 operand
#define CF_USE5 0x01000   ///< The instruction uses value of the 5 operand
#define CF_USE6 0x02000   ///< The instruction uses value of the 6 operand
#define CF_JUMP 0x04000   ///< The instruction passes execution using indirect
                          ///< jump or call (thus needs additional analysis)
#define CF_SHFT 0x08000   ///< Bit-shift instruction (shl,shr...)
#define CF_HLL  0x10000   ///< Instruction may be present in a high level
                          ///< language function.
//@}
};
```

Pointer to `instruc_t` can be found at exported entry `LPH` of processor module at offset +0x94. At offset +0x90 is stored number of instructions. This information can be resolved at runtime.

On a side note I parsed all processor modules and got a list of all `CF_CALL` & `CF_JUMP` instructions. Can be found [here](https://github.com/oct0xor/ida_pro_graph_styling/blob/master/call_instrs_67.txt).

{% include twitter_plug.html %}