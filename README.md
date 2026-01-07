# DOORS to JSON Export Script

A DXL script for IBM DOORS 9.5 that exports module data and relationships to JSON format for downstream processing.

## Overview

This script exports a DOORS module and its linked modules to a single JSON file, preserving:
- Object hierarchy (parent-child relationships)
- All non-empty object attributes
- Links between objects (both within and across modules)
- Module metadata

The output JSON is designed to feed into a Python script for SQLite database creation.

## Features

- **Hierarchical Export**: Preserves DOORS object tree structure with nested children
- **Complete Attribute Export**: Dynamically extracts all non-empty attributes of any type
- **Relationship Tracking**: Captures both incoming and outgoing links with full context
- **Cross-Module Support**: Automatically follows links to other modules (up to 2 levels deep)
- **Duplicate Prevention**: Tracks visited modules to avoid redundant exports
- **Progress Logging**: Displays processing status and statistics
- **Error Handling**: Gracefully handles missing modules and invalid paths

## Requirements

- IBM DOORS 9.5 or compatible version
- Read access to the modules you want to export
- Write permissions in the script directory (for output.json)

## Installation

1. Clone or download this repository
2. Place `export_to_json.dxl` in a convenient location
3. No additional installation required

## Usage

### Basic Usage

Run the script in DOORS batch mode with the module path as an argument:

```bash
doors -batch export_to_json.dxl "/Project/Requirements"
```

### Command Syntax

```bash
doors -batch <path_to_script>/export_to_json.dxl "<module_path>"
```

**Parameters:**
- `<module_path>`: Full path to the DOORS module to export (required)

**Example:**

```bash
doors -batch /home/user/scripts/export_to_json.dxl "/MyProject/SystemRequirements"
```

### Output

The script creates `output.json` in the same directory as the script file.

## Configuration

You can modify these constants in the script:

```dxl
const int MAX_DEPTH = 2           // Original module (0) + 1 level of linked modules
const string OUTPUT_FILE = "output.json"  // Output filename
```

### Depth Levels Explained

- **Depth 0**: The original module specified as argument
- **Depth 1**: Modules linked from the original module
- **Depth 2+**: Not processed (configurable via MAX_DEPTH)

## Output Format

### JSON Structure

```json
{
  "exportDate": "Mon Jan 06 15:30:00 2026",
  "rootModule": "/Project/Requirements",
  "maxDepth": 2,
  "modules": [
    {
      "modulePath": "/Project/Requirements",
      "moduleName": "System Requirements",
      "depth": 0,
      "objects": [
        {
          "id": "123",
          "level": 1,
          "absoluteNumber": "1",
          "Object Heading": "Safety Requirements",
          "Object Text": "The system shall ensure...",
          "Created By": "John Doe",
          "Created On": "2024-01-15",
          "links": [
            {
              "type": "satisfies",
              "direction": "outgoing",
              "targetModule": "/Project/Design",
              "targetObjectId": "456"
            }
          ],
          "children": [
            {
              "id": "124",
              "level": 2,
              "absoluteNumber": "1.1",
              "Object Heading": "Hardware Safety",
              "Object Text": "Hardware components shall...",
              "links": [],
              "children": []
            }
          ]
        }
      ]
    },
    {
      "modulePath": "/Project/Design",
      "moduleName": "System Design",
      "depth": 1,
      "objects": [...]
    }
  ]
}
```

### Object Structure

Each object contains:

- **id**: DOORS object identifier
- **level**: Hierarchy level within the module
- **absoluteNumber**: DOORS absolute number (e.g., "1.2.3")
- **Dynamic Attributes**: All non-empty attributes (Object Heading, Object Text, custom attributes, etc.)
- **links**: Array of link objects
  - **type**: Link module name
  - **direction**: "outgoing" or "incoming"
  - **targetModule** / **sourceModule**: Full path to linked module
  - **targetObjectId** / **sourceObjectId**: Identifier of linked object
- **children**: Array of child objects (recursive structure)

## How It Works

### Processing Flow

1. **Initialization**: Parse command-line arguments and validate module path
2. **Root Module Processing**:
   - Open the specified module
   - Iterate through all top-level objects
   - For each object:
     - Extract all non-empty attributes
     - Extract incoming and outgoing links
     - Recursively process child objects
     - Queue linked modules for processing
3. **Linked Module Processing**:
   - Process each unique linked module found (depth = 1)
   - Apply same extraction logic
   - Do NOT follow further links (depth limit reached)
4. **JSON Generation**: Build complete JSON structure
5. **File Output**: Write to output.json

