# API Contract: Text Rendering Service

**Module**: `src/services/renderEngine.js`  
**Responsibility**: Render text samples on Canvas with customizable typography  
**Dependencies**: Canvas API (built-in), opentype.js (for glyph measurement)  

---

## Exported Functions

### `renderText(canvas, textSample, font, adjustments)`

**Purpose**: Render a text sample on Canvas with optional glyph adjustments

**Input**:
```javascript
{
  canvas: HTMLCanvasElement,
  textSample: {
    text: string,
    fontSize: number,           // points (default: 16)
    lineHeight: number,         // points (default: fontSize * 1.5)
    letterSpacing: number,      // points (default: 0)
    textAlign: 'left' | 'center' | 'right',
    textColor: string,          // hex or rgb
    backgroundColor: string     // hex or rgb
  },
  font: {
    parsedFont: OpenTypeFont,
    metrics: { ascender, descender, unitsPerEm }
  },
  adjustments: {
    [glyphName]: {
      leftBearing?: number,
      advanceWidth?: number,
      rightBearing?: number
    }
  }
}
```

**Output**:
```javascript
{
  success: true,
  canvasWidth: number,
  canvasHeight: number,
  renderedText: string,
  warnings: string[]            // e.g., "Glyph not found for character 'ñ'"
}
```

**Behavior**:
- Clears canvas and fills with background color
- Measures text layout (line breaks, line widths)
- Positions each glyph accounting for line height, letter spacing, alignment
- Applies adjustments (custom metrics) to glyph positioning
- Renders glyphs as paths on Canvas
- Returns rendering metadata

**Performance**: < 500ms for samples up to 1,000 characters

**Edge Cases**:
- Missing glyphs: Skip glyph, leave space, add warning
- Text exceeds canvas: Clip or wrap (behavior TBD per design)
- Font size too small (<8pt): Clamp to 8pt, show warning
- Empty text: Render nothing, return success

---

### `measureTextLayout(text, font, fontSize, lineHeight, letterSpacing)`

**Purpose**: Calculate text layout without rendering (for analysis)

**Input**:
```javascript
{
  text: string,
  font: OpenTypeFont,
  fontSize: number,
  lineHeight: number,
  letterSpacing: number
}
```

**Output**:
```javascript
{
  lines: Array<{
    text: string,
    width: number,
    height: number,
    glyphs: Array<{
      character: string,
      glyphName: string,
      x: number,                // Absolute position
      y: number,
      advanceWidth: number,
      ascender: number,
      descender: number
    }>
  }>,
  totalWidth: number,
  totalHeight: number
}
```

**Behavior**:
- Breaks text into lines (by newline character)
- For each line, measures glyph positions and dimensions
- No rendering; pure measurement for layout analysis

---

### `renderGlyph(canvas, glyphName, font, size, position)`

**Purpose**: Render a single glyph on Canvas

**Input**:
```javascript
{
  canvas: HTMLCanvasElement,
  glyphName: string,
  font: OpenTypeFont,
  size: number,                // points
  position: { x: number, y: number }
}
```

**Output**:
```javascript
{
  success: boolean,
  width: number,
  height: number
}
```

**Behavior**:
- Renders glyph as path at specified size and position
- Used for confusion analysis (glyph comparison)

---

### `clearCanvas(canvas, backgroundColor)`

**Purpose**: Clear canvas and set background color

**Input**:
```javascript
{
  canvas: HTMLCanvasElement,
  backgroundColor: string       // hex or rgb (default: #ffffff)
}
```

**Output**: Void (modifies canvas in-place)

---

### `setCanvasSize(canvas, width, height, dpi)`

**Purpose**: Set canvas resolution with DPI awareness

**Input**:
```javascript
{
  canvas: HTMLCanvasElement,
  width: number,                // pixels
  height: number,               // pixels
  dpi: number                   // device pixel ratio (default: 1)
}
```

**Output**: Void (modifies canvas)

**Behavior**:
- Accounts for high-DPI displays (Retina, etc.)
- Adjusts internal resolution vs. display size for crisp rendering

---

## Error Handling

| Error | Cause | Recovery |
|-------|-------|----------|
| Glyph not found | Character not in font | Skip glyph, leave space, add warning |
| Canvas not ready | Canvas element undefined | Error state in UI |
| Invalid color | Bad hex/rgb format | Fallback to black or white |

---

## Usage Example

```javascript
// In React component
import { renderText, measureTextLayout } from '../services/renderEngine';

function TextPreview({ font, textSample, adjustments, theme }) {
  const canvasRef = useRef(null);
  
  useEffect(() => {
    if (!canvasRef.current || !font) return;
    
    const canvas = canvasRef.current;
    const bgColor = theme === 'dark' ? '#1a1a1a' : '#ffffff';
    const textColor = theme === 'dark' ? '#ffffff' : '#000000';
    
    renderText(canvas, {
      ...textSample,
      backgroundColor: bgColor,
      textColor: textColor
    }, font, adjustments);
  }, [font, textSample, adjustments, theme]);
  
  return <canvas ref={canvasRef} width={800} height={600} />;
}
```

---

**Version**: 1.0  
**Last Updated**: 2025-12-02
