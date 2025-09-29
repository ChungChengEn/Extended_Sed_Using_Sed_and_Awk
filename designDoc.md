# Extended Sed (msed) - Design Document

## 1. Project Overview

### 1.1 Purpose
Extended Sed (msed) is a meta-preprocessor that extends the standard `sed` stream editor with additional commands and features. It translates enhanced sed syntax into standard sed commands, providing a more powerful and intuitive interface while maintaining full compatibility with traditional sed output.

### 1.2 Goals
- Extend sed with useful commands that simplify common operations
- Support argument substitution in sed scripts
- Enable reverse line addressing (counting from file end)
- Maintain compatibility with standard sed
- Provide two implementation strategies (sed-based and awk-based)

### 1.3 Non-Goals
- Replace sed entirely
- Provide a completely new syntax
- Support all possible edge cases of complex nested structures

## 2. Architecture

### 2.1 High-Level Architecture

```
Input Stream → Capture → Argument Processing → Script Preprocessing → 
  Line Reconstruction → Label Generation → Standard Sed Execution → Output
```

### 2.2 Component Overview

#### 2.2.1 Input Capture (Line 3)
- Reads all stdin into temporary file `t4`
- Allows multiple passes over input
- Enables reverse line addressing

#### 2.2.2 Script Initialization (Line 8)
- Stores the sed script ($1) in file `t5`
- Adds leading space to facilitate processing
- Prepares script for argument substitution

#### 2.2.3 Argument Substitution Loop (Lines 10-22)
- Iterates through arguments $2, $3, ... $N
- Processes in reverse order to avoid substitution conflicts
- Replaces `$N` placeholders with actual argument values
- Handles escaped `\$N` to preserve literal text
- Preserves flags (arguments starting with `-`)

#### 2.2.4 Semicolon Processing (Line 30)
- Converts semicolons to line separators
- Preserves escaped semicolons (`\;`) using form feed (`\f`)
- Removes leading space added in initialization

#### 2.2.5 Script Extraction (Line 33/35)
- Extracts processing rules from the script itself
- Creates sed/awk program in `t7`
- Self-contained design: processing rules embedded in script

