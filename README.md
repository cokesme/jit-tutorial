# How to write a JIT compiler
First up, you probably don't want to. JIT, or more accurately "dynamic code
generation," is typically not the most effective way to optimize a project, and
common techniques end up trading away a lot of portability and require fairly
detailed knowledge about processor-level optimization.

That said, though, writing JIT compiler is a lot of fun and a great way to
learn stuff. The first thing to do is to write an interpreter.

## MandelASM
GPUs are fine for machine learning, but serious fractal enthusiasts design
their own processors to generate Mandelbrot sets. And the first step in
processor design, of course, is to write an emulator for it. Our emulator will
interpret the machine code we want to run and emit an image to stdout.

To keep it simple, our processor has four complex-valued registers called `a`,
`b`, `c`, and `d`, and it supports three in-place operations:

- `=ab`: assign register `a` to register `b`
- `+ab`: add register `a` to register `b`
- `*ab`: multiply register `b` by register `a`

For each pixel, the interpreter will zero all of the registers and then set `a`
to the current pixel's coordinates. It then iterates the machine code for up to
256 iterations waiting for register `b` to "overflow" (i.e. for its complex
absolute value to exceed 2). That means that the code for a standard Mandelbrot
set is `*bb+ab`.

### Simple interpreter
The first thing to do is write up a bare-bones interpreter in C. It would be
simpler to use `complex.h` here, but I'm going to write it in terms of
individual numbers because the JIT compiler will end up generating the longhand
logic. In production code we'd include bounds-checks and stuff, but I'm
omitting those here for simplicity.

```c
// simple.c
#include <stdio.h>
#include <stdlib.h>

#define sqr(x) ((x) * (x))

typedef struct { double r; double i; } complex;

void interpret(complex *registers, char const *code) {
  complex *src, *dst;
  double r, i;
  for (; *code; code += 3) {
    dst = &registers[code[2] - 'a'];
    src = &registers[code[1] - 'a'];
    switch (*code) {
      case '=':
        dst->r = src->r;
        dst->i = src->i;
        break;
      case '+':
        dst->r += src->r;
        dst->i += src->i;
        break;
      case '*':
        r = dst->r * src->r - dst->i * src->i;
        i = dst->r * src->i + dst->i * src->r;
        dst->r = r;
        dst->i = i;
        break;
      default:
        fprintf(stderr, "undefined instruction %s (ASCII %x)\n", code, *code);
        exit(1);
    }
  }
}

int main(int argc, char **argv) {
  complex registers[4];
  int i, x, y;
  char line[1600];
  printf("P5\n%d %d\n%d\n", 1600, 900, 255);
  for (y = 0; y < 900; ++y) {
    for (x = 0; x < 1600; ++x) {
      registers[0].r = 2 * 1.6 * (x / 1600.0 - 0.5);
      registers[0].i = 2 * 0.9 * (y /  900.0 - 0.5);
      for (i = 1; i < 4; ++i) registers[i].r = registers[i].i = 0;
      for (i = 0; i < 256 && sqr(registers[1].r) + sqr(registers[1].i) < 4; ++i)
        interpret(registers, argv[1]);
      line[x] = i;
    }
    fwrite(line, 1, sizeof(line), stdout);
  }
  return 0;
}
```

Now we can see the results by using `display` from ImageMagick
(`apt-get install imagemagick`), or by saving to a file:

```sh
$ gcc simple.c -o simple
$ ./simple *bb+ab | display -           # imagemagick version
$ ./simple *bb+ab > output.pgm          # save a grayscale PPM image
$ time ./simple *bb+ab > /dev/null      # quick benchmark
real	0m2.369s
user	0m2.364s
sys	0m0.000s
$
```

