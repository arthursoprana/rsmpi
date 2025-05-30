# MPI bindings for Rust

[![GitHub Actions][actions-shield]][actions]
[![Documentation][doc-shield]][doc]
[![Crates.io][crate-shield]][crate]
[![License: Apache License 2.0 or MIT][license-shield]][license]

The [Message Passing Interface][MPI] (MPI) is a specification for a
message-passing style concurrency library. Implementations of MPI are often used to structure
parallel computation on High Performance Computing systems. The MPI specification describes
bindings for the C programming language (and through it C++) as well as for the Fortran
programming language. This library tries to bridge the gap into a more rustic world.

[actions-shield]: https://github.com/rsmpi/rsmpi/workflows/Test/badge.svg
[actions]: https://github.com/rsmpi/rsmpi/actions
[doc-shield]: https://docs.rs/mpi/badge.svg
[doc]: https://docs.rs/mpi
[crate-shield]: https://img.shields.io/crates/v/mpi.svg?style=flat-square
[crate]: https://crates.io/crates/mpi
[license-shield]: https://img.shields.io/badge/license-Apache_License_2.0_or_MIT-blue.svg?style=flat-square
[license]: https://github.com/rsmpi/rsmpi#license
[MPI]: http://www.mpi-forum.org

## Requirements

An implementation of the C language interface that conforms to MPI-3.1. `rsmpi` is currently tested with these implementations:
- [OpenMPI][OpenMPI] 4.0.3 on Ubuntu-20.04, 4.1.2 on macOS
- [MPICH][MPICH] 3.3.2 on Ubuntu 20.04
- [MS-MPI (Windows)][MS-MPI] 10.1.2 on Windows 2022

Users have also had success with these MPI implementations, but they are not tested in CI:
- [Spectrum MPI][Spectrum-MPI] 10.3.0.1
- [Cray MPI][Cray-MPI] 8.1.16 with `PrgEnv-amd/8.3.3`

For a reasonable chance of success with `rsmpi` with any MPI implementation, you must have one of:

- export `MPI_PKG_CONFIG` to be the name or path for pkg-config for your implementation
  - `rsmpi` automatically uses `CRAY_MPICH_DIR` on Cray environments so the above need not be set
  - `mpich` and `ompi` are tried by default as a last resort
  - Tip: test with a command like `pkg-config --cflags --libs mpich`
- The implementation provides a C compiler wrapper `mpicc`
  - export `MPICC=/path/to/mpicc` to specify fully
  - otherwise tries `mpicc` in `$PATH`
  - `mpicc -show` should print the full command line that is used to invoke the wrapped C compiler in gcc-compatible syntax (e.g., `-lmpi`, `-I/usr/local/include`, ...)
- On Windows,
  - for MS-MPI the variables `MSMPI_INC` and either `MSMPI_LIB32` or `MSMPI_LIB64` should be set
  - for Intel MPI the variable `I_MPI_ROOT` should be set

Since the MPI standard leaves some details of the C API unspecified (whether to implement certain constants and even functions using preprocessor macros or native C constructs, the details of most types, etc.) `rsmpi` takes a two step approach to generating functional low-level bindings.

First, it uses a thin static library written in C (see [rsmpi.h][rsmpih] and [rsmpi.c][rsmpic]) that tries to capture the underspecified identifiers and re-exports them with a fixed C API. This library is built from [build.rs][buildrs] using the `gcc` crate.

Second, to generate FFI definitions tailored to each MPI implementation, `rsmpi` uses `rust-bindgen` which needs `libclang`. See the [bindgen project page][bindgen] for more information.

Furthermore, `rsmpi` uses the `libffi` crate which installs the native `libffi` which depends on certain build tools. See the [libffi project page][libffi] for more information.

[OpenMPI]: https://www.open-mpi.org
[MPICH]: https://www.mpich.org
[MS-MPI]: https://docs.microsoft.com/en-us/message-passing-interface/microsoft-mpi
[Spectrum-MPI]: https://www.ibm.com/products/spectrum-mpi
[Cray-MPI]: https://docs.nersc.gov/development/programming-models/mpi/cray-mpich/
[rsmpih]: https://github.com/rsmpi/rsmpi/blob/main/mpi-sys/src/rsmpi.h
[rsmpic]: https://github.com/rsmpi/rsmpi/blob/main/mpi-sys/src/rsmpi.c
[buildrs]: https://github.com/rsmpi/rsmpi/blob/main/mpi-sys/build.rs
[bindgen]: https://github.com/servo/rust-bindgen
[libffi]: https://github.com/tov/libffi-rs

