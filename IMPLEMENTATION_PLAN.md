# DXL DOORS to JSON Export - Implementation Plan

## Project Requirements Summary

**Goal**: Create a DXL script for IBM DOORS 9.5 that exports a module and its linked modules to a single JSON file.

**Key Requirements**:
- Command-line batch mode execution
- Takes module path as argument
- Exports all non-empty object attributes
- Preserves object hierarchy (nested children)
- Embeds links/relationships within each object
- Follows links to other modules (max 2 levels: original + 1 level deep)
- Outputs to `output.json` in script directory
- Exports all link types
- JSON will feed a Python script for SQLite database creation

---

## Step-by-Step Implementation Plan

### Phase 1: Script Setup & Structure

#### Step 1.1: Create Main Script File
- **File**: `export_to_json.dxl`
- **Purpose**: Main DXL script entry point
- **Actions**:
  - Set up DXL script header/comments
  - Define global variables for configuration
  - Set max depth constant: `MAX_DEPTH = 2`

#### Step 1.2: Implement Argument Handling
- **Actions**:
  - Check command-line arguments for module path
  - Validate that module path is provided
  - Display usage message if arguments invalid
  - Store module path in variable

#### Step 1.3: Define Data Structures
- **Actions**:
  - Create Skip list to track visited modules (prevent duplicates)
  - Create Buffer/string variable for JSON output
  - Define counters: current depth, object count, module count

---

### Phase 2: Module & Object Processing

#### Step 2.1: Implement Module Opening Function
- **Function**: `Module openModuleReadOnly(string modulePath)`
- **Actions**:
  - Attempt to open module in read-only mode
  - Handle errors if module doesn't exist or can't be opened
  - Return module handle or null

#### Step 2.2: Implement Object Iteration Function
- **Function**: `void processObject(Object obj, int depth, Buffer jsonBuffer)`
- **Actions**:
  - Extract object identifier
  - Extract all non-empty attributes dynamically
  - Determine object level/hierarchy position
  - Store object data in temporary structure

#### Step 2.3: Implement Attribute Extraction
- **Function**: `void extractAttributes(Object obj, Buffer attrBuffer)`
- **Actions**:
  - Iterate through all attributes of object
  - Check if attribute value is non-empty
  - Handle different attribute types (string, int, date, rich text)
  - Escape special JSON characters in strings
  - Add attribute key-value pairs to buffer

---

### Phase 3: Hierarchy & Relationship Handling

#### Step 3.1: Implement Hierarchy Preservation
- **Actions**:
  - Track parent-child relationships during traversal
  - Build nested JSON structure with "children" arrays
  - Recursively process child objects
  - Maintain proper indentation/nesting levels

#### Step 3.2: Implement Link Detection
- **Function**: `void extractLinks(Object obj, Buffer linkBuffer)`
- **Actions**:
  - Get all outgoing links from current object
  - Get all incoming links to current object
  - For each link, extract:
    - Link type/module name
    - Source object ID
    - Target object ID
    - Target module path
  - Add links to "links" array in object JSON

#### Step 3.3: Implement Cross-Module Link Following
- **Function**: `void followLinksToModules(Object obj, int currentDepth)`
- **Actions**:
  - Check if currentDepth < MAX_DEPTH
  - Identify links pointing to other modules
  - Extract target module paths
  - Check if module already visited (Skip list)
  - If not visited and within depth, queue for processing

---

### Phase 4: Module Traversal & Export

#### Step 4.1: Implement Recursive Module Processor
- **Function**: `void processModule(string modulePath, int depth, Buffer jsonBuffer)`
- **Actions**:
  - Check depth limit (if depth >= MAX_DEPTH, return)
  - Check if module already in Skip list
  - Add module to Skip list
  - Open module
  - Create module JSON object with metadata (name, path)
  - Iterate all objects in module
  - For each object:
    - Process object and extract attributes
    - Extract links
    - Process child objects recursively
    - Identify linked modules
  - Close module
  - Return linked module paths for further processing

#### Step 4.2: Implement Main Export Loop
- **Actions**:
  - Start with initial module (depth = 0)
  - Process initial module
  - Collect linked module paths
  - For each linked module (depth = 1):
    - Process linked module
    - Do NOT follow further links (max depth reached)
  - Build complete JSON structure

---

### Phase 5: JSON Generation

#### Step 5.1: Design JSON Structure
```json
{
  "exportDate": "2026-01-06T15:04:00Z",
  "rootModule": "/path/to/module",
  "maxDepth": 2,
  "modules": [
    {
      "modulePath": "/path/to/module",
      "moduleName": "Requirements Module",
      "depth": 0,
      "objects": [
        {
          "id": "123",
          "level": 1,
          "heading": "System Requirements",
          "text": "The system shall...",
          "customAttr1": "value1",
          "links": [
            {
              "type": "satisfies",
              "direction": "outgoing",
              "targetModule": "/path/to/design",
              "targetObjectId": "456"
            }
          ],
          "children": [
            {
              "id": "124",
              "level": 2,
              ...
            }
          ]
        }
      ]
    }
  ]
}
```

