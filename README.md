It’s easy to test Perl modules in various scenarios via GitHub Actions!

See below …

# Linux, standard builds

```
  linux:
    runs-on: ubuntu-latest

    strategy:
      fail-fast: false
      matrix:
        perl-version:
          - '5.34'
          - '5.32'
          - '5.30'
          - '5.28'
          - '5.26'
          - '5.24'
          - '5.22'
          - '5.20'
          - '5.18'
          - '5.16'
          - '5.14'
          - '5.12'
          - '5.10'

    container:
      image: perldocker/perl-tester:${{ matrix.perl-version }}

    steps:
      - uses: actions/checkout@v3
        with:
            submodules: recursive
      - run: perl -V
      - run: cpanm --notest --installdeps .
      - run: perl Makefile.PL
      - run: make
      - run: prove -wlvmb t
```
Assuming your module only depends on CPAN modules, the above will test it against all standard perls from 5.10 to 5.34.

## External Dependencies

At some point early in your `steps`, add `apt install -y ...`, where `...` is your list of apt packages to install.

## Coverage Testing via [Coveralls](https://coveralls.io)

Under `matrix`, include this:
```
        include:
          - perl-version: '5.32'   # or whichever version you want to scope this to
            os: ubuntu-latest
            coverage: true
```
… then, instead of the `prove` line in the original `steps`, do:
```
      - name: Run Tests (no coverage)
        if: ${{ !matrix.coverage }}
        run: make test
      - name: Run tests (with coverage)
        if: ${{ matrix.coverage }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          cpanm -n Devel::Cover::Report::Coveralls
          cover -test -report Coveralls
```
Then, ensure that Coveralls is set up to receive coverage reports from your repository.

# macOS

Even easier than Linux:
```
  macOS:
    runs-on: macOS-latest
```
… then the same `steps` as above.

## External Dependencies

`brew install ...` where on Linux you did `apt`.

# Windows

As easy as macOS:
```
  windows:
    runs-on: windows-latest
```

## External Dependencies

Use [Chocolatey](https://chocolatey.org/); however, I’ve not gotten this to work for, e.g., compiling XS modules.

# Big-Endian

The magic of [QEMU](https://qemu.org) makes this possible …

```
  big-endian:
    runs-on: ubuntu-latest

    steps:
      - name: Get the qemu container
        run: docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
      - name: Run tests on s390x/ubuntu
        run: docker run --rm --interactive s390x/ubuntu bash -c "git clone --recurse-submodules $GITHUB_SERVER_URL/$GITHUB_REPOSITORY; cd $( echo $GITHUB_REPOSITORY | cut -d/ -f2 ); curl -L https://cpanmin.us | perl - --notest --installdeps .; perl Makefile.PL; make; prove -wlvmb t"
```

# Linux, long-double & quadmath

Via [unofficial builds courtesy of simcop2387](https://hub.docker.com/r/simcop2387/perl-tester/tags):
```
  linux-alt-perl:
    runs-on: ubuntu-latest

    strategy:
      fail-fast: false
      matrix:
        perl-version:
          - '5.020.003-main-longdouble-buster'
          - '5.020.003-main-quadmath-buster'

    container:
      image: simcop2387/perl-tester:${{ matrix.perl-version }}

    steps:
```
… with the same `steps` you usually do.

Check [simcop2387’s perl containers](https://hub.docker.com/r/simcop2387/perl/tags) for possible values of `perl-version`.

# Cygwin

This is ugly, but it works:
```
  cygwin:
    runs-on: windows-latest

    steps:
      - name: Set up Cygwin
        uses: egor-tensin/setup-cygwin@v3
        with:
            platform: x64
            packages: perl_base perl-ExtUtils-MakeMaker make gcc-g++ libcrypt-devel libnsl-devel bash
      - uses: actions/checkout@v3
        with:
          submodules: recursive
      - run: perl -V
        shell: C:\tools\cygwin\bin\bash.exe --login --norc -eo pipefail -o igncr '{0}'
      - run: cpan App::cpanminus
        shell: C:\tools\cygwin\bin\bash.exe --login --norc -eo pipefail -o igncr '{0}'
      - name: Install Dependencies
        run: cd $GITHUB_WORKSPACE; cpanm --verbose --notest --installdeps --with-configure --with-develop .
        shell: C:\tools\cygwin\bin\bash.exe --login --norc -eo pipefail -o igncr '{0}'
      - name: perl Makefile.PL
        run: cd $GITHUB_WORKSPACE; perl Makefile.PL
        shell: C:\tools\cygwin\bin\bash.exe --login --norc -eo pipefail -o igncr '{0}'
      - name: make
        run: cd $GITHUB_WORKSPACE; make
        shell: C:\tools\cygwin\bin\bash.exe --login --norc -eo pipefail -o igncr '{0}'
      - name: prove -wlvmb t
        run: cd $GITHUB_WORKSPACE; make test
        shell: C:\tools\cygwin\bin\bash.exe --login --norc -eo pipefail -o igncr '{0}'
```
There’s probably a better approach than having to give that `shell` on each step … ?

Note the `with`.`packages`; see Cygwin’s package repository for names of available packages. (Unlike plain Windows, this _does_ work seamlessly to text XS modules’ integrations with external C libraries.)

# FreeBSD
```
  freebsd:
    runs-on: macos-10.15
    steps:
    - uses: actions/checkout@v2
    - uses: vmactions/freebsd-vm@v0.1.6
      with:
        prepare: pkg install -y p5-ExtUtils-PkgConfig  # demonstration
        run: |
            perl -V
            curl -L https://cpanmin.us | perl - --notest --installdeps .
            perl Makefile.PL
            make
            prove -wlvmb t
```
Note the `prepare` statement to install any external dependencies; it may not be needed for your case.
