# Programming languages for Chrome OS

## Userspace

Chrome OS's userspace is predominantly written in C++, with some third-party
packages written in C, some functionality written in shell scripts, and some
newer codebases written in Rust.

### Using C in new userspace projects

Usage of C in new first-party userspace projects is strongly discouraged. Unless
there is a critical, external to Chrome OS reason to write a new userspace
project in C, usage of C will be disallowed.

`platform` and `platform2` OWNERS are encouraged to reject new C code in
first-party userspace Chrome OS components.
