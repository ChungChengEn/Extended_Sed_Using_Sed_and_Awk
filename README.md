# Extended Sed (msed)

An enhanced version of the standard `sed` stream editor that adds powerful extensions while maintaining compatibility with traditional sed syntax. Available in two implementations: one using sed+csh, another using awk+csh.

## Overview

`msed` (meta-sed) extends the standard sed utility with additional commands and features that make complex text processing tasks more intuitive and powerful. The tool preprocesses extended sed scripts into standard sed commands, allowing you to use enhanced syntax while producing portable, standard sed output.

## Features

### Extended Commands

- **`Z`** - Delete first line including newline (simplifies multi-line processing)
- **`C<text>`** - Replace pattern space with text (cleaner than `s/.*/text/`)
- **`W<file>`** - Append pattern space to file with proper newline handling
- **`D`** - Enhanced delete command with better multi-line behavior
- **`f`** - Flag operation for pattern space manipulation
- **`F`** - Branch with automatic label management

### Advanced Features

- **Argument substitution**: Use `$2`, `$3`, etc. in your sed scripts to reference command-line arguments
- **Reverse line addressing**: Use `$-N` syntax to address lines from the end of the file
  - `$-0` = last line
  - `$-1` = second-to-last line
  - `$-N` = Nth line from the end
- **Automatic label generation**: The `F` command generates unique labels automatically
- **Enhanced semicolon handling**: Better support for semicolons within commands
- **Escaped semicolon support**: Use `\;` for literal semicolons

## Installation

### Prerequisites

- C shell (csh or tcsh)
- sed (standard version)
- awk (for the awk+csh implementation)

### Setup

1. Clone the repository:
```bash
git clone https://github.com/yourusername/extended-sed.git
cd extended-sed
```

2. Make the script executable:
```bash
chmod +x msed
```

3. (Optional) Add to your PATH or create a symlink:
```bash
sudo ln -s $(pwd)/msed /usr/local/bin/msed
```

## Usage

Basic syntax:
```bash
msed 'sed_script' [arguments...] < input_file
cat input_file | msed 'sed_script' [arguments...]
```

### Examples

#### Example 1: Argument Substitution
```bash
# Replace "old" with a command-line argument
echo "hello old world" | msed 's/old/$2/' "new"
# Output: hello new world
```

#### Example 2: Reverse Line Addressing
```bash
# Print the last 3 lines of a file
msed '$-2,$p' -n < file.txt

# Delete the last line
msed '$-0d' < file.txt
```

#### Example 3: Using Extended Commands
```bash
# Replace entire line with fixed text
echo -e "line1\nline2\nline3" | msed '2Creplacement'
# Output:
# line1
# replacement
# line3

# Delete first line including newline
echo -e "line1\nline2\nline3" | msed 'Z'
# Output:
# line2
# line3
```

#### Example 4: Multiple Arguments
```bash
# Use multiple arguments in substitution
echo "foo bar baz" | msed 's/foo/$2/;s/bar/$3/' "NEW1" "NEW2"
# Output: NEW1 NEW2 baz
```

## Implementation Details

### Two Versions Available

1. **sed+csh version**: Pure sed implementation for maximum compatibility
2. **awk+csh version**: Uses awk for enhanced pattern matching and processing

Both versions produce identical results but may have different performance characteristics depending on your use case.

### Processing Pipeline

1. **Input capture**: Stdin is saved to temporary file `t4`
2. **Argument substitution**: Command-line arguments replace `$N` placeholders
3. **Script preprocessing**: Extended commands are translated to standard sed
4. **Line reconstruction**: Commands split by semicolons are properly reassembled
5. **Label generation**: Unique labels are created for branch commands
6. **Execution**: The processed script runs against the captured input

### Temporary Files

The script creates temporary files in the current directory:
- `t4`: Captured input
- `t5`, `t6`: Intermediate processing
- `t7`: Extracted sed/awk processing rules
- `t8`: Preprocessed command structure
- `t9`: Final sed script

These files are automatically managed but may persist if the script is interrupted.

## Technical Notes

### Footnote 1: Argument Processing
Arguments are processed in reverse order (from last to first) to prevent earlier substitutions from interfering with later ones.

### Footnote 2: Escaping
- Use `\$N` to include a literal `$N` in your script
- Use `\;` for literal semicolons
- Backslash escaping follows standard sed conventions

### Footnote 3: Limitations
- Character classes like `[;]` are not specially handled
- Complex nested structures may require careful escaping
- Some edge cases with backslash-heavy regex may need adjustment

### Footnote 4: Branch Labels
The `F` command uses automatic label generation with the pattern `label7N` where N is a counter. This prevents label conflicts in complex scripts.

## Differences Between Implementations

### sed+csh version
- Pure sed processing
- More portable across systems
- Uses sed's native pattern matching
- Better for traditional sed users

### awk+csh version
- Leverages awk's powerful text processing
- More flexible string manipulation
- Enhanced pattern matching capabilities
- Better handling of complex edge cases
- Additional flags: `flagL1`, `flagL2`

## Acknowledgments

This project extends the classic sed utility with modern conveniences while respecting the elegance of the original design.