## Usage

Add the `mpi` crate as a dependency in your `Cargo.toml`:

```toml
# "features" is optional
[dependencies]
mpi = { version = "0.8.0", features = ["user-operations", "derive"] }
```

Then use it in your program like this:

```rust

use mpi::request::WaitGuard;
use mpi::traits::*;

fn main() {
    let universe = mpi::initialize().unwrap();
    let world = universe.world();
    let size = world.size();
    let rank = world.rank();

    let next_rank = (rank + 1) % size;
    let previous_rank = (rank - 1 + size) % size;

    let msg = vec![rank, 2 * rank, 4 * rank];
    mpi::request::scope(|scope| {
        let _sreq = WaitGuard::from(
            world
                .process_at_rank(next_rank)
                .immediate_send(scope, &msg[..]),
        );

        let (msg, status) = world.any_process().receive_vec();

        println!(
            "Process {} got message {:?}.\nStatus is: {:?}",
            rank, msg, status
        );
        let x = status.source_rank();
        assert_eq!(x, previous_rank);
        assert_eq!(vec![x, 2 * x, 4 * x], msg);

        let root_rank = 0;
        let root_process = world.process_at_rank(root_rank);

        let mut a;
        if world.rank() == root_rank {
            a = vec![2, 4, 8, 16];
            println!("Root broadcasting value: {:?}.", &a[..]);
        } else {
            a = vec![0; 4];
        }
        root_process.broadcast_into(&mut a[..]);
        println!("Rank {} received value: {:?}.", world.rank(), &a[..]);
        assert_eq!(&a[..], &[2, 4, 8, 16]);
    });
}
```

## Features

The bindings follow the MPI 3.1 specification.

Currently supported:

- **Groups, Contexts, Communicators**:
  - Group and (Intra-)Communicator management from section 6 is mostly complete.
  - no Inter-Communicators
  - no process topologies
- **Point to point communication**:
  - standard, buffered, synchronous and ready mode send in blocking and non-blocking variants
  - receive in blocking and non-blocking variants
  - send-receive
  - probe
  - matched probe/receive
- **Collective communication**:
  - barrier
  - broadcast
  - (all) gather
  - scatter
  - all to all
  - varying counts operations
  - reductions/scans
  - blocking and non-blocking variants
- **Datatypes**: Bridging between Rust types and MPI basic types as well as custom MPI datatypes which can act as views into buffers.

Not supported (yet):

- One-sided communication (RMA)
- MPI parallel I/O
- A million small things


### Optional Cargo Features

These optional features can be enabled in your cargo manifest. See the [Usage](#usage) section
above.

`user-operations` enables capturing lambdas and safe creation in `UserOperation`. This feature
requires the `libffi` system library, which is not available on all systems out-of-the-box.

```rust
let mut h = 0;
comm.all_reduce_into(
    &(rank + 1),
    &mut h,
    &UserOperation::commutative(|x, y| {
        let x: &[Rank] = x.downcast().unwrap();
        let y: &mut [Rank] = y.downcast().unwrap();
        for (&x_i, y_i) in x.iter().zip(y) {
            *y_i += x_i;
        }
    }),
);
```

`derive` enables the `Equivalence` derive macro, which makes it easy to send structs
over-the-wire without worrying about safety around padding, and allowing arbitrary datatype
matching between structs with the same field order but different layout.

```rust
#[derive(Equivalence)]
struct MyProgramOpts {
    name: [u8; 100],
    num_cycles: u32,
    material_properties: [f64; 20],
}
```

## Documentation

Every public item of `rsmpi` should at least have a short piece of documentation associated with it. Documentation can be generated via:

```
cargo doc
```

Documentation for the latest version of the crate released to crates.io is [hosted on Github pages][doc].

## Examples

See files in [examples/][examples]. These examples also act as [integration tests][actions].

[examples]: https://github.com/rsmpi/rsmpi/tree/master/examples

## Python integration

It is possible to use `rsmpi` with a communicator provided by [mpi4py](https://mpi4py.readthedocs.io/en/stable/). An example project demonstrating this is [mpi4py_with_rsmpi](https://github.com/betckegroup/mpi4py_with_rsmpi).

## License

Licensed under either of

 * Apache License, Version 2.0, ([LICENSE-APACHE](LICENSE-APACHE) or http://www.apache.org/licenses/LICENSE-2.0)
 * MIT license ([LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT)

at your option.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally
submitted for inclusion in the work by you, as defined in the Apache-2.0
license, shall be dual licensed as above, without any additional terms or
conditions.
