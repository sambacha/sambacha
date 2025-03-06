Related to #11105 and #9353.

  

## Abstract / Motivation

This proposal changes the way paths in import statements, on the CLI and in Standard JSON are handled by the compiler and translated into internal source unit IDs.

  

The goal is to make imports more intuitive by directly exposing user to the way compiler identifies files internally. The current system hides the abstraction that happens between the actual filesystem and compiler's virtual filesystem and makes users expect import paths to behave like filesystem paths even though they work differently.

  

The change is meant to preserve a forward-compatible subset of the old syntax to make it possible to have the same files compile with both old and new compiler by only changing the remappings and compiler options.

  

The syntax for named roots intentionally follows the established convention of using `@` placeholders in imports.

  

To avoid changing the meaning of existing syntax in a confusing way relative imports of the form `import "project/contract.sol";` are disallowed rather than made equivalent to `import "./project/contract.sol";` even though having both work the same way would be quite intiuitive.

  

### Examples

* Many cases of users being confused by the current system (including me :)) can be found in the bug tracker: [[CLI] --base-path unexpectedly effects both import and source URLs #9346](https://github.com/ethereum/solidity/issues/9346), [Standard JSON compilation cannot find or read files. #2266](https://github.com/ethereum/solidity/issues/2266), [Non-deterministic output according to filesystem path #9790](https://github.com/ethereum/solidity/issues/9790), [Different bytecodes for the same contracts #6487](https://github.com/ethereum/solidity/issues/6487), [fail to import the sol in parent path #4914](https://github.com/ethereum/solidity/issues/4914), [Compiler ignores files located under the base path in Standard JSON #11038](https://github.com/ethereum/solidity/issues/11038), [gitter.im/ethereum/solidity?at=60521c4883533831b4e736eb](https://gitter.im/ethereum/solidity?at=60521c4883533831b4e736eb).

* The path in `import "/project/contract.sol";` looks like an absolute path and indeed will load `/project/contract.sol` in the simplest case. It is however relative to `--base-path`, just like `import "project/contract.sol";`.

* `import "./contract.sol";` is relative to the current source file while `import "contract.sol";` is relative to the base path (or current working directory if base path is not set). This distinction is not obvious since in the shell both paths are equivalent and lead to the same file.

* Paths are not normalized which means that `project/contract.sol` and `project//contract.sol` are seen as two completely different files (and actually can be different files when the source is provided via Standard JSON) but cause the same file to be loaded from the filesystem. The resulting errors are confusing if the user is not aware of how the compiler decides whether files are distinct or not.

* Relative paths starting with `../` or `./` are normalized, but only partially. If `../project//contract.sol` is imported from `/work//contracts/../token.sol`, the path resolves into `/work//contracts/contract.sol`. Note `..` being treated as an actual directory and `//` in one part not being replaced with `/`.

* The way a file is referenced on the command line affects whether it matches an import. For example `solc contract.sol` will be seen by the compiler as the same as `import "contract.sol";` but if we go to the parent directory and compile it as `solc dir/contract.sol` it will be seen as a different file and compiled twice.

* What's worse, it affects the metadata - both cases produce different bytecode because different paths end up in the metadata - even though the files are identical and still reside in the same directories.

* When used as a target of a remapping, `../` no longer works as relative to the source file. It's now relative to the current working directory because the remapping happens after the relative paths are resolved.

  

There are more examples listed in #11036. While they were originally reported as bugs, ultimately most of them are actually just unintuitive side-effects of the current design that mostly show up in corner cases.

  

## Specification

### Overview

Paths given in import statements, on the command line and in Standard JSON are used for two purposes:

  

* to find and load the source code into compiler's virtual filesystem,

* to generate a unique source unit ID that determines whether two paths actually refer to the same source unit.

  

A source unit ID consists of a named or unnamed root and a source path. E.g. `@openzeppelin/utils/math/Math.sol` or `@/contracts/token.sol`.

  

There are several ways files can get into the virtual filesystem. The most important one is an import statement. Paths in import statements can be specified in three ways:

  

1. **Direct import**: specifies the root and source path explicitly: `@openzeppelin/utils/math/Math.sol`.

2. **Relative import**: specifies only a part of the source path, relative to the source ID of the importing file: `./math/Math.sol` is equivalent to `@openzeppelin/utils/math/Math.sol` when imported from `@openzeppelin/utils/Arrays.sol`.

3. **Remote import**: uses an URL in place of the root: `https://github.com/OpenZeppelin/openzeppelin-contracts/contracts/utils/math/Math.sol`.

  

For a remote import to be valid, user needs to assign a named root to a matching prefix (on the CLI or in Standard JSON). For example `https://github.com/OpenZeppelin/openzeppelin-contracts/contracts=@openzeppelin`. After the remapping, the path is processed as if it were a direct import. It's also possible to remap one named root to another (e.g. `@openzeppelin=@oz`). Every remapping to a named root becames a part of contract metadata because the mapping happens between the import path and the source unit ID and changing it may affect the result of the compilation even if the source stays they same.

  

In typical usage named roots represent libraries or independent submodules of your project. The main project itself is represented just by `@`. `@` is special in that it can represent different directories, depending on where it is used. When used in a file located under some named root it represents that root. This way, when writing a library you can safely refer to its root just as `@` (i.e. `import "@/utils/math/Math.sol";`). A standalone project using your library can refer to library files via a named root (`import "@openzeppelin/utils/math/Math.sol";`) and use `@` for its own files without a conflict. The substitution happens when import path is translated into a source unit ID - in the virtual filesystem the source IDs of library files always contain the full named root.

  

To be able to locate the file and load it the compiler passes its source unit ID to the source loader. The loader determines how roots translate to specific locations. In case of the command-line compiler, locations must be existing directories. All named roots must be explicitly mapped for a contract to be compilable. The unnamed root is by default mapped to compiler's working directory but can also be explicitly remapped.

  

```shell

solc ../contracts/contract.sol @openzeppelin=node_modules/openzeppelin/contracts/ @=../contracts/

```

  

Files on the command line can be specified in two ways:

  

1. As filesystem paths: these are platform-specific and can be relative to the current working directory. E.g. `../contracts/contract.sol` or `C:\project\contracts\contract.sol`.

2. As source unit IDs: instead of specifying the path directly you specify a source unit ID like `@openzeppelin/utils/math/Math.sol` and have the compiler resolve it by passing it to the source loader.

  

When supplying files using Standard JSON, you always specify source IDs yourself. These IDs must of course contain a named or unnamed root. E.g. `math/Math.sol` is not a valid source unit ID.

  

Instead of supplying the source as a part of the JSON file (via the `content` key) you can specify its location (via the `urls` key). It can be a path or an URL and whether it can be successfully resolved depends on the compiler interface you use. The command-line interface can only resolve filesystem paths and source unit IDs. The JavaScript interface can also handle URLs or even arbitrary identifiers - it's all up to the user-defined callback.

  

Many details in the above description were intentionally omitted to keep it concise. Additional sections below clarify finer points of the new system.

  

### Normalization

Source unit IDs used internally are always in a normalized form:

  

* Root name can contain only letters, numbers and maybe some safe special characters like `_` and `-`.

* It starts with `@` and ends with `/`.

* `@` and `:` are not allowed inside root name.

* Source path:

* is case-sensitive and in UNIX format regardless of the underlying platform,

* cannot start with `.` or contain any `./` or `../` segments,

* does not contain sequences of multiple slashes, trailing slashes, leading/trailing whitespace.

  

Source IDs specified in Standard JSON must be already normalized. In other contexts compiler may automatically apply some normalization rules:

  

* Relative imports start with `./`, which is stripped by the compiler.

* The part of the import that is the prefix is never normalized. Source path (the part left after stripping the prefix) must be normalized like in any other path.

* Filesystem paths given on the command line undergo the usual normalization expected from shell commands:

* multiple slashes are squashed into one.

* relative paths are relative to the current working directory and converted into absolute ones.

* `./` segments are stripped,

* `../` segments are collapsed.

* The path is also converted from a platform-specific format into the UNIX format before it is used to construct source unit ID.

* Source unit IDs given on the command line must be already normalized.

* Filesystem paths and source unit IDs given in `urls` in standard JSON behave just like the ones specified on the command line (though they are never used to form source unit IDs so the only thing that matters is which file they resolve to).

  

### `@` escaping

An escaping mechanism is needed to discern named roots from paths starting with `@` character in contexts where both are allowed. For that purpose a leading `@@` is always interpreted as a single `@` and causes the following value not to be seen as a root.

  

### Relative imports

* Relative imports work by taking the source unit ID of the importing module as a base. Everything after the last slash is removed from the importing ID and `./` is stripped from the import path. Then they are combined.

* Apart from the leading `./`, the path must be normalized according to the same rules as source path in source unit IDs.

* Relative imports **must** start with `./`. `../` is not allowed.

  

### Remote imports

* A remote import must start with `protocol://`, where `protocol` can be anything except for `file`.

* The prefix of any remote import must be mapped to a named root. The length of the prefix is up to the user but it must at least include the `protocol://` part.

* The part left after stripping the prefix becomes the source path in the VFS.

* If multiple remappings match the same prefix, the longer one wins.

  

### Import remapping vs root remapping

There are two kinds of remappings:

  

* **Import remapping**: always remaps something to a named root.

* This is allowed from URL prefixes or other named roots.

* These remappings are included in contract metadata.

* Remapping to `@` is not allowed.

* Remappings are not recursive. `@a=@b @b=@c @c=@d` will remap `@a` to `@b`, not `@d`.

* Remapping a root to itself (`@abc=@abc`) is allowed and can be used as a way to prevent a shorter remapping from matching (e.g. adding `@contract=@token` to `@con=@pro` will prevent `@contract` from being remapped to `@protract`.

* **Root remapping**: always remaps a root to something that is not a root

* The target must be something that combined with the source path is recognized by source loader as a valid source location.

* On the CLI it must be a path. If the path is relative, it's interpreted as relative to current working directory, converted into an absolute one and normalized.

* In the JavaScript interface it could be an URL or something completely arbitrary.

* These remappings are system-dependent and not stored in the metadata. Checksums stored in metadata are used instead to ensure that compiler input is the same regardless of the system.

  

### Remapping context

To solve conflicts caused by different libraries referring to their dependencies in the same way, it's possible to qualify import remappings with a _context_.

  

* the context must be a named or unnamed root (filesystem paths are not allowed),

* context can only be used for **import remappings** from named roots,

* using context when remapping URL prefixes is not allowed (URLs by their nature are expected to be absolute).

  

If an import remapping has a context, the substitution is only performed on imports found inside the files located under the named root used as context.

  

Examples:

  

```

@libA:@oz=@openzeppelin

@libB:@oz=@oz

@oz=@australia

```

  

### Supplying files on the CLI

All filesystem paths specified on the CLI that lead to files to be compiled must be located within one of the roots. Since the unnamed root is by default mapped to current working directory, files from that directory can still be conveniently compiled without specifying any remappings in simple cases. The source unit ID for the file is constructed by normalizing the path and finding the root that is mapped to the longest matching prefix.

  

The CLI supports source unit IDs but **not** direct imports. I.e. `@` never refers to a named root and import remappings are not taken into account.

  

### Supplying files via Standard JSON

Source unit IDs specified in Standard JSON must be already normalized and contain a root. As a special case it can also be equal to `<stdin>`. Any other form of an ID is disallowed.

  

URLs specified in `sources.urls` are treated as raw URLs, not remote imports. I.e. remappings are not applied to them. Source unit IDs specified there are also not direct imports.

  

### Standard input

A special source ID `<stdin>` is reserved for the content of compiler's standard input.

  

* It is present in the VFS only when the `-` command-line flag is specified.

* It cannot be used in remappings.

* The parent source ID used when resolving relative imports is `@`.

* Its content can be provided explicitly in Standard JSON (to ensure feature parity between Standard JSON and CLI).

  

### Base path

The base path has no function in the new system but could be retained for backwards-compatibility. `--base-path <dir>` would have the same effect as remapping `@=<dir>`.

  

### Allowed paths

* All directories mapped to named roots are automatically added to the list of allowed directories.

* `--allowed-paths` option is also still available. It is the only way to compile the project when the directory a root is mapped to contains symlinks that lead outside of it.

  

### Possible extensions

#### Library path

Specifying mapping for all named roots may be tedious. To make it more convenient we could introduce the concept of library path. It would be defined by a variable called `SOLIDITYPATH` and work in a way similar to `PATH` in Bash or `PYTHONPATH` in Python. All subdirectories of directories listed in `SOLIDITYPATH` would automatically become valid named roots.

  

## Backwards-compatibility

The proposal only restricts current syntax and does not introduce any new elements.

  

* Imports starting with `../` and `/` are no longer allowed.

* Non-normalized paths are no longer allowed in many contexts

* Arbitrary mapping targets and prefixes are no longer allowed.

* Mapping context is must now start with `@`.

a

As such it's not backwards-compatible but any file compilable after the change should also be compilable with older compilers given the right remappings.

  

Filesystem paths on the CLI will now produce different source unit IDs because paths are absolute and converted to relative to a root (though, arguably, this is how it was originally supposed to work with `--base-path` and could be considered a bug instead: [#11038 (comment)](https://github.com/ethereum/solidity/issues/11038#issuecomment-793659406)).

  

To use URLs as imports an intermediate mapping to and from a named root is required. This makes it impossible to support arbitrary URLs (though arbitrary URLs within a single protocol are still possible). Reader callback passed to the JavaScript interface now receives files after root remapping. Before it was getting source unit IDs directly. This will affect Remix IDE.


---

https://github.com/ethereum/solidity/issues/11036

The way the compiler resolves imports into actual filesystem paths is intuitive in simple cases but gets convoluted quickly when you take into account all the factors that can affect it: remappings, base path, various flavors of relative paths, special file names, etc. I checked how it works in corner cases and in a lot of them the beahavior is either buggy or at least does not work the way I would expect it to.

## Issues
This section lists problems I have identified. See the reference at the end of the post for detailed examples.

1. Base path is sometimes prepended to absolute paths.
   
   * Base path is prepended to source directory path in imports starting with `.` or `..`.
   * Base path is prepended to remapped absolute imports.
2. Imports starting with `.` or `..` are not always relative to the source directory
   
   * Absolute imports can be remapped into imports starting with `.` or `..` but they then become relative to the current working directory.
   * Base path replaces source directory in absolute imports remapped to relative imports starting with `.` or `..`.
     
     * It should behave the same way as relative imports starting with `..` or `..`.
   * Paths that become relative after stripping of `file://` are always relative to the current working directory.
     
     * This does not happen when base paths is used.
3. Using a relative path to the source file on the command line sometimes makes relative imports resolve to different paths than when using an absolute path.
   
   * Remapping of `.` and `..` works differently when the path to the source file given to the compiler is relative.
   * Remappings ignore any number of leading `.` and `..` segments when the path to the source file given to the compiler is relative.
4. Relative paths with `..` segments that go beyond filesystem root are silently replaced with valid ones.
   
   * Remappings ignore any number of leading `.` and `..` segments in imports going beyond the root directory.
   * Remappings ignore any number of leading `.` and `..` segments when the path to the source file given to the compiler is relative.
   * Relative imports going beyond root directory ignore `..` and are relative to the current working directory (or to the base directory when base path is used).
5. It's possible to remap some of the paths added implicitly to relative imports.
   
   * Parts of the path that `.` and `..` get resolved into can get remapped.
   * Source directory path can be remapped.
6. Not all prefixes can be remapped.
   
   * Leading `.` and `..` in imports cannot be remapped.
   * `file://` prefix as a whole cannot be remapped but its fragments can.
7. Special behavior of `<stdin>`.
   
   * A file called `<stdin>` from current working directory can only be imported if use of the standard input is not requested.
     
     * If a file called `<stdin>` exists and standard input was requested, importing it should be an error.
   * Behavior of `<stdin>` in general might be unintended though there's probably no harm in allowing it to be remapped.
8. Trailing slashes in imports are replaced with `/.` (unless they start with `.` or `..`).
   
   * They should either be stripped or replaced with a single slash.
9. Multiple remappings of the same prefix are ignored and the last one takes effect.

## Current path resolution behavior
This section is meant mostly as a reference. There's no need to read it all to understand the issue. Skimming the headers should give you a good picture of what is happening.

Use it to see examples of the problems I listed above or just the current behavior in specific cases.

### Common setup for all examples below
* File containing the imports: `/project/contract.sol`.
* Working directory: `/home/user/work/`.

### Simple imports
#### Absolute
```solidity
import "/tmp/code/token.sol";         // -/tmp/code/token.sol
import "/tmp/code/../code/token.sol"; // -/tmp/code/token.sol
```

#### Relative imports are relative to current working directory
```solidity
import "token/token.sol";     // -/home/user/work/token/token.sol
import ".../token/token.sol"; // -/home/user/work/.../token/token.sol
```

#### Relative imports starting with `.` or `..` are relative to the source directory
```solidity
import "./token/token.sol";          // -/project/token/token.sol
import "../project/token/token.sol"; // -/project/token/token.sol
```

#### Relative imports going beyond root directory ignore `..` and are relative to the current working directory
```solidity
import "token/token.sol";                  // -/home/user/work/token/token.sol
import "../token/token.sol";               // -/home/user/token/token.sol
import "../../token/token.sol";            // -/home/token/token.sol
import "../../../token/token.sol";         // -/token/token.sol
import "../../../../token/token.sol";      // -/home/user/work/token/token.sol
import "../../../../../token/token.sol";   // -/home/user/work/token/token.sol
import "../../.././../../token/token.sol"; // -/home/user/work/token/token.sol
```

### Path Remapping
#### Absolute imports can be remapped
```solidity
// Remappings: /tmp/code=/usr/lib
import "/tmp/code/token.sol"; // -/usr/lib/token.sol
```

#### Relative imports can be remapped
```solidity
// Remappings: token=contract
import "token/token.sol"; // -/home/user/work/contract/token.sol
```

#### Relative imports can be remapped into absolute imports
```solidity
// Remappings: token=/usr/lib
import "token/token.sol"; // -/usr/lib/token.sol
```

#### Absolute imports can be remapped into imports relative to the current working directory
```solidity
// Remappings: /usr/lib=contract
import "/usr/lib/token.sol"; // -/home/user/work/contract/token.sol
```

#### Absolute imports can be remapped into imports starting with `.` or `..` but they are also relative to the current working directory
Normally imports starting with `.` or `..` are relative to the source directory.

```solidity
// Remappings: /usr/lib=./contract
import "/usr/lib/token.sol"; // -/home/user/work/contract/token.sol
```

```solidity
// Remappings: /usr/lib=../contract
import "/usr/lib/token.sol"; // -/home/user/contract/token.sol
```

#### Only prefix can be remapped
```solidity
// Remappings: /usr=/tmp /lib=/bin contracts=token token.sol=contract.sol
import "/usr/usr/lib/token.sol";                // -/tmp/usr/lib/token.sol
import "contracts/contracts/usr/lib/token.sol"; // -/home/user/work/token/contracts/usr/lib/token.sol
import "token.sol";                             // -/home/user/work/contract.sol
```

#### The remapping with the longest matching prefix is used
```solidity
// Remappings: /usr=/project/dex /usr/lib=/project/token contracts=tokens contracts/token.sol=dex.sol
import "/usr/lib/contracts/token.sol"; // -/project/token/contracts/token.sol
import "contracts/token.sol";          // -/home/user/work/dex.sol
```

#### Remapping is not recursive
```solidity
// Remappings: /a=/b /b=/c /c=/a
import "/a/token.sol"; // -/b/token.sol
```

#### The last remapping takes precedence
```solidity
// Remappings: /a=/b /a=/c /a=/d
import "/a/token.sol"; // -/d/token.sol
```

#### Remappings do not have to match whole path segments
```solidity
// Remappings: /c=/k c=k
import "/contracts/contract.sol"; // -/kontracts/contract.sol
import "contracts/contract.sol";  // -/home/user/work/kontracts/contract.sol
```

#### Leading `.` and `..` in imports cannot be remapped
```solidity
// Remappings: .=/tmp
import "./token/token.sol";   // -/project/token/token.sol
import "../token/token.sol";  // -/project/token.sol
import ".../token/token.sol"; // -/tmp../token/token.sol
```

```solidity
// Remappings: ./token=contract ../token=contract .../token=contract
import "./token/token.sol";   // -/project/token/token.sol
import "../token/token.sol";  // -/project/token.sol
import ".../token/token.sol"; // -/home/user/work/contract/token.sol
```

#### Parts of the path that `.` and `..` get resolved into can get remapped
```solidity
// Remappings: /project=/tmp /token=/tmp /project/token.sol=/tmp/dex.sol
import "./token/token.sol";   // -/tmp/token/token.sol
import "../token/token.sol";  // -/tmp/token.sol
import "./token.sol";         // -/tmp/dex.sol
```

#### Remapping of `.` and `..` works differently when the path to the source file given to the compiler is relative
This is one of the cases where the path to the **file containing imports** (specified on the compiler's command line) affects how the imports are resolved.

The effect is different depending on the locations of the source directory and the current working directory relative to each other.

##### Example: Current working directory is a prefix of the source directory
* Source file passed to the compiler: `../contract.sol`.
* Working directory: `/project/subdir/`.

```solidity
// Remappings: ./token=contract ../token=contract
import "./token/token.sol";   // -/home/user/work/contract/token.sol
import "../token/token.sol";  // -/home/user/work/token.sol
```

#### Paths are not canonicalized before remapping
```solidity
// Remappings: /tmp/code=/usr/lib token=contract
import "/tmp/../tmp/code/token.sol"; // -/tmp/code/token.sol
import "code/../token/token.sol";    // -/home/user/work/token/token.sol
```

#### Remappings ignore any number of leading `.` and `..` segments in imports going beyond the root directory
```solidity
// Remappings: token=contract
import "token/token.sol";                  // -/home/user/work/contract/token.sol
import "../token/token.sol";               // -/home/user/token/token.sol
import "../../token/token.sol";            // -/home/token/token.sol
import "../../../token/token.sol";         // -/token/token.sol
import "../../../../token/token.sol";      // -/home/user/work/contract/token.sol
import "../../../../../token/token.sol";   // -/home/user/work/contract/token.sol
import "../../.././../../token/token.sol"; // -/home/user/work/contract/token.sol
```

#### Remappings ignore any number of leading `.` and `..` segments when the path to the source file given to the compiler is relative
The effect is different depending on the locations of the source directory and the current working directory relative to each other.

##### Example: Current working directory is a prefix of the source directory
* Source file passed to the compiler: `../contract.sol`.
* Working directory: `/project/subdir/`.

```solidity
// Remappings: token=contract
import "token/token.sol";                  // -/home/user/work/contract/token.sol
import "../token/token.sol";               // -/home/user/work/contract/token.sol
import "../../token/token.sol";            // -/home/user/work/contract/token.sol
import "../../../token/token.sol";         // -/home/user/work/contract/token.sol
import "../../../../token/token.sol";      // -/home/user/work/contract/token.sol
import "../../../../../token/token.sol";   // -/home/user/work/contract/token.sol
import "../../.././../../token/token.sol"; // -/home/user/work/contract/token.sol
```

#### Current working directory path cannot be remapped
```solidity
// Remappings: /home/user/work=/workdir
import "token/token.sol"; // -/home/user/work/token/token.sol
```

#### Source directory path can be remapped
```solidity
// Remappings: /project=/tmp
import "./token/token.sol";          // -/tmp/token/token.sol
import "../project/token/token.sol"; // -/tmp/token/token.sol
```

### Base Path
#### Base path does not affect absolute imports
```solidity
// Base path: /base
import "/tmp/code/token.sol";         // -/tmp/code/token.sol
import "/tmp/code/../code/token.sol"; // -/tmp/code/token.sol
```

#### Base path replaces current working directory in relative imports
```solidity
// Base path: /base
import "token/token.sol"; // -/base/token/token.sol
```

#### Base path is prepended to source directory path in imports starting with `.` or `..`
```solidity
// Base path: /base
import "./token/token.sol";  // -/base/project/token/token.sol
import "../token/token.sol"; // -/base/token/token.sol
```

#### Relative imports going beyond root directory ignore `..` and are relative to the base directory
```solidity
// Base path: /base
import "token/token.sol";                  // -/base/token/token.sol
import "../token/token.sol";               // -/base/token/token.sol
import "../../token/token.sol";            // -/base/token/token.sol
import "../../../token/token.sol";         // -/base/token/token.sol
import "../../../../token/token.sol";      // -/base/token/token.sol
import "../../../../../token/token.sol";   // -/base/token/token.sol
import "../../.././../../token/token.sol"; // -/base/token/token.sol
```

#### Base path is prepended to remapped absolute imports
```solidity
// Base path: /base
// Remappings: /tmp/code=/usr/lib
import "/tmp/code/token.sol"; // -/base/usr/lib/token.sol
```

#### Base path replaces current working directory in remapped relative imports
```solidity
// Base path: /base
// Remappings: token=contract
import "token/token.sol"; // -/base/contract/token.sol
```

#### Base path is prepended to relative imports remapped into absolute imports
```solidity
// Base path: /base
// Remappings: token=/usr/lib
import "token/token.sol"; // -/base/usr/lib/token.sol
```

#### Base path replaces current working directory in absolute imports remapped to relative imports
```solidity
// Base path: /base
// Remappings: /usr/lib=contract
import "/usr/lib/token.sol"; // -/base/contract/token.sol
```

#### Base path replaces source directory in absolute imports remapped to relative imports starting with `.` or `..`
Without remapping such imports are relative to the source directory, not the current working directory.

```solidity
// Base path: /base
// Remappings: /usr/lib=./contract
import "/usr/lib/token.sol"; // -/base/contract/token.sol
```

```solidity
// Base path: /base
// Remappings: /usr/lib=../contract
import "/usr/lib/token.sol"; // -/contract/token.sol
```

#### Parts of the path that `.` and `..` get resolved into can get remapped when base path is used and such remappings don't override the base path
```solidity
// Base path: /base
// Remappings: /project=/tmp /token=/tmp /project/token.sol=/tmp/dex.sol
import "./token/token.sol";   // -/base/tmp/token/token.sol
import "../token/token.sol";  // -/base/tmp/token.sol
import "./token.sol";         // -/base/tmp/dex.sol
```

#### Base path cannot be remapped
```solidity
// Base path: /base
// Remappings: /base=/tmp
import "token/token.sol"; // -/base/token/token.sol
```

#### Source directory path can be remapped even when base path is used
```solidity
// Base path: /base
// Remappings: /project=/tmp /base=/usr/lib
import "./token/token.sol";          // -/base/tmp/token/token.sol
import "../project/token/token.sol"; // -/base/tmp/token/token.sol
```

#### Relative base path is relative to the current working directory
```solidity
// Base path: base
import "token/token.sol";    // -/home/user/work/base/token/token.sol
import "./token/token.sol";  // -/home/user/work/base/project/token/token.sol
import "../token/token.sol"; // -/home/user/work/base/token/token.sol
```

```solidity
// Base path: .
import "token/token.sol";    // -/home/user/work/token/token.sol
import "./token/token.sol";  // -/home/user/work/project/token/token.sol
import "../token/token.sol"; // -/home/user/work/token/token.sol
```

```solidity
// Base path: ..
import "token/token.sol";    // -/home/user/token/token.sol
import "./token/token.sol";  // -/home/user/project/token/token.sol
import "../token/token.sol"; // -/home/user/token/token.sol
```

### Standard input
#### Standard input can be used as if it was an actual file called `<stdin>` if requested
```solidity
// Command line: contains `-`
import "<stdin>"; // -stdin
```

If there is an actual file called in the current directory, it's ignored.

#### A file called `<stdin>` from current working directory can only be imported if use of the standard input is not requested.
```solidity
// Command line: does not contain `-`
import "<stdin>"; // -/home/user/work/<stdin>
```

#### `<stdin>` can be remapped to a path when it represents standard input
```solidity
// Command line: contains `-`
// Remappings: <stdin>=/tmp/code/token.sol
import "<stdin>"; // -/tmp/code/token.sol
```

#### `<stdin>` can be remapped to a path when it represents a file
```solidity
// Command line: does not contain `-`
// Remappings: <stdin>=/tmp/code/token.sol
import "<stdin>"; // -/tmp/code/token.sol
```

#### Paths can be remapped to `<stdin>` when it represents standard input
```solidity
// Command line: contains `-`
// Remappings: /tmp/code/token.sol=<stdin>
import "/tmp/code/token.sol"; // -stdin
```

#### Paths can be remapped to `<stdin>` when it represents a file
```solidity
// Command line: does not contain `-`
// Remappings: /tmp/code/token.sol=<stdin>
import "/tmp/code/token.sol"; // -/home/user/work/<stdin>
```

#### Base path does not affect `<stdin>` when it represents standard input
```solidity
// Command line: contains `-`
// Base path: /base
import "<stdin>"; // -stdin
```

#### Base path affects `<stdin>` when it does not represent standard input
```solidity
// Command line: does not contain `-`
// Base path: /base
import "<stdin>"; // -/base/<stdin>
```

### Extra Slashes
#### Multiple slashes between path segments in imports are squashed into one
```solidity
import "/tmp//code/token.sol";   // -/tmp/code/token.sol
import "/tmp////code/token.sol"; // -/tmp/code/token.sol
```

#### Trailing slashes in imports are replaced with `/.`
```solidity
import "/tmp/code/token.sol/";    // -/tmp/code/token.sol/.
import "/tmp/code/token.sol//";   // -/tmp/code/token.sol/.
import "/tmp/code/token.sol////"; // -/tmp/code/token.sol/.
```

#### Trailing slashes in relative imports that start with `.` or `..` imports are stripped
```solidity
import "./token/token.sol/";     // -/project/token/token.sol
import "./token/token.sol//";    // -/project/token/token.sol
import "./token/token.sol////";  // -/project/token/token.sol
import "../token/token.sol/";    // -/project/token.sol
import "../token/token.sol//";   // -/project/token.sol
import "../token/token.sol////"; // -/project/token.sol
```

#### Two Leading slashes in imports are preserved
```solidity
import "//tmp/code/token.sol";    // -//tmp/code/token.sol
```

#### More than two Leading slashes in imports are squashed into one
```solidity
import "///tmp/code/token.sol";    // -/tmp/code/token.sol
import "////tmp/code/token.sol";   // -/tmp/code/token.sol
import "/////tmp/code/token.sol";  // -/tmp/code/token.sol
```

#### Slashes after the leading `.` or `..` in relative imports are squashed into one
```solidity
import "./token/token.sol";      // -/project/token/token.sol
import ".//token/token.sol";     // -/project/token/token.sol
import ".///token/token.sol";    // -/project/token/token.sol
import ".////token/token.sol";   // -/project/token/token.sol
import "..//token/token.sol";    // -/token/token.sol
import "..///token/token.sol";   // -/token/token.sol
import "..////token/token.sol";  // -/token/token.sol
import "../////token/token.sol"; // -/token/token.sol
```

#### Slashes in base path follow the same rules as slashes in imports
```solidity
// Base path: ////base1////base2/////base3////
import "token/token.sol"; // -/base1/base2/base3/token/token.sol
```

#### Slashes in remappings must match exactly
```solidity
// Remappings: /tmp/code=/usr/lib
import "/tmp//code/token.sol";    // -/tmp/code/token.sol
import "/tmp/code/token.sol/";    // -/usr/lib/token.sol/
import "/tmp/code/token.sol//";   // -/usr/lib/token.sol/
import "//tmp/code/token.sol";    // -//tmp/code/token.sol
import "///tmp/code/token.sol";   // -/tmp/code/token.sol
```

```solidity
// Remappings: /tmp/code/token.sol//=/usr/lib/token.sol
import "/tmp/code/token.sol";    // -/usr/lib/token.sol
import "/tmp/code/token.sol/";   // -/usr/lib/token.sol/.
import "/tmp/code/token.sol//";  // -/usr/lib/token.sol
import "/tmp/code/token.sol///"; // -/usr/lib/token.sol/.
```

#### Multiple trailing slashes in remapping targets are squashed into one
```solidity
// Remappings: /tmp/code=////usr////lib////
import "/tmp/code/token.sol"; // -/usr/lib/token.sol
```

### URLs
#### `file://` prefix is stripped from paths
```solidity
import "file:///tmp/code/token.sol"; // -/tmp/code/token.sol
import "/tmp/code/file://token.sol"; // -/tmp/code/file:/token.sol
```

#### Other protocol prefixes are not stripped
```solidity
import "http:///tmp/code/token.sol";  // -/home/user/work/http:/tmp/code/token.sol
import "https:///tmp/code/token.sol"; // -/home/user/work/https:/tmp/code/token.sol
import "ftp:///tmp/code/token.sol";   // -/home/user/work/ftp:/tmp/code/token.sol
```

#### Paths that become relative after stripping of `file://` are always relative to the current working directory
```solidity
import "file://token.sol";    // -/home/user/work/token.sol
import "file://./token.sol";  // -/home/user/work/token.sol
import "file://../token.sol"; // -/home/user/token.sol
```

#### Base path is prepended to paths that become relative after stripping of `file://`
```solidity
// Base path: /base
import "file://token.sol";    // -/base/home/user/work/token.sol
```

#### Prepending base path to paths that start with `.` or `..` after stripping of `file://` makes them relative to the source directory
```solidity
// Base path: /base
import "file://./token.sol";  // -/base/project/token.sol
import "file://../token.sol"; // -/base/token.sol
```

#### `file://` prefix as a whole cannot be remapped
```solidity
// Remappings: file://=/usr/lib
import "file:///tmp/code/token.sol"; // -/tmp/code/token.sol
```

```solidity
// Remappings: file:/=/usr/lib
import "file:///tmp/code/token.sol"; // -/tmp/code/token.sol
```

#### Fragments of the `file://` prefix can be remapped
```solidity
// Remappings: file=/usr/lib
import "file:///tmp/code/token.sol"; // -/usr/lib:/tmp/code/token.sol
```

```solidity
// Remappings: f=/usr/lib
import "file:///tmp/code/token.sol"; // -/usr/libile:/tmp/code/token.sol
```

