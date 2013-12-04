Recently I’ve been doing some benchmarking and came upon a very surprising behavior from a number of different Intel i7 CPUs (it manifests on Sandy Bridge and Haswell desktop-class CPUs as well as Sandy Bridge-EP Xeon CPUs).

The benchmark is very simple and the result is… bizarre. Perhaps one of the readers of my blog knows what’s going on here. Here’s the C code for the benchmark (full code with a makefile is available in this [Gist](https://gist.github.com/eliben/7770377):

``` c
const unsigned N = 400 * 1000 * 1000;

volatile unsigned long long counter = 0;

// Don't inline the benchmarking code into main
void __attribute__((noinline)) tightloop();
void __attribute__((noinline)) loop_with_extra_call();

void tightloop() {
  unsigned j;
  for (j = 0; j < N; ++j) {
    counter += j;
  }
}

void foo() {
}

void loop_with_extra_call() {
  unsigned j;
  for (j = 0; j < N; ++j) {
    __asm__("call foo");
    counter += j;
  }
}
```

We’re benchmarking tightloop vs. loop_with_extra_call, which does exactly the same thing (increment a volatile counter) but has a dummy call to a do-nothing function in the middle. I don’t think anyone has doubts about how this should behave, right? How much slower do you think the extra call will make this loop? Twice as slow? 10% slower?

Here’s the driving main function:

``` c
int main(int argc, char** argv) {
  if (argc <= 1) {
    return 1;
  }

  if (argv[1][0] == 't') {
    tightloop();
  } else if (argv[1][0] == 'c') {
    loop_with_extra_call();
  }

  return 0;
}
```

Building the code with gcc version 4.8 (same output code is produced by 4.6, as well as when replacing -O2 by -O3):

``` bash
$ gcc -O2 loop-call-weirdness.c -o build/loop-call-weirdness
```

Now I’ll run it on my Intel i7-4771 (Haswell) CPU. First run the version with tightloop:

``` bash
$ perf stat -r 10 -e cycles,instructions  build/loop-call-weirdness t

 Performance counter stats for 'build/loop-call-weirdness t' (10 runs):

     2,659,506,002 cycles       #    0.000 GHz              ( +-  0.19% )
     2,401,144,539 instructions #    0.90  insns per cycle  ( +-  0.00% )

       0.685642994 seconds time elapsed                     ( +-  0.24% )
```

… and with the extra call:

``` bash
$ perf stat -r 10 -e cycles,instructions  build/loop-call-weirdness c

 Performance counter stats for 'build/loop-call-weirdness c' (10 runs):

     2,336,765,798 cycles       #    0.000 GHz              ( +-  0.34% )
     3,201,055,823 instructions #    1.37  insns per cycle  ( +-  0.00% )

       0.602387097 seconds time elapsed                     ( +-  0.39% )
```

Yes, the extra call makes the code faster! You didn’t expect that, did you.

Looking at the disassembly, the compiler is doing fine here, producing quite expected code:

```
0000000000400530 <tightloop>:
  400530:     xor    %eax,%eax
  400532:     nopw   0x0(%rax,%rax,1)
  400538:     mov    0x200b01(%rip),%rdx        # 601040 <counter>
  40053f:     add    %rax,%rdx
  400542:     add    $0x1,%rax
  400546:     cmp    $0x17d78400,%rax
  40054c:     mov    %rdx,0x200aed(%rip)        # 601040 <counter>
  400553:     jne    400538 <tightloop+0x8>
  400555:     repz retq
  400557:     nopw   0x0(%rax,%rax,1)

0000000000400560 <foo>:
  400560:     repz retq

0000000000400570 <loop_with_extra_call>:
  400570:     xor    %eax,%eax
  400572:     nopw   0x0(%rax,%rax,1)
  400578:     callq  400560 <foo>
  40057d:     mov    0x200abc(%rip),%rdx        # 601040 <counter>
  400584:     add    %rax,%rdx
  400587:     add    $0x1,%rax
  40058b:     cmp    $0x17d78400,%rax
  400591:     mov    %rdx,0x200aa8(%rip)        # 601040 <counter>
  400598:     jne    400578 <loop_with_extra_call+0x8>
  40059a:     repz retq
  40059c:     nopl   0x0(%rax)
```

Note that the volatile is key here, since it forces the compiler to produce a load and store from the global on each iteration. Without volatile, the benchmark behaves normally (the extra call makes it significantly slower).

It’s easy to see that tightloop runs 6 instructions per iteration, which computes with the numbers reported by perf (400 million iterations, times 6 instructions, is 2.4 billion instructions). loop_with_extra_call adds two more instructions per iteration (the call to foo and the ret from it), and that also corresponds to the performance numbers.

That’s right, even though the version with the extra call executes 33% more instructions, it manages to do it quicker.

Unfortunately, my fast Haswell CPU (or the Linux kernel coming with Ubuntu 13.10) doesn’t support the whole range of perf stat counters, but running on an older CPU (where the anomaly also exists though the performance difference in smaller), I see that the tightloop benchmark has a lot of frontend and backend stalls (mostly frontend), for a total of 0.92 stalled cycles per instruction. The version with the extra call has just 0.25 stalled cycles per instruction.

So would it be right to assume that the tight loop stalls on loading from counter because the rest of the instructions in the loop depend on its value? So how does the call and ret help here? By providing non-data-dependent instructions that can be run in parallel while the others are stalled? Still, whatever that is, I find this result astonishing.

Let me know if you have any insights.

Related posts:

1. An observation on writing line-processing loop code
2. Book review: “Efficient C++: Performance Programming Techniques” by Bulka & Mayhew
3. SICP section 1.2.1
4. Position Independent Code (PIC) in shared libraries on x64


------

From : http://eli.thegreenplace.net/2013/12/03/intel-i7-loop-performance-anomaly/


