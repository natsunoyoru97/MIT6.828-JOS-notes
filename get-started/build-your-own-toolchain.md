# Get Ready: Build Your Own Toolchain

Here I use my **MacOS** as the environment to do ALL the labs, but I build my toolchain by compiling the source code of the dependencies.

The dependencies listed in MIT 6.828 website are:
```
    ftp://ftp.gmplib.org/pub/gmp-5.0.2/gmp-5.0.2.tar.bz2
    https://www.mpfr.org/mpfr-3.1.2/mpfr-3.1.2.tar.bz2
    http://www.multiprecision.org/downloads/mpc-0.9.tar.gz
    http://ftpmirror.gnu.org/binutils/binutils-2.21.1.tar.bz2
    http://ftpmirror.gnu.org/gcc/gcc-4.6.4/gcc-core-4.6.4.tar.bz2
    http://ftpmirror.gnu.org/gdb/gdb-7.3.1.tar.bz2
```

Things I did to build my toolchain:

Note: Usually these steps **require privilege** in macOS.

- ``sudo wget <website>``. You may noticed that the download speed is extremely slow if fetching these files directly, so downloading from the mirror sites is VERY helpful. Here, the ``wget`` command would help you to handle this issue.
- Unzip all the ``.tar.bz2`` files.
- Then