### Key Algorithms

**Hierarchy Preservation**: Uses recursive traversal, processing only level 1 objects at the module level, then recursively handling children.

**Duplicate Prevention**: Uses a Skip list to track visited module paths, ensuring each module is exported only once.

**Link Following**: Collects linked module paths during object processing, queues them for batch processing after the current module completes.

## Supported Attribute Types

The script handles these DOORS attribute types:

- Text (attrText_)
- String (attrString_)
- Integer (attrInt_)
- Real numbers (attrReal_)
- Dates (attrDate_)
- Boolean (attrBoolean_)

**Note**: Binary attributes (OLE objects, images) are skipped.

## Error Handling

The script handles:

- Missing or invalid module paths
- Modules that cannot be opened
- File write permission errors
- Null or empty attribute values (skipped)

Error messages are printed to console with descriptive information.

## Performance Considerations

### Large Modules

For very large modules (1000+ objects):
- Processing may take several minutes
- Output JSON file may be several megabytes
- Monitor console for progress updates

### Many Linked Modules

The script limits depth to prevent exponential growth:
- MAX_DEPTH = 2 means original module + 1 level of links
- Adjust MAX_DEPTH if needed, but beware of performance impact

### Memory Usage

The entire JSON is built in memory before writing. For extremely large exports:
- Close other DOORS modules before running
- Ensure sufficient system memory
- Consider splitting large modules if necessary

## Troubleshooting

### "Cannot open module" Error

**Cause**: Module path is incorrect or module is locked
**Solution**:
- Verify the exact module path in DOORS
- Ensure module is not exclusively locked by another user
- Use read-only access

### "Cannot create output file" Error

**Cause**: Insufficient write permissions
**Solution**:
- Check directory permissions
- Ensure disk space is available
- Close output.json if open in another program

### Incomplete Export

**Cause**: Module depth exceeded or module not linked
**Solution**:
- Increase MAX_DEPTH if needed
- Verify links exist between modules
- Check console output for which modules were processed

### Invalid JSON Output

**Cause**: Special characters in attribute values
**Solution**:
- The script escapes common JSON characters
- If still invalid, report the specific attribute causing issues
- Use a JSON validator to identify the problem

## Integration with Python/SQLite

The output JSON is structured for easy database import:

### Python Example

```python
import json
import sqlite3

# Load JSON
with open('output.json', 'r') as f:
    data = json.load(f)

# Create database
conn = sqlite3.connect('doors_export.db')
cursor = conn.cursor()

# Create tables
cursor.execute('''
    CREATE TABLE objects (
        id TEXT PRIMARY KEY,
        module_path TEXT,
        level INTEGER,
        heading TEXT,
        text TEXT
    )
''')

cursor.execute('''
    CREATE TABLE links (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        source_id TEXT,
        target_id TEXT,
        link_type TEXT,
        FOREIGN KEY (source_id) REFERENCES objects(id),
        FOREIGN KEY (target_id) REFERENCES objects(id)
    )
''')

# Insert data (flatten hierarchy as needed)
# Implementation depends on your specific requirements

conn.commit()
conn.close()
```

## Customization

### Adding Custom Logic

You can modify the script to:

1. **Filter objects**: Add conditions in `processObjectRecursive` to skip certain objects
2. **Transform attributes**: Modify `extractAttributes` to rename or transform values
3. **Add metadata**: Include additional module or object information in the JSON
4. **Change output format**: Modify JSON structure generation in `exportToJSON`

### Example: Filter by Object Type

```dxl
// In processObjectRecursive function, add:
string objType = obj."Object Type" ""
if (objType == "Information") {
    return  // Skip information objects
}
```

## Limitations

- **Binary attributes**: Not exported (images, OLE objects)
- **Module baselines**: Exports current version only, not historical baselines
- **Access control**: Does not export DOORS permissions or access rights
- **DXL addins**: Does not execute or export DXL attribute logic
- **Depth limit**: Does not follow links beyond configured MAX_DEPTH

## License

This script is provided as-is for use with IBM DOORS 9.5.

## Support

For issues or questions:
1. Check the Troubleshooting section above
2. Review DOORS DXL Reference Manual for API details
3. Verify your DOORS version compatibility

## Version History

- **v1.0** (2026-01-06): Initial release
  - Full module export with hierarchy
  - Link extraction and cross-module following
  - JSON output with embedded relationships

## Contributing

Suggestions and improvements are welcome. When contributing:
1. Test thoroughly with DOORS 9.5
2. Document any new configuration options
3. Update this README with changes
4. Ensure backward compatibility where possible
