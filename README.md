# PC-IX
Interactive UNIX information, files, info and setup guides.

PC/IX 4.x
Interactive UNIX, also known as PC/IX, and 386/ix were UNIX derivitives created for the IBM PC in the early 1980's. PC/IX was the first UNIX sold directly from IBM, but not the first UNIX sold for the IBM PC. (Venix/86 was the first.) The original PC/IX software sold was on 19 floppy disks and sold for 900 dollars. In 1985, 386/ix was introduced, later named Interactive UNIX. The last version released was 4.1.1 in July 1998 and was supported up until 2006.

# Vintage UNIX Toolchain Bootstrap for SCO Interactive UNIX 4.1.1

This document outlines the minimal, verified sequence to bootstrap a working GNU development environment on **Interactive UNIX 4.1.1** (SVR4, i386) using only the native `/bin/cc` and no preexisting GNU tools.

The goal: **compile XFree86 3.3.6** on real hardware (`immortis.smc.local`) using period-correct tool versions that actually build on this platform.

---

## Build Order

Each step depends on the previous. All versions are chosen specifically for **K&R/ANSI C compatibility**, lack of Autoconf/glibc, and historical use with XFree86.

### 1. [`make-3.75`](https://ftp.gnu.org/gnu/make/make-3.75.tar.gz)

#### https://ftp.gnu.org/gnu/make/make-3.75.tar.gz

- **Why**: ISC’s native `make` is too primitive for GNU software.  

- **Special note**: This is the **last GNU Make release with `build.sh`**, designed explicitly for bootstrapping without an existing `make`.  

- **Build method**: Manual compile + link (no `ar`/`ranlib`). **see notes section below**

- Use local `make-3.75` to remake `make-3.75` and install it. 

  ```bash
  # Now have a working make, rebuild make-3.75 the standard way - no build.sh or patches needed. 
  # Unpack a clean source tree and...
  cd /downloads/make-3.75
  # configure detects ISC’s native fnmatch(), skips building fnmatch.c, avoids libglob.a entirely, and produces a clean Makefile.
  ./configure --prefix=/usr/local
  # The new `make` handles the rest—no `ranlib`, no manual linking, no surprises.
  make
  make install	# into /usr/local 
  # Add to the path
  PATH=/usr/local/bin:$PATH
  export PATH
  ```

#### N.B. Purpose of /usr/local in Unix

The `/usr/local` directory is used for software and files that are installed locally by the system administrator. 

| Attribute  | /usr                      | /usr/local                 |
| :--------- | :------------------------ | :------------------------- |
| Managed by | System package manager    | Local administrator        |
| Purpose    | System distribution files | Locally installed software |
| Upgrades   | May be overwritten        | Typically preserved        |

This structure helps maintain a clean separation between system-managed and user-managed files, enhancing system stability and organization.



### 2. [`m4-1.4o`](https://ftp.gnu.org/gnu/m4/m4-1.4.tar.gz)

#### https://ftp.gnu.org/gnu/m4/m4-1.4.tar.gz

- **Why**: Required by `bison` to generate parser code.  
- **Special note**: Later versions depend on Autoconf; 1.4o builds with just `make` and `cc`.

### 3. [`bison-1.25`](https://ftp.gnu.org/gnu/bison/bison-1.25.tar.gz)

#### https://ftp.gnu.org/gnu/bison/bison-1.25.tar.gz

- **Why**: Replaces ISC’s ancient `yacc`; needed to build GCC and XFree86 parsers.  
- **Special note**: 1.25 is the last version before C99 dependencies crept in.

### 4. [`flex-2.5.39`](https://github.com/westes/flex/archive/refs/tags/flex-2.5.39.tar.gz)

#### https://github.com/westes/flex/archive/refs/tags/flex-2.5.39.tar.gz

- **Why**: Modern `lex` replacement; used by XFree86’s `imake` and some GCC components.  
- **Clarification**: `2.5.39` is Debian’s package name for upstream **`2.5.4a`** (1996). This version avoids C99 and builds cleanly with `-Xp`.

### 5. [`gcc-2.8.1`](https://ftp.gnu.org/gnu/gcc/gcc-2.8.1.tar.gz)

#### https://ftp.gnu.org/gnu/gcc/gcc-2.8.1.tar.gz

- **Why**: First GCC version with full SVR4 support and known to work on ISC UNIX 4.1.1.  
- **Special note**: Does **not** use Autoconf; uses a simple `configure` shell script. Matches XFree86 3.3.x requirements.

### 6. [`XFree86-3.3.6`](http://ftp.xfree86.org/pub/XFree86/3.3.6/source/)

#### http://ftp.xfree86.org/pub/XFree86/3.3.6/source/

- **Why**: Final goal — a working graphical desktop on vintage ISC UNIX.  
- **Special note**: This version was **explicitly ported** to SVR4 systems like ISC and SCO. It uses `Imake`, not Autoconf, and expects GCC 2.7–2.8.


---

## Bootstrap Notes for ISC UNIX 4.1.1

Technical notes, patches, and workarounds for building a GNU toolchain on **Interactive UNIX 4.1.1** (i386, SVR4) using only the native `/bin/cc`.

---

## GNU Make 3.75: Patching `build.sh`

### Problem

`build.sh` attempts to create `glob/libglob.a` and link against it.  
ISC UNIX 4.1.1 **lacks `ranlib`**, and its `ar` produces **unindexed archives** that the linker cannot read → **“symbol referencing error”**.

### Solution

Run configure shell script to generate `build.sh` .

Edit `build.sh` to **remove `glob/libglob.a`** and link the `glob` object files directly.

### Steps

Open `build.sh` in `vi`:

```sh
vi build.sh
```

Find the line starting with:

```makefile
objs="commands.o job.o ... glob/libglob.a glob/glob.o ...
```

**Delete `glob/libglob.a`** from the list. Keep `glob/glob.o` and `glob/fnmatch.o`.

Save and exit.

Run:

```bash
chmod +x build.sh
./build.sh
```

### Why it works

- `build.sh` already compiles `glob/glob.o` and `glob/fnmatch.o`
- Linking `.o` files directly avoids `ar`/`ranlib` entirely
- The `-Xp` flag (used by `build.sh`) enables ANSI C mode, so `fnmatch.c` compiles cleanly

## Compiler Flags

### `-Xp`

- **Purpose**: Enables POSIX/ANSI C mode in ISC’s `/bin/cc`
- **Used by**: `build.sh` (automatically), and needed for any ANSI C source (e.g., `fnmatch.c`)
- **Do not use `-Xc`** — it’s not supported on all ISC 4.1.1 installs

### `-D_POSIX_SOURCE`

- **Purpose**: Exposes POSIX functions in system headers
- **Use when**: Compiling core GNU tools manually

## Filesystem Layout

- **Install all custom tools to `/usr/local/bin`**

  - Keeps vendor (`/usr/bin`) and local software separate
  - Matches XFree86 and GCC conventions

- **Add to `.profile`**:

  ```bash
  PATH=/usr/local/bin:$PATH
  export PATH
  ```