![image](http://storage2.static.itmages.com/i/17/0308/h_1488996910_5153802_e6927d8be0.jpeg)

### Performance analysis
JIT can basically eliminate the interpreter overhead, which we can easily model
here by replacing `interpret()` with a hard-coded Mandelbrot calculation. This
will provide an upper bound on realistic JIT performance, since we're unlikely
to optimize as well as `gcc` does.

```c
// hardcoded.c
#include <stdio.h>
#include <stdlib.h>

#define sqr(x) ((x) * (x))

typedef struct { double r; double i; } complex;

void interpret(complex *registers, char const *code) {
  complex *a = &registers[0];
  complex *b = &registers[1];
  double r, i;
  r = b->r * b->r - b->i * b->i;
  i = b->r * b->i + b->i * b->r;
  b->r = r;
  b->i = i;
  b->r += a->r;
  b->i += a->i;
}

int main(int argc, char **argv) {
  complex registers[4];
  int i, x, y;
  char line[1600];
  printf("P5\n%d %d\n%d\n", 1600, 900, 255);
  for (y = 0; y < 900; ++y) {
    for (x = 0; x < 1600; ++x) {
      registers[0].r = 2 * 1.6 * (x / 1600.0 - 0.5);
      registers[0].i = 2 * 0.9 * (y /  900.0 - 0.5);
      for (i = 1; i < 4; ++i) registers[i].r = registers[i].i = 0;
      for (i = 0; i < 256 && sqr(registers[1].r) + sqr(registers[1].i) < 4; ++i)
        interpret(registers, argv[1]);
      line[x] = i;
    }
    fwrite(line, 1, sizeof(line), stdout);
  }
  return 0;
}
```

This version runs about twice as fast as the simple interpreter:

```sh
$ gcc hardcoded.c -o hardcoded
$ time ./hardcoded *bb+ab > /dev/null
real	0m1.329s
user	0m1.328s
sys	0m0.000s
$
```

### JIT design
The basic strategy is to replace `interpret(registers, code)` with a function
`compile(code)` that returns a pointer to a function whose signature is this:
`void compiled(registers*)`. The memory for the function needs to be allocated
using `mmap` so we can set permission for the processor to execute it.

The easiest way to start with something like this is probably to emit the
assembly for `simple.c` to see how it works:

```sh
$ gcc -S simple.c
```

Edited/annotated highlights from the assembly `simple.s`, which is much more
complicated than what we'll end up generating:

```s
interpret:
        // The stack contains local variables referenced to the "base pointer"
        // stored in hardware register %rbp. Here's the layout:
        //
        //   double i  = -8(%rbp)
        //   double r  = -16(%rbp)
        //   src       = -24(%rbp)
        //   dst       = -32(%rbp)
        //   registers = -40(%rbp)      <- comes in as an argument in %rdi
        //   code      = -48(%rbp)      <- comes in as an argument in %rsi

        pushq   %rbp
        movq    %rsp, %rbp              // standard x86-64 function header
        subq    $48, %rsp               // allocate space for six local vars
        movq    %rdi, -40(%rbp)         // registers arg -> local var
        movq    %rsi, -48(%rbp)         // code arg -> local var
        jmp     for_loop_condition      // commence loopage

for_loop_body:
        // (a bunch of stuff to set up *src and *dst)

        cmpl    $43, %eax               // case '+'
        je      add_branch
        cmpl    $61, %eax               // case '='
        je      assign_branch
        cmpl    $42, %eax               // case '*'
        je      mult_branch
        jmp     switch_default          // default

assign_branch:
        // the "bunch of stuff" above calculated *src and *dst, which are
        // stored in -24(%rbp) and -32(%rbp).
        movq    -24(%rbp), %rax         // %rax = src
        movsd   (%rax), %xmm0           // %xmm0 = src.r
        movq    -32(%rbp), %rax         // %rax = dst
        movsd   %xmm0, (%rax)           // dst.r = %xmm0

        movq    -24(%rbp), %rax         // %rax = src
        movsd   8(%rax), %xmm0          // %xmm0 = src.i
        movq    -32(%rbp), %rax         // %rax = dst
        movsd   %xmm0, 8(%rax)          // dst.i = %xmm0

        jmp     for_loop_step

add_branch:
        movq    -32(%rbp), %rax         // %rax = dst
        movsd   (%rax), %xmm1           // %xmm1 = dst.r
        movq    -24(%rbp), %rax         // %rax = src
        movsd   (%rax), %xmm0           // %xmm0 = src.r
        addsd   %xmm1, %xmm0            // %xmm0 += %xmm1
        movq    -32(%rbp), %rax         // %rax = dst
        movsd   %xmm0, (%rax)           // dst.r = %xmm0

        movq    -32(%rbp), %rax         // same thing for src.i and dst.i
        movsd   8(%rax), %xmm1
        movq    -24(%rbp), %rax
        movsd   8(%rax), %xmm0
        addsd   %xmm1, %xmm0
        movq    -32(%rbp), %rax
        movsd   %xmm0, 8(%rax)

        jmp     for_loop_step

mult_branch:
        movq    -32(%rbp), %rax
        movsd   (%rax), %xmm1
        movq    -24(%rbp), %rax
        movsd   (%rax), %xmm0
        mulsd   %xmm1, %xmm0
        movq    -32(%rbp), %rax
        movsd   8(%rax), %xmm2
        movq    -24(%rbp), %rax
        movsd   8(%rax), %xmm1
        mulsd   %xmm2, %xmm1
        subsd   %xmm1, %xmm0
        movsd   %xmm0, -16(%rbp)        // double r = src.r*dst.r - src.i*dst.i

        movq    -32(%rbp), %rax
        movsd   (%rax), %xmm1
        movq    -24(%rbp), %rax
        movsd   8(%rax), %xmm0
        mulsd   %xmm0, %xmm1
        movq    -32(%rbp), %rax
        movsd   8(%rax), %xmm2
        movq    -24(%rbp), %rax
        movsd   (%rax), %xmm0
        mulsd   %xmm2, %xmm0
        addsd   %xmm1, %xmm0
        movsd   %xmm0, -8(%rbp)         // double i = src.r*dst.i + src.i*dst.r

        movq    -32(%rbp), %rax
        movsd   -16(%rbp), %xmm0
        movsd   %xmm0, (%rax)           // dst.r = r
        movq    -32(%rbp), %rax
        movsd   -8(%rbp), %xmm0
        movsd   %xmm0, 8(%rax)          // dst.i = i
        jmp     for_loop_step

for_loop_step:
        addq    $3, -48(%rbp)

for_loop_condition:
        movq    -48(%rbp), %rax         // %rax = code (the pointer)
        movzbl  (%rax), %eax            // %eax = *code (move one byte)
        testb   %al, %al                // is %eax 0?
        jne     for_loop_body           // if no, then continue

        leave                           // otherwise rewind stack
        ret                             // pop and jmp
```

#### Compilation strategy
Most of the above is register-shuffling fluff that we can get rid of. We're
compiling the code up front, which means all of our register addresses are
known quantities and we won't need any unknown indirection at runtime. So all
of the shuffling into and out of `%rax` can be replaced by a much simpler move
directly to or from `N(%rdi)` -- since `%rdi` is the argument that points to
the first register's real component.

If you haven't already, at this point I'd recommend downloading the [Intel
software developer's
manual](https://software.intel.com/en-us/articles/intel-sdm), of which volume 2
describes the semantics and machine code representation of every instruction.

**NOTE:** GCC uses AT&T assembly syntax, whereas the Intel manuals use Intel
assembly syntax. An important difference is that AT&T reverses the arguments:
`mov %rax, %rbx` (AT&T syntax) assigns to `%rbx`, whereas `mov rax, rbx` (Intel
syntax) assigns to `rax`. All of my code examples use AT&T, and none of this
will matter once we're working with machine code.

##### Example: the Mandelbrot function `*bb+ab`
```s
// Step 1: multiply register B by itself
movsd 16(%rdi), %xmm0                   // %xmm0 = b.r
movsd 24(%rdi), %xmm1                   // %xmm1 = b.i
movsd 16(%rdi), %xmm2                   // %xmm2 = b.r
movsd 24(%rdi), %xmm3                   // %xmm3 = b.i
movsd %xmm0, %xmm4                      // %xmm4 = b.r
mulsd %xmm2, %xmm4                      // %xmm4 = b.r*b.r
movsd %xmm1, %xmm5                      // %xmm5 = b.i
mulsd %xmm3, %xmm5                      // %xmm5 = b.i*b.i
subsd %xmm5, %xmm4                      // %xmm4 = b.r*b.r - b.i*b.i
movsd %xmm4, 16(%rdi)                   // b.r = %xmm4

mulsd %xmm0, %xmm3                      // %xmm3 = b.r*b.i
mulsd %xmm1, %xmm2                      // %xmm2 = b.i*b.r
addsd %xmm3, %xmm2                      // %xmm2 = b.r*b.i + b.i*b.r
movsd %xmm2, 24(%rdi)                   // b.i = %xmm2

// Step 2: add register A to register B
movpd (%rdi), %xmm0                     // %xmm0 = (a.r, a.i)
addpd %xmm0, 16(%rdi)                   // (b.r, b.i) += %xmm0
```

The multiplication code isn't optimized for the squaring-a-register use case;
instead, I left it fully general so we can use it as a template when we start
generating machine code.

### JIT mechanics
Rather than compiling a real language, let's just get a basic JIT setup.

```c
// jitproto.c
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>

typedef long(*fn)(long);

fn compile_identity(void) {
  // Allocate some memory and set its permissions correctly. In particular, we
  // need PROT_EXEC (which isn't normally enabled for data memory, e.g. from
  // malloc()), which tells the processor it's ok to execute it as machine
  // code.
  char *memory = mmap(NULL,             // address
                      4096,             // size
                      PROT_READ | PROT_WRITE | PROT_EXEC,
                      MAP_PRIVATE | MAP_ANONYMOUS,
                      -1,               // fd (not used here)
                      0);               // offset (not used here)
  if (!memory) {
    perror("failed to allocate memory");
    exit(1);
  }

  int i = 0;

  // mov %rdi, %rax
  memory[i++] = 0x48;           // REX.W prefix
  memory[i++] = 0x8b;           // MOV opcode, register/register
  memory[i++] = 0xc7;           // MOD/RM byte for %rdi -> %rax

  // ret
  memory[i++] = 0xc3;           // RET opcode

  return (long(*)(long)) memory;
}

int main() {
  fn f = compile_identity();
  int i;
  for (i = 0; i < 10; ++i)
    printf("f(%d) = %ld\n", i, (*f)(i));
  munmap((void*) f, 4096);
  return 0;
}
```

This does what we expect: we've just produced an identity function.

```sh
$ gcc jitproto.c -o jitproto
$ ./jitproto
f(0) = 0
f(1) = 1
f(2) = 2
f(3) = 3
f(4) = 4
f(5) = 5
f(6) = 6
f(7) = 7
f(8) = 8
f(9) = 9
```
