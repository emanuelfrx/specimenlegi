# API Contract: Font Export

**Module**: `src/services/exportManager.js`  
**Responsibility**: Export modified font to disk in OTF/TTF format  
**Dependencies**: opentype.js (for font serialization), Electron IPC (for file dialogs)  

---

## Exported Functions

### `prepareExport(font, adjustments, format = 'otf')`

**Purpose**: Prepare modified font for export (apply adjustments)

**Input**:
```javascript
{
  font: OpenTypeFont,
  adjustments: Adjustment[],       // All adjustments to apply
  format: 'otf' | 'ttf' | 'ufo'   // Target format
}
```

**Output**:
```javascript
{
  success: boolean,
  font: OpenTypeFont,              // Modified copy
  format: string,
  sizeEstimate: number,            // Bytes
  errors: string[]
}
```

**Behavior**:
- Creates copy of font
- Applies all adjustments to glyph metrics
- Validates result (no overlaps, reasonable metrics)
- Returns modified font ready for serialization

---

### `serializeFont(font, format = 'otf')`

**Purpose**: Convert font object to binary data

**Input**:
```javascript
{
  font: OpenTypeFont,
  format: 'otf' | 'ttf' | 'ufo'
}
```

**Output**:
```javascript
{
  success: boolean,
  buffer: ArrayBuffer,             // Binary font data
  size: number,                    // Bytes
  format: string,
  errors: string[]
}
```

**Behavior**:
- Uses opentype.js `toArrayBuffer()` for OTF/TTF
- UFO format: throws "Not yet implemented" error (defer to v1.1)
- Returns binary data ready for file save

**Error Cases**:
```
format === 'ufo' → Error: "UFO export not yet implemented; planned for v1.1"
font is null → Error: "Font object is null"
serialization fails → Error: "Font serialization failed: [opentype.js error]"
```

---

### `saveFont(buffer, filename, directory = null)`

**Purpose**: Write font binary to disk

**Input**:
```javascript
{
  buffer: ArrayBuffer,
  filename: string,                // e.g., "MyFont-Optimized.otf"
  directory: string | null         // Optional starting directory
}
```

**Output** (success):
```javascript
{
  success: true,
  filePath: string,                // Absolute path to saved file
  fileSize: number,
  format: string,
  savedAt: Date
}
```

**Output** (user canceled):
```javascript
{
  success: false,
  canceled: true,
  reason: "User canceled save dialog"
}
```

**Output** (error):
```javascript
{
  success: false,
  error: string,                   // Error message
  code: 'PERMISSION_DENIED' | 'DISK_FULL' | 'WRITE_ERROR'
}
```

**Behavior**:
- Shows OS file save dialog
- Default filename: "{fontName}-Optimized.{ext}"
- Filters by format (OTF, TTF)
- Writes buffer to selected path
- Returns saved file metadata

**Electron Integration**:
```javascript
// Uses Electron IPC bridge
const result = await window.electronAPI.saveFontFile(buffer, filename);
```

---

### `exportFont(font, adjustments, format = 'otf', filename = null)`

**Purpose**: Complete export workflow (prepare → serialize → save)

**Input**:
```javascript
{
  font: OpenTypeFont,
  adjustments: Adjustment[],
  format: 'otf' | 'ttf',
  filename: string | null          // e.g., "MyFont-Optimized"
}
```

**Output**:
```javascript
{
  success: boolean,
  filePath?: string,
  fileSize?: number,
  error?: string,
  canceled?: boolean,
  duration: number                 // Milliseconds
}
```

**Behavior**:
- Calls `prepareExport()` to apply adjustments
- Calls `serializeFont()` to get binary data
- Calls `saveFont()` to write to disk
- Returns complete result with status

---

### `validateExportFormat(format, font)`

**Purpose**: Check if format is supported for this font

**Input**:
```javascript
{
  format: 'otf' | 'ttf' | 'ufo',
  font: OpenTypeFont
}
```

**Output**:
```javascript
{
  isSupported: boolean,
  message: string,                 // User-friendly explanation
  alternatives: string[]           // Other supported formats
}
```

---

### `getExportFilename(font, format = 'otf', suffix = 'Optimized')`

**Purpose**: Generate a sensible export filename

**Input**:
```javascript
{
  font: { name: string },
  format: 'otf' | 'ttf',
  suffix: string                   // e.g., "Optimized", "Adjusted"
}
```

**Output**:
```javascript
string                             // e.g., "Roboto-Optimized.otf"
```

**Behavior**:
- Cleans font name (removes special chars)
- Removes existing extension
- Appends suffix
- Adds format extension
- Examples:
  - "Roboto" → "Roboto-Optimized.otf"
  - "MyFont-Regular" → "MyFont-Regular-Optimized.otf"

---

## Electron File Handler

**Module**: `electron/handlers/fontHandler.js`

### IPC: `save-font-file`

**Main process listener**:
```javascript
ipcMain.handle('save-font-file', async (event, buffer, defaultName) => {
  const result = await dialog.showSaveDialog(mainWindow, {
    defaultPath: defaultName,
    filters: [
      { name: 'OpenType Font', extensions: ['otf'] },
      { name: 'TrueType Font', extensions: ['ttf'] },
      { name: 'All Files', extensions: ['*'] }
    ]
  });
  
  if (result.canceled) {
    return { canceled: true };
  }
  
  try {
    fs.writeFileSync(result.filePath, Buffer.from(buffer));
    return {
      filePath: result.filePath,
      size: buffer.byteLength
    };
  } catch (error) {
    return { error: error.message };
  }
});
```

**React usage**:
```javascript
async function handleExport() {
  const result = await window.electronAPI.saveFontFile(
    fontBuffer,
    'MyFont-Optimized.otf'
  );
  
  if (result.error) {
    showError(result.error);
  } else if (result.canceled) {
    // User canceled
  } else {
    showSuccess(`Font saved to ${result.filePath}`);
  }
}
```

---

## Error Handling

| Error | Cause | Recovery |
|-------|-------|----------|
| UFO export | Unsupported format | Show message: "UFO export planned for v1.1" |
| Serialization failed | Font corruption | Log error, suggest reimporting font |
| Write permission | Disk permission denied | Show message: "Permission denied" + retry button |
| Disk full | Out of space | Show message: "Insufficient disk space" |
| Invalid filename | Path with invalid chars | Auto-sanitize or ask user |

---

## Usage Example

```javascript
import { exportFont, getExportFilename } from '../services/exportManager';

async function handleExportClick() {
  setIsExporting(true);
  showProgress("Preparing font for export...");
  
  try {
    const suggestedFilename = getExportFilename(
      font,
      exportFormat,
      'Optimized'
    );
    
    const result = await exportFont(
      font,
      state.adjustments,
      exportFormat,
      suggestedFilename
    );
    
    if (result.success) {
      showSuccess(
        `Font successfully exported to ${result.filePath}`
      );
      
      // Log for observability
      logger.info('Font exported', {
        filename: result.filePath,
        size: result.fileSize,
        format: exportFormat,
        adjustmentsCount: state.adjustments.length,
        duration: result.duration
      });
    } else if (result.canceled) {
      showNotification("Export canceled by user");
    } else {
      showError(`Export failed: ${result.error}`);
    }
  } finally {
    setIsExporting(false);
  }
}
```

---

**Version**: 1.0  
**Last Updated**: 2025-12-02
