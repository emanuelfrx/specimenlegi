# API Contract: Adjustment Management

**Module**: `src/services/adjustmentManager.js`  
**Responsibility**: Apply, validate, and track glyph metric adjustments  

---

## Exported Functions

### `createAdjustment(glyphName, changes, source = 'manual', description)`

**Purpose**: Create a new adjustment record

**Input**:
```javascript
{
  glyphName: string,
  changes: {
    leftBearing?: number,       // Delta from original
    advanceWidth?: number,
    rightBearing?: number
  },
  source: 'manual' | 'recommendation',
  description: string            // Why this change
}
```

**Output**:
```javascript
{
  id: string,                   // UUID
  glyphName: string,
  source: string,
  changes: object,
  description: string,
  appliedAt: Date,
  isValid: boolean,
  validationErrors: string[]
}
```

**Behavior**:
- Generates unique ID
- Validates adjustment doesn't create overlaps
- Returns validation errors if constraints violated
- Does NOT modify font yet (just creates record)

---

### `applyAdjustment(font, adjustment, currentAdjustments = [])`

**Purpose**: Apply an adjustment to the font's state

**Input**:
```javascript
{
  font: OpenTypeFont,
  adjustment: { glyphName, changes },
  currentAdjustments: Adjustment[]  // Existing adjustments to apply first
}
```

**Output**:
```javascript
{
  success: boolean,
  modifiedFont: OpenTypeFont,   // Copy with adjustment applied
  appliedAdjustment: Adjustment,
  warnings: string[]
}
```

**Behavior**:
- Creates copy of font (immutable pattern)
- Applies all current adjustments first (to get baseline)
- Applies new adjustment
- Validates result has no overlaps
- Returns modified font copy

**Example**:
```javascript
const glyph = font.glyphs.get('A');
// Original metrics
glyph.advanceWidth = 600;
glyph.leftBearing = 50;
glyph.rightBearing = 50;

// Apply adjustment: { advanceWidth: -30 }
// Result: advanceWidth = 570
```

---

### `validateAdjustment(font, glyphName, changes, appliedAdjustments = [])`

**Purpose**: Check if adjustment violates constraints

**Input**:
```javascript
{
  font: OpenTypeFont,
  glyphName: string,
  changes: { leftBearing?, advanceWidth?, rightBearing? },
  appliedAdjustments: Adjustment[]
}
```

**Output**:
```javascript
{
  isValid: boolean,
  errors: string[],
  warnings: string[]
}
```

**Validation Rules**:
```javascript
// Rule 1: Advance width must be positive
newAdvanceWidth > 0

// Rule 2: No overlaps
newLeftBearing + glyphWidth + newRightBearing <= newAdvanceWidth

// Rule 3: Metrics must be reasonable
newLeftBearing >= -glyphWidth * 0.1  // Allow small overhang
newRightBearing >= -glyphWidth * 0.1

// Rule 4: Reasonable bounds (prevent nonsensical values)
newAdvanceWidth < 10000
newLeftBearing < 5000
newRightBearing < 5000
```

**Error Messages**:
- "Advance width must be > 0"
- "Adjustment creates glyph overlap (right side protrudes)"
- "Adjustment creates glyph overlap (left side protrudes)"
- "Left bearing must be within reasonable bounds"

---

### `undoAdjustment(adjustmentHistory, currentIndex)`

**Purpose**: Revert to previous adjustment state

**Input**:
```javascript
{
  adjustmentHistory: Adjustment[],
  currentIndex: number            // Current position in history
}
```

**Output**:
```javascript
{
  adjustment: Adjustment | null,  // Previous adjustment (or null if at start)
  newIndex: number,
  adjustments: Adjustment[]       // Updated list up to newIndex
}
```

---

### `redoAdjustment(adjustmentHistory, currentIndex)`

**Purpose**: Reapply an adjustment after undo

**Input**:
```javascript
{
  adjustmentHistory: Adjustment[],
  currentIndex: number
}
```

**Output**:
```javascript
{
  adjustment: Adjustment | null,  // Next adjustment (or null if at end)
  newIndex: number,
  adjustments: Adjustment[]       // Updated list up to newIndex
}
```

---

### `getAdjustmentsForGlyph(adjustments, glyphName)`