#### 2.2.6 Command Reconstruction (Line 38/40)
- Processes the semicolon-separated commands
- Reconstructs multi-part commands (`s`, `y`, `\`, `/`)
- Handles commands that legitimately contain semicolons
- Outputs structured command list to `t8`

#### 2.2.7 Reverse Line Addressing (Lines 46-64)
- Detects `$-N` syntax in commands
- Calculates actual line numbers based on input file size
- Substitutes calculated line numbers into commands

#### 2.2.8 Label Uniquification (Lines 63-66)
- Makes branch labels unique using counter
- Prevents conflicts in scripts with multiple branch commands
- Generates pattern: `label7N` where N is counter

#### 2.2.9 Final Execution (Lines 70-72)
- Replaces $1 with processed script file
- Executes standard sed with all arguments
- Outputs results

## 3. Extended Command Set

### 3.1 Command Specifications

#### Z - Delete First Line
- **Syntax**: `Z`
- **Translation**: `s/[^\n]*\n\{,1\}//`
- **Function**: Deletes first line including its newline
- **Use Case**: Multi-line pattern space manipulation

#### C - Replace Pattern Space
- **Syntax**: `C<text>`
- **Translation**: `s/.*/<text>/`
- **Function**: Replaces entire pattern space with specified text
- **Use Case**: Cleaner whole-line replacement than `s/.*/text/`

#### W - Append to File
- **Syntax**: `W<filename>`
- **Translation**: Complex multi-command sequence
- **Function**: Appends pattern space to file with proper newline handling
- **Implementation**: Uses hold space to preserve state
  ```
  {s/.*/\v\&/;H;s/\n.*//;s/\v//;w<filename>
  g;s/\n\v.*//;x;s/.*\v//}
  ```

#### D - Enhanced Delete
- **Syntax**: `D`
- **Translation**: `{/\n/!s/.*/\&\n/;D;}`
- **Function**: Enhanced multi-line delete with better behavior
- **Use Case**: Complex multi-line processing

#### f - Flag Operation
- **Syntax**: `f`
- **Translation**: `s/.*/\&/`
- **Function**: Pattern space flag operation
- **Use Case**: State management in complex scripts

#### F - Branch with Label
- **Syntax**: `F`
- **Translation**: `tlabel7N;:label7N` (N = unique counter)
- **Function**: Conditional branch with automatic unique label
- **Use Case**: Complex flow control without manual label management

### 3.2 State Preservation

Commands Z, C, W, and D include automatic hold space and flag preservation:

```awk
# Before command execution
TflagL1; x; s/$/\r\v/; x; :flagL1

# After command execution  
TflagL2; :flagL2; x; s/\r\v$//; x
```

This ensures that user flags and hold space content are not corrupted by extended commands.

## 4. Special Features

### 4.1 Argument Substitution

**Syntax**: `$N` where N is 2, 3, 4, ...

**Processing**:
1. Arguments processed in reverse order (high to low)
2. Prevents cascading substitution issues
3. Escaped form `\$N` produces literal `$N`

**Example**:
```bash
msed 's/foo/$2/;s/bar/$3/' "replacement1" "replacement2"
```

### 4.2 Reverse Line Addressing

**Syntax**: `$-N` where N is 0, 1, 2, ...

**Semantics**:
- `$-0` = last line of file
- `$-1` = second-to-last line
- `$-N` = (total_lines - N)

**Implementation**:
1. Count lines in input file (stored in `t4`)
2. Calculate: actual_line = total_lines - N
3. Substitute calculated value into command

**Example**:
```bash
# Print last 5 lines
msed '$-4,$p' -n < file.txt
```

### 4.3 Escape Handling

**Special Characters**:
- `\;` → Literal semicolon (preserved as `\f` during processing)
- `\\` → Literal backslash (preserved as `\a` during processing)
- `\$N` → Literal `$N` string

**Processing Pipeline**:
1. Convert `\\` to `\a` (bell character)
2. Convert `\;` to `\f` (form feed)
3. Process commands normally
4. Convert back: `\a` → `\\`, `\f` → `\;`

## 5. Implementation Differences

### 5.1 Sed+Csh Version

**Characteristics**:
- Pure sed for all text processing
- Uses sed's BRE (Basic Regular Expressions)
- More portable across Unix systems
- Simpler processing logic

**Key Processing**:
```bash
# Argument substitution using sed
sed '/\\\\/s//\a/g;s/\([^\]\)$'$i'/\1'$argv[$i]'/g;s/\\$'$i'/$'$i'/g'
```

**Command Reconstruction**:
- Uses sed branching and labels extensively
- Pattern matching with BRE
- Multiple branch labels: `incomplete`, `incomplete1`, `incomplete2`, etc.

### 5.2 Awk+Csh Version

**Characteristics**:
- Uses awk for complex pattern matching
- Leverages awk's string functions
- More powerful text manipulation
- Better handling of edge cases

**Key Processing**:
```bash
# Argument substitution using awk
awk '/\\\\/{gsub(/\\\\/,"\a");}/[^\\]\$/{...}/\\\$/{...}1'
```

**Command Reconstruction**:
- Uses awk's `match()`, `substr()`, `getline`
- More sophisticated pattern detection
- Cleaner handling of complex cases

**Advantages**:
- Better string manipulation
- More readable logic for complex patterns
- Enhanced flag management with `flagL1`, `flagL2`
- More robust handling of incomplete commands

## 6. Processing Details

### 6.1 Command Line Reconstruction

**Problem**: Semicolons within commands (like `s/;/X/` or `y/;/,/`) get split into separate lines.

**Solution**: Detect incomplete commands and reassemble them.

**Detection Patterns**:
- `s` command: needs 2 unescaped delimiters after the `s`
- `y` command: needs 2 unescaped delimiters after the `y`
- `\x` command: needs 1 unescaped delimiter after the backslash
- `/` pattern: needs 1 unescaped `/` to close

**Reconstruction Algorithm**:
1. Identify command type
2. Count delimiters
3. If incomplete, read next line(s)
4. Continue until complete command assembled

### 6.2 Predicate Handling

**Challenge**: Commands like `1,10s/a/b/` or `/pattern/d` have predicates before the command.

**Solution**: Insert `\v` marker after predicate to separate it from command.

**Patterns Handled**:
- Line numbers: `5`, `1,10`
- Regular expressions: `/pattern/`, `/start/,/end/`
- Line address plus pattern: `5,/end/`
- Reverse addresses: `$-5,$`

**Processing**:
1. Detect predicate pattern
2. Find end of predicate
3. Insert `\v` marker
4. Remove `\v` after processing

### 6.3 Brace Handling

**Feature**: Support for `{` to group commands.

**Processing**:
```
/pattern/{command1;command2}
```

becomes:
```
/pattern/{
command1;command2}
```

**Implementation**:
1. Detect `\v{` pattern
2. Print everything up to and including `{`
3. Move rest to next line
4. Remove `\v` markers

## 7. Temporary File Management

### 7.1 File Purposes

| File | Purpose |
|------|---------|
| `t4` | Captured stdin input |
| `t5` | Script with arguments being substituted |
| `t6` | Updated script after each operation |
| `t7` | Extracted processing rules (sed/awk code) |
| `t8` | Preprocessed command structure |
| `t9` | Final standard sed script |

### 7.2 Cleanup Considerations

**Current Behavior**: Files persist after execution

**Potential Improvements**:
- Add trap to clean up on exit
- Use `mktemp` for unique names
- Option to preserve for debugging

## 8. Edge Cases and Limitations

### 8.1 Known Limitations

1. **Character classes**: `[;]` not specially handled
2. **Nested structures**: Complex nesting may require careful escaping
3. **Backslash-heavy regex**: May need additional escaping
4. **Argument count**: No explicit limit, but large N may be slow

### 8.2 Unsupported Patterns

- Character class with semicolons: `[abc;def]`
- Extremely complex nested escape sequences
- Binary data in pattern space

### 8.3 Error Handling

**Current State**: Minimal error checking

**Improvement Opportunities**:
- Validate argument count
- Check for file write permissions
- Detect malformed extended commands
- Provide meaningful error messages

## 9. Performance Considerations

### 9.1 Time Complexity

- Argument substitution: O(N × M) where N = arguments, M = script size
- Line reconstruction: O(L) where L = number of commands
- Reverse addressing: O(C × F) where C = commands with $-, F = file lines

### 9.2 Space Complexity

- Input file stored entirely in memory (t4)
- Multiple temporary files created
- Not suitable for extremely large files

### 9.3 Optimization Opportunities

- Stream processing for large files (avoid full capture)
- Parallel argument substitution
- Caching of processed scripts

## 10. Future Enhancements

### 10.1 Potential Features

1. **Additional commands**: More extended commands (e.g., conditional operations)
2. **Named arguments**: Support `$name` instead of just `$N`
3. **Include files**: `@include "file.msed"`
4. **Macros**: Define reusable sed fragments
5. **Better errors**: Comprehensive error messages with line numbers

### 10.2 Technical Improvements

1. **Streaming mode**: Process large files without full capture
2. **Caching**: Cache processed scripts for repeated use
3. **Debugging mode**: Verbose output of transformation steps
4. **Test suite**: Comprehensive test coverage
5. **Documentation**: Man page in standard Unix format

## 11. Testing Strategy

### 11.1 Test Categories

1. **Basic functionality**: Each extended command in isolation
2. **Argument substitution**: Various argument patterns
3. **Reverse addressing**: Edge cases (first line, last line, out of bounds)
4. **Escape handling**: All special characters
5. **Complex scripts**: Real-world use cases
6. **Edge cases**: Empty files, single-line files, very long lines

### 11.2 Test Examples

```bash
# Test Z command
echo -e "line1\nline2\nline3" | msed 'Z'

# Test argument substitution
echo "test" | msed 's/test/$2/' "replacement"

# Test reverse addressing
seq 1 10 | msed '$-2,$p' -n

# Test escaping
echo "a;b" | msed 's/\;/,/'
```

## 12. Conclusion

Extended Sed provides a powerful enhancement to the classic sed utility while maintaining compatibility and portability. The two-implementation approach (sed+csh and awk+csh) offers flexibility for different environments and use cases. The design prioritizes clarity and maintainability while solving real-world text processing challenges.