---
title: crashing binary ninja with 4 instructions
date: 2024-07-24 19:00:00 +/-TTTT
tags: [reversing]

media_subpath: /assets/posts/binaryninja_crash
---

# Introduction

I first noticed this crash around the beginning of 2022, when EAC deployed [its VM](https://www.unknowncheats.me/forum/anti-cheat-bypass/488156-talk-eac-obfuscation.html). Upon loading the driver binary inside Binary Ninja, it would get analyzed for around 10 minutes and then the process would just crash around 3rd stage of analysis. I didn't care about this behaviour at the time, thinking it may be related to the lack of RAM on my machine, and after disabling **"Control Flow Graph Analysis"** no more issues were observed. [Some things](https://en.wikipedia.org/wiki/Russian_invasion_of_Ukraine) came up in life, making me unable to allocate a meaningful amount of time for further analysis.

Fast forward two years later, I decided to check whether or not the crash was still relevant. Much to my surprise, the problem was still present. I thought someone would report the crash by now, but that wasn't the case.

# Analysis

After letting the instance run for **~5 minutes** we get a crash.
![Desktop View](windbg_crash.png){: .normal }

Yeah, my initial assumption about the crash being memory-related was wrong. Likely there's some intentionally crafted payload inside the EAC driver to exploit a logic problem within Binary Ninja. [Let's load the Binary Ninja DLL that caused the crash into Binary Ninja](https://youtu.be/hV-O6immhcU?t=3344) to investigate it further.

By having an implant inside **Vector 35**, I was able to get ahold of the `.pdb`{: .filepath} for `binaryninjacore.dll`{: .filepath} module to analyze the crash(at the point of writing this blog post I forgot in which order I reverse engineered parts that we're going to look at, so let's just pretend that the situation that was described before is what we have on our hands).

Here's what the faulty code looks like in disassembly view:
![Desktop View](idiv_crash.png){: .normal }

We get into this else case by failing this if case:

```cpp
if (in_cst_type != SignedRangeValue)
```
{: .nolineno}

The function where the crash occurs has the following signature:
```cpp
struct mw_const_info* mw::cst::mul_apply_range(struct mw_const_info* in_cst_info, struct mw_const_info* out_cst_info, uint64_t cst, int64_t operand_size)
```
{: .nolineno }

* **in_cst_info - input argument**, that is going to be restricted based on the constant provided
* **out_cst_info - output argument**, result of input constant info restricted by constant provided
* **cst - input argument**, this is the value by which our range of values gets multiplied
* **operand_size - input argument**, size of operation in bytes

So, to summarize this function, it tries to process a **mul** instruction and, in our case, find possible values that are going to be produced based on a range(it can also be a **set of values**, **constant value**, etc.) and constant used in multiplication.

If we look at cross-references to **mw::cst::mul_apply_range**, there's only one function that references it:
![Desktop View](mlil_dispatcher.png){: .normal }

```cpp
struct mw_const_info* mw::cst::possible_value(struct mw_mlil_ssa_func* ssa_func, struct mw_const_info* out_const_info, struct mw_mlil_info* mlil_inst, int64_t** arg4, int512_t arg5 @ zmm0)
```
{: .nolineno }

This function takes in [SSA form](https://mapping-high-level-constructs-to-llvm-ir.readthedocs.io/en/latest/control-structures/ssa-phi.html) of the MLIL function and the instruction within that function and tries to constant propagate possible values that the **mlil_inst** could attain in a backward slicy kind of way; **mlil_inst** has a lot of useful information to continue our investigation. At offset **0x40**, there's an address of a corresponding assembly instruction. So let's try to load into WinDbg again and catch this address.

We will place a simple breakpoint that'll print **rcx** register(first argument that has info about mlil instruction) at **0x10da348**(right after case statement, we don't bp on **mw::cst::possible_value** because it could be recursively called).
```
bp 10da348 + binaryninjacore "r rcx;g"
```

![Desktop View](idiv_crash_traced.png){: .normal }

Now we can use the latest **mlil_info** pointer and add **0x40** to obtain an address of the crashing instruction

![Desktop View](dq_mlil_result.png){: .normal}

**0x14057cfd6**! Let's try to load the driver binary inside **IDA Pro Jiang Ying edition** (cuz we don't want to crash) and confirm that it is a signed **mul** instruction indeed.

![Desktop View](ida_dis_mul.png){: .normal }

For now, we're somewhere inside a function and don't know the start address, but it's not a concern since we can easily extract it from the associated MLIL function at offset **0x48** inside **mlil_info**.

![Desktop View](reclass_view.png){: .normal }

By manually jumping to **0x14057CF96** and defining the function before Binary Ninja discovers it, we immediately get a crash, confirming that the start address of the crashing function is correct.

{%
    include embed/video.html
    src='crash_by_jump.mp4' 
    title='Binja Crash'
%}

Now let's switch back to IDA and look at this function. 

![Desktop View](ida_full_func_view.png){: .normal}

Combining all of our knowledge we need to:

* Force create a ranged constant by sign extending a 32bit register into a 64bit one
* Sign multiply **0xffffffffffffffff** by our ranged constant
* Jmp to any register used in the calculation to force expression evaluation on **MLIL level**

All of these steps can be done in **4 x86-64** instructions.
Let's create a simple binary using only these and see if that's enough to reproduce the crash. I'm going to use [Zig](https://ziglang.org/) for this one, since it is really nice and easy to set up.

```zig
pub fn main() !void {
    _ = asm volatile (
        \\.intel_syntax
        \\movsxd rax, ecx
        \\mov rbx, 0xFFFFFFFFFFFFFFFF
        \\imul rbx, rax
        \\jmp rbx
    );
}
```

And that's all it takes! Let's see how it looks in IDA with **-Doptimize=ReleaseSmall** specified during build.

![Desktop View](program_generated_ida.png){: .normal}

Nothing unneeded was linked; now, upon the binary load, we get a crash instantly, success.

{%
    include embed/video.html
    src='crash_reproduced.mp4' 
    title='Crash Reproduced'
%}

# Conclusion

Reverse engineering decompilers can be a blast. Give it a shot sometime - you might discover some fun quirks like this one.