**Purpose**: Find all adjustments affecting a specific glyph

**Input**:
```javascript
{
  adjustments: Adjustment[],
  glyphName: string
}
```

**Output**:
```javascript
Adjustment[]                      // Array of adjustments for glyph
```

---

### `getCurrentMetrics(originalMetrics, adjustments, glyphName)`

**Purpose**: Calculate current metrics after all adjustments

**Input**:
```javascript
{
  originalMetrics: { advanceWidth, leftBearing, rightBearing },
  adjustments: Adjustment[],
  glyphName: string
}
```

**Output**:
```javascript
{
  advanceWidth: number,
  leftBearing: number,
  rightBearing: number,
  appliedAdjustmentCount: number
}
```

**Behavior**:
- Sum all adjustments for the glyph
- Return final metrics as original + deltas

---

### `batchValidateAdjustments(font, adjustments)`

**Purpose**: Validate multiple adjustments at once

**Input**:
```javascript
{
  font: OpenTypeFont,
  adjustments: Adjustment[]
}
```

**Output**:
```javascript
{
  isValid: boolean,
  results: Array<{
    adjustmentId: string,
    isValid: boolean,
    errors: string[]
  }>
}
```

---

## State Management Integration

### Adjustment History Pattern

```javascript
// AppState
{
  adjustments: Adjustment[],              // Current applied adjustments
  adjustmentHistory: Adjustment[],        // All changes (for undo/redo)
  adjustmentHistoryIndex: number,         // Current position in history
}

// Actions
function applyNewAdjustment(adjustment) {
  // 1. Validate
  const validation = validateAdjustment(font, adjustment.glyphName, adjustment.changes, adjustments);
  if (!validation.isValid) throw new Error(validation.errors[0]);
  
  // 2. Add to history (remove any "future" history if we're rewinding)
  const newHistory = adjustmentHistory.slice(0, adjustmentHistoryIndex + 1);
  newHistory.push(adjustment);
  
  // 3. Update state
  return {
    adjustments: [...adjustments, adjustment],
    adjustmentHistory: newHistory,
    adjustmentHistoryIndex: newHistory.length - 1
  };
}

function undo() {
  const newIndex = Math.max(0, adjustmentHistoryIndex - 1);
  return {
    adjustments: adjustmentHistory.slice(0, newIndex + 1),
    adjustmentHistoryIndex: newIndex
  };
}

function redo() {
  const newIndex = Math.min(adjustmentHistory.length - 1, adjustmentHistoryIndex + 1);
  return {
    adjustments: adjustmentHistory.slice(0, newIndex + 1),
    adjustmentHistoryIndex: newIndex
  };
}
```

---

## Error Handling

| Error | Cause | Recovery |
|-------|-------|----------|
| Invalid glyph name | Glyph doesn't exist in font | Error state, disable adjustment |
| Overlap violation | Adjustment creates overlap | Show specific error message, suggest correction |
| History empty | Undo at start | Disable undo button |
| No future | Redo with no future | Disable redo button |

---

## Usage Example

```javascript
import { 
  createAdjustment, 
  applyAdjustment, 
  validateAdjustment 
} from '../services/adjustmentManager';

// User adjusts spacing for glyph 'A'
function handleGlyphAdjustment(glyphName, newLeftBearing, newAdvanceWidth) {
  // 1. Create adjustment record
  const adjustment = createAdjustment(glyphName, {
    leftBearing: newLeftBearing - original.leftBearing,
    advanceWidth: newAdvanceWidth - original.advanceWidth
  }, 'manual', 'User adjusted spacing');
  
  // 2. Validate
  const validation = validateAdjustment(
    font, 
    glyphName, 
    adjustment.changes, 
    state.adjustments
  );
  
  if (!validation.isValid) {
    showError(validation.errors[0]);
    return;
  }
  
  // 3. Apply
  const result = applyAdjustment(font, adjustment, state.adjustments);
  
  // 4. Update state
  dispatch({
    type: 'ADJUSTMENT_APPLIED',
    payload: {
      adjustment,
      font: result.modifiedFont
    }
  });
  
  // 5. Re-render preview with new font
  renderPreview();
}
```

---

**Version**: 1.0  
**Last Updated**: 2025-12-02