#### Step 5.2: Implement JSON Builder Functions
- **Function**: `string escapeJSON(string value)`
  - Escape quotes, backslashes, newlines, tabs

- **Function**: `void appendJSONObject(Buffer buf, Object obj, int indent)`
  - Build object JSON with proper formatting

- **Function**: `void appendJSONArray(Buffer buf, string arrayName, int indent)`
  - Handle JSON array formatting

#### Step 5.3: Implement JSON File Writer
- **Function**: `void writeJSONFile(Buffer jsonBuffer, string filename)`
- **Actions**:
  - Open file for writing in script directory
  - Write buffer contents to file
  - Close file
  - Display success message with file location

---

### Phase 6: Error Handling & Logging

#### Step 6.1: Implement Error Handling
- **Actions**:
  - Handle module not found errors
  - Handle file write permission errors
  - Handle invalid object references
  - Provide meaningful error messages to user

#### Step 6.2: Implement Progress Logging
- **Actions**:
  - Print start message with module path
  - Print progress updates (e.g., "Processing module X...")
  - Print object count, module count
  - Print completion message with output file path

---

### Phase 7: Testing & Validation

#### Step 7.1: Unit Testing Scenarios
- Test with single module (no links)
- Test with module containing links to 1 other module
- Test with circular references (A→B→A)
- Test with deep hierarchy (many nested levels)
- Test with various attribute types
- Test with empty/null attributes (should skip)
- Test with special characters in text

#### Step 7.2: Integration Testing
- Verify JSON is valid (use JSON validator)
- Verify all objects exported
- Verify links correctly captured
- Verify hierarchy preserved
- Verify max depth respected

#### Step 7.3: Python Script Compatibility Test
- Feed output.json to Python SQLite script
- Verify database created correctly
- Verify relationships captured properly

---

## Implementation Order

1. ✅ Create main script file with basic structure
2. ✅ Implement argument handling and validation
3. ✅ Implement module opening function
4. ✅ Implement basic object iteration (no links, no hierarchy)
5. ✅ Implement attribute extraction for single object
6. ✅ Implement JSON escaping utility
7. ✅ Test basic single-object export
8. ✅ Implement hierarchy preservation (parent-child)
9. ✅ Test hierarchical export
10. ✅ Implement link extraction (within module)
11. ✅ Test link export
12. ✅ Implement cross-module link detection
13. ✅ Implement visited module tracking (Skip list)
14. ✅ Implement recursive module processing with depth limit
15. ✅ Test multi-module export
16. ✅ Implement complete JSON structure generation
17. ✅ Implement file writing
18. ✅ Add error handling throughout
19. ✅ Add progress logging
20. ✅ Final integration testing

---

## Key DXL APIs to Use

### Module Operations
- `read(string path, false)` - Open module read-only
- `close(Module m)` - Close module
- `name(Module m)` - Get module name
- `fullName(Module m)` - Get full module path

### Object Iteration
- `for obj in entire module` - Iterate all objects
- `for child in obj` - Iterate children of object
- `level(Object obj)` - Get object level
- `identifier(Object obj)` - Get object ID

### Attribute Access
- `for attr in obj` - Iterate attributes
- `name(Attribute attr)` - Get attribute name
- `obj.attr_name` - Get attribute value
- `null(value)` - Check if value is null

### Link Operations
- `for lnk in obj -> "*"` - Outgoing links
- `for lnk in obj <- "*"` - Incoming links
- `module(Link lnk)` - Get target module
- `target(Link lnk)` - Get target object
- `source(Link lnk)` - Get source object

### String/Buffer Operations
- `Buffer buf = create()` - Create buffer
- `buf += "string"` - Append to buffer
- `tempStringOf(buf)` - Convert buffer to string

---

## Potential Challenges & Solutions

### Challenge 1: Large Modules / Memory
**Problem**: Very large modules may cause memory issues
**Solution**: Process objects in batches, write intermediate results

### Challenge 2: Rich Text Formatting
**Problem**: DOORS rich text contains HTML-like formatting
**Solution**: Strip formatting or preserve as escaped string

### Challenge 3: Binary Attributes
**Problem**: Some attributes may be binary (OLE objects, images)
**Solution**: Skip binary attributes or encode as base64

### Challenge 4: Performance
**Problem**: Deep recursion and many modules may be slow
**Solution**: Show progress indicators, optimize Skip list lookups

---

## Deliverables

1. `export_to_json.dxl` - Main DXL script
2. `output.json` - Sample output file (for testing)
3. `README.md` - Usage instructions
4. `TEST_RESULTS.md` - Test documentation

---

## Next Steps

1. Review this plan with user
2. Get confirmation on any unclear aspects
3. Begin implementation following the order above
4. Test incrementally after each major feature
5. Deliver working DXL script

