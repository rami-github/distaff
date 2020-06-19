# Distaff
Distaff is a zero-knowledge virtual machine written in Rust. For any program executed on Distaff VM, a STARK-based proof of execution is automatically generated. This proof can then be used by anyone to verify that a program was executed correctly without the need for re-executing the program or even knowing what the program was.

### Status
**DO NOT USE IN PRODUCTION.** Distaff is in an early alpha. This means that current functionality is incomplete, and there are known and unknown bugs and security flaws.

## Usage
Distaff crate exposes `execute()` and `verify()` functions which can be used to execute programs and verify their execution. Both are explained below, but you can also take a look at several working examples [here](https://github.com/GuildOfWeavers/distaff/blob/master/src/main.rs).

### Executing a program 
To execute a program on Distaff VM, you can use `execute()` function. The function takes the following parameters:

* `program: &Program` - the program to be executed. A program can be constructed manually by building a program execution graph, or compiled from Distaff assembly (see [here](#Writing-programs)).
* `inputs: &ProgramInputs` - inputs for the program. These include public inputs used to initialize the stack, as well as secret inputs consumed during program execution (see [here](#Program-inputs)).
* `num_outputs: usize` - number of items on the stack to be returned as program output. Currently, at most 8 outputs can be returned.
* `options: &ProofOptions` - config parameters for proof generation. The default options target 120-bit security level.

If the program is executed successfully, the function returns a tuple with 2 elements:

* `outputs: Vec<u128>` - the outputs generated by the program. The number of elements in the vector will be equal to the `num_outputs` parameter.
* `proof: StarkProof` - proof of program execution. `StarkProof` implements `serde`'s `Serialize` and `Deserialize` traits - so, it can be easily serialized and de-serialized.

#### Program inputs
To provide inputs for a program, you must create a [ProgramInputs](https://github.com/GuildOfWeavers/distaff/blob/master/src/programs/inputs.rs) object which can contain the following:

* A list of public inputs which will be used to initialize the stack. Currently, at most 8 public inputs can be provided.
* Two lists of secret inputs. These lists can be thought of as tapes `A` and `B`. You can use `read` operations to read values from these tapes and push them onto the stack.

Besides the `ProgramInputs::new()` function, you can also use `ProgramInputs::from_public()` and `ProgramInputs:none()` convenience functions to construct the inputs object.

#### Writing programs
To execute a program, Distaff VM consumes a [Program](https://github.com/GuildOfWeavers/distaff/blob/master/src/programs/mod.rs) object. This object contains an execution graph for the program, as well as other info needed to execute the program. There are two way of constructing a `Program` object:

1. You can construct a `Program` object by manually building program execution graph from raw Distaff VM [instructions](docs/isa.md).
2. You can compile [Distaff assembly](docs/assembly.md) source code into a `Program` object.

The latter approach is strongly encouraged because building programs from raw Distaff VM instructions is tedious, error-prone, and requires an in-depth understanding of VM internals. All examples throughout these docs use Distaff assembly syntax.

A general description of Distaff VM is also provided 👉 [here](docs). If you are trying to learn how to write programs for Distaff VM, this would be a good place to start.

#### Program execution example
Here is a simple example of executing a program which pushes two numbers onto the stack and computes their sum:
```Rust
use distaff::{ self, ProofOptions, ProgramInputs, assembly };

// we'll be using default options
let options = ProofOptions::default();

// this is our program, we compile it from assembly code
let program = assembly::compile(
    "push.3 push.5 add",
    options.hash_fn()).unwrap();

// let's execute it
let (outputs, proof) = distaff::execute(
        &program,
        &ProgramInputs::none(), // we won't provide any inputs
        1,                      // we'll return a single item from the stack
        &options);

// the output should be 8
assert_eq!(vec![8], outputs);
```

### Verifying program execution
To verify program execution, you can use `verify()` function. The function takes the following parameters:

* `program_hash: &[u8; 32]` - an array of 32 bytes representing a hash of the program to be verified.
* `public_inputs: &[u128]` - a list of public inputs against which the program was executed.
* `outputs: &[u128]` - a list of outputs generated by the program.
* `proof: &StarkProof` - the proof generated during program execution.

The function returns `Result<bool, String>` which will be `Ok<true>` if verification passes, or `Err<message>` if verification fails, with `message` describing the reason for the failure.

Verifying execution proof of a program basically means the following:

> If a program with the provided hash is executed against some secret inputs and the provided public inputs, it will produce the provided outputs.

Notice how the verifier needs to know only the hash of the program - not what the actual program was.

#### Verifying execution example
Here is a simple example of verifying execution of the program from the previous example:
```Rust
use distaff;

let program =   /* value from previous example */;
let proof =     /* value from previous example */;

// let's verify program execution
match distaff::verify(program.hash(), &[], &[8], &proof) {
    Ok(_) => println!("Execution verified!"),
    Err(msg) => println!("Execution verification failed: {}", msg)
}
```

## Fibonacci calculator
Let's write a simple program for Distaff VM (using [Distaff assembly](docs/assembly.md)). Our program will compute the 5-th [Fibonacci number](https://en.wikipedia.org/wiki/Fibonacci_number):

```
push.0      // stack state: 0
push.1      // stack state: 1 0
swap        // stack state: 0 1
dup.2       // stack state: 0 1 0 1
drop        // stack state: 1 0 1
add         // stack state: 1 1
swap        // stack state: 1 1
dup.2       // stack state: 1 1 1 1
drop        // stack state: 1 1 1
add         // stack state: 2 1
swap        // stack state: 1 2
dup.2       // stack state: 1 2 1 2
drop        // stack state: 2 1 2
add         // stack state: 3 2
```
Notice that except for the first 2 operations which initialize the stack, the sequence of `swap dup.2 drop add` operations repeats over and over. In fact, we can repeat these operations an arbitrary number of times to compute an arbitrary Fibonacci number. In Rust, it would like like this (this is actually a simplified version of the example in [fibonacci.rs](https://github.com/GuildOfWeavers/distaff/blob/master/src/examples/fibonacci.rs)):
```Rust
use distaff::{ self, ProofOptions, ProgramInputs, assembly };

// use default proof options
let options = ProofOptions::default();

// set the number of terms to compute
let n = 50;

// build the program
let mut source = String::new();
for _ in 0..(n - 1) {
    source.push_str("swap dup.2 drop add ");
}
let program = assembly::compile(&source, options.hash_fn()).unwrap();

// initialize the stack with values 0 and 1
let inputs = ProgramInputs::from_public(&[1, 0]);

// execute the program
let (outputs, proof) = distaff::execute(
        &program,
        &inputs,
        1,          // top stack item is the output
        &options);

// the output should be the 50th Fibonacci number
assert_eq!(vec![12586269025], outputs);
```
Above, we used public inputs to initialize the stack rather than using `push` operations. This makes the program a bit simpler, and also allows us to run the program from arbitrary starting points without changing program hash.

This program is rather efficient: the stack never gets more than 4 items deep.

## Performance
Here are some very informal benchmarks of running the Fibonacci calculator on Intel Core i5-7300U @ 2.60GHz (single thread) for a given number of operations (remember, it takes 3 operations to compute a single Fibonacci term):

| Operation Count | Execution time | Execution RAM  | Verification time | Proof size |
| --------------- | :------------: | :------------: | :---------------: | :--------: |
| 2<sup>8</sup>   | 180 ms         | negligible     | 2 ms              | 57 KB      |
| 2<sup>10</sup>  | 300 ms         | negligible     | 2 ms              | 78 KB      |
| 2<sup>12</sup>  | 900 ms         | < 100 MB       | 2 ms              | 101 KB     |
| 2<sup>14</sup>  | 3.5 sec        | ~ 350 MB       | 3 ms              | 124 KB     |
| 2<sup>16</sup>  | 14.4 sec       | 1.4 GB         | 3 ms              | 156 KB     |
| 2<sup>18</sup>  | 1 min          | 5.2 GB         | 3 ms              | 188 KB     |
| 2<sup>20</sup>  | 15 min         | > 5.6 GB       | 4 ms              | 225 KB     |

A few notes about the results:
1. Execution time is dominated by the proof generation time. In fact, the time needed to run the program is only about 0.05% of the time needed to generate the proof.
2. For 2<sup>20</sup> case, RAM on my machine maxed out at 5.6 GB, but for efficient execution ~16 GB would be needed. This probably explains why proving time is so poor in this case as compared to other cases. If there was sufficient RAM available, execution time would have likely been around 4 mins.
3. The benchmarks use default proof options which target 120-bit security level. The security level can be increased by either increasing execution time or proof size. In general, there is a trade-off between proof time and proof size (i.e. for a given security level, you can reduce proof size by increasing execution time, up to a point).

## References
Proofs of execution generated by Distaff VM are based on STARKs. A STARK is a novel proof-of-computation scheme that allows you to create an efficiently verifiable proof that a computation was executed correctly. The scheme was developed by Eli-Ben Sasson and team at Technion - Israel Institute of Technology. STARKs do not require an initial trusted setup, and rely on very few cryptographic assumptions.

Here are some resources to learn more about STARKs:

* STARKs whitepaper: [Scalable, transparent, and post-quantum secure computational integrity](https://eprint.iacr.org/2018/046)
* STARKs vs. SNARKs: [A Cambrian Explosion of Crypto Proofs](https://nakamoto.com/cambrian-explosion-of-crypto-proofs/)

Vitalik Buterin's blog series on zk-STARKs:
* [STARKs, part 1: Proofs with Polynomials](https://vitalik.ca/general/2017/11/09/starks_part_1.html)
* [STARKs, part 2: Thank Goodness it's FRI-day](https://vitalik.ca/general/2017/11/22/starks_part_2.html)
* [STARKs, part 3: Into the Weeds](https://vitalik.ca/general/2018/07/21/starks_part_3.html)

StarkWare's STARK Math blog series:
* [STARK Math: The Journey Begins](https://medium.com/starkware/stark-math-the-journey-begins-51bd2b063c71)
* [Arithmetization I](https://medium.com/starkware/arithmetization-i-15c046390862)
* [Arithmetization II](https://medium.com/starkware/arithmetization-ii-403c3b3f4355)
* [Low Degree Testing](https://medium.com/starkware/low-degree-testing-f7614f5172db)
* [A Framework for Efficient STARKs](https://medium.com/starkware/a-framework-for-efficient-starks-19608ba06fbe)

# License
[MIT](/LICENSE) © 2020 Guild of Weavers