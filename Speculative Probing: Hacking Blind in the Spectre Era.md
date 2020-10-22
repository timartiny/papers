# Overview
Code "protected" by ASLR can be attacked using a method known as Blind Probing.
This involves randomly probing with vulnerabilities which often results in
crashes, until the correct offset is determined.

This method of Blind Probing is ineffective against highly sensitive programs
such as the Linux Kernel, where multiple crashes are easily detected.

In the era of Spectre however, blind probes can continue without actually
causing crashes.

Most Spectre defenses protect against architechtural components, such as the
Branch Target Buffer, this paper will exploit software vulnerabilities.

# Background
## Code Reuse attacks
CRAs exploit existing code to write out of bounds, overwriting the code pointer
to point to a series of gadgets to execute malicious code

Kernel Address Space Layout Randomization (KASLR) prevents this by randomizing
the location of the Kernel base code. Thus above attacks require a leak of the
the actual address of code to run the exploit, or cause the program to crash.

## "Blind" Code Reuse attacks
These attacks do not require an info leak of the address space, but instead
require probing. E.g., One attack attempts to implement a return-to-libc style
attack by modifyin the return address to be each address in programs that will
spawn new processes in the event of a crash.

## Cache attacks
Flush+Reload style attacks. By measuring the time needed to load information an
attacker can determine which cache lines are currently in memory

## Speculative Execution Attacks
### Spectre V1
Train branch predictor to go down certain paths, flush cache, provide
"incorrect" input to leak information from out-of-bounds variables
### Spectre V2
Poison the Branch Target Buffer (BTB) to hijack indirect calls to a controlled
location and execute unexpected code.
### Defenses
Spectre defenses prevent user-level code from influencing microarchitechture.
This work utilizes these methods, but doesn't not actually execute Spectre
attacks, bypassing defenses.

# Threat Model
Attacker is aware of a privilege escalation vuln on a system. They are able to
execute code on that system. 

The target of the attack is the Linux Kernel with allll the defenses: KASLR,
Meltdown Defenses, Foreshadow defenses, retpolines, and restricts unauthorized
out-of-bound memory accesses.

# Speculative Probing
Example code:
```
if (expression) {
    /*  */
    f_ptr()
    /* */
}
```
Spectre V2 attacks require poisoning the BTB into guessing where `f_ptr()` is
called. In Speculative Probing, this is directly poisoned using a software vuln
similar to code reuse. On its surface this might cause the crashes we want to
avoid. But as the call is inside an if statement it can be executed
speculatively, avoiding crashes.

**Small Note:** the `f_ptr()` call will also be speculated, but that will
resolve first as the incorrect call (due to software manipulation) then the
"real" (but corrupted) function will be called. Retpoline like defenses cripple
the secondary speculation, but that isn't the goal.

# Speculative Probing Primatives
Like code reuse attacks in a KASLR system there are two stages:
1. Stage one: generic probes with no a priori knowledge to find gadget locations
2. Stage two: specific probes using found gadgets to find memory offsets.

Both stages use the set up above of poisoning the function pointer in a
speculative scenario.

## Code region probing
1. Use software vuln to overwrite function pointer
2. Train CPU predictor to speculatively dereference poisoned code pointer
3. Prime part of cache with eviction set
4. Issue syscall to cause kernel to speculatively execute function pointer
5. Use `P+P` to determine if address is in cache.

Note that because the function pointer is executed speculatively, if the address
is invalid or not executable, there is no crash.

If it *is* executable then the address will be in the cache line.

## Gadget Probing
CRAs consist of gadget probing as well. Searching for popular use gadgets to
execute their attacks.

The only difference here is the layer of abstraction of speculation. So in
particular the first gadget they search for is a Spectre gadget (for Spectre
V1).

Then future gadgets are searched for, now using cachelines as the notification.

## Data Region Probing
In theory only code region probing is *necessary*. However, this can be slow, as
finding everything through speculation is cumbersome. 

In addition, attacks proposed in this paper often require the location of the
heap. To find it they use two chained de-references to determine whether a page
has successfully been read by a corrupted pointer, if so then it is a data page.

## Object Probing
To find location of specific objects you need 3 chained de-references, to ensure
object is loaded.

## Spectre Probing
If specific memory values are necessary for an attack, use a Spectre gadget to
actually read the data.

## Optimizations
Cache stuff is noisy, so lots of tests to ensure no false positives.

# Exploitation
