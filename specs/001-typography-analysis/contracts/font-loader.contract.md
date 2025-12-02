# API Contract: Font Loader Service

**Module**: `src/services/fontLoader.js`  
**Responsibility**: Load, parse, and validate font files  
**Dependencies**: opentype.js  

---

## Exported Functions

### `loadFont(arrayBuffer, filename)`

**Purpose**: Parse a font file from binary data

**Input**:
```javascript
{
  arrayBuffer: ArrayBuffer,     // Binary font file data
  filename: string              // Original filename (for format detection)
}
```

**Output** (success):
```javascript
{
  id: string,                   // Session-unique ID
  name: string,                 // Font name from metadata
  format: 'otf' | 'ttf',       // Detected format
  glyphCount: number,
  unitsPerEm: number,
  ascender: number,
  descender: number,
  parsedFont: OpenTypeFont,     // opentype.js Font object
  importedAt: Date,
  isValid: true,
  validationErrors: []
}
```

**Output** (error):
```javascript
throws Error {
  message: string,              // User-readable error message
  code: 'INVALID_FONT' | 'UNSUPPORTED_FORMAT' | 'CORRUPTED_FILE'
}
```

**Behavior**:
- Validates font format via magic bytes or extension
- Parses font using opentype.js
- Extracts metadata (name, glyph count, metrics)
- Validates font integrity (no corrupted glyphs)
- Returns font object ready for rendering/analysis

**Performance**: < 2 seconds for fonts up to 5,000 glyphs

---

### `validateFont(font)`

**Purpose**: Verify font is suitable for analysis

**Input**:
```javascript
{
  parsedFont: OpenTypeFont
}
```

**Output**:
```javascript
{
  isValid: boolean,
  errors: string[],             // List of validation issues
  warnings: string[]            // Non-critical issues
}
```

**Validation Rules**:
- Font has at least 1 glyph
- All glyph contours are valid
- Font metrics are sensible (ascender > 0, descender < 0)

---

### `getGlyphMetrics(font, glyphName)`

**Purpose**: Extract metrics for a specific glyph

**Input**:
```javascript
{
  parsedFont: OpenTypeFont,
  glyphName: string             // e.g., 'A', 'space', '.notdef'
}
```

**Output**:
```javascript
{
  glyphName: string,
  unicode: string | null,
  character: string | null,
  advanceWidth: number,
  leftBearing: number,
  rightBearing: number,
  xMin: number,
  xMax: number,
  yMin: number,
  yMax: number
}
```

**Behavior**:
- Returns null if glyph not found
- Calculates bearings from bounding box and advance width

---

## Error Handling

| Error | Code | Cause | Recovery |
|-------|------|-------|----------|
| Invalid font file | INVALID_FONT | Corrupted binary or wrong format | User selects different file |
| Unsupported format | UNSUPPORTED_FORMAT | Format is not OTF/TTF | Display list of supported formats |
| File too large | FILE_TOO_LARGE | > 50MB | Show file size limit |
| Parse failed | PARSE_ERROR | opentype.js couldn't parse | Log error, ask user to validate font in FontLab |

---

## Usage Example

```javascript
// In React component
import { loadFont } from '../services/fontLoader';

async function handleFontImport(file) {
  try {
    const buffer = await file.arrayBuffer();
    const font = await loadFont(buffer, file.name);
    
    dispatch({
      type: 'FONT_LOADED',
      payload: font
    });
  } catch (error) {
    showErrorNotification(error.message);
  }
}
```

---

**Version**: 1.0  
**Last Updated**: 2025-12-02
