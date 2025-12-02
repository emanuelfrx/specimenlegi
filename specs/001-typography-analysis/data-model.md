# Data Model: Typography Analysis & Optimization Tool

**Phase**: 1 (Design)  
**Date**: 2025-12-02  
**Status**: Complete  
**Input**: Specification, Constitution, Research findings

---

## Domain Entities

### 1. Font

**Responsibility**: Represents a loaded font file with metadata and glyph collection

```javascript
{
  // Immutable metadata
  id: string,                    // Unique session identifier
  name: string,                  // Font name from metadata
  format: 'otf' | 'ttf',        // Import format
  glyphCount: number,            // Total glyphs
  unitsPerEm: number,           // Font units (typically 1000 or 2048)
  ascender: number,             // Ascender height
  descender: number,            // Descender depth
  
  // Raw font object (from opentype.js)
  parsedFont: OpenTypeFont,      // Parsed font library object
  
  // Timestamps
  importedAt: Date,
  lastModifiedAt: Date,
  
  // Validation state
  isValid: boolean,
  validationErrors: string[]
}
```

**Relationships**:
- Owns collection of Glyphs (accessed via `font.parsedFont.glyphs`)
- Referenced by TextSample, Adjustment

**State Lifecycle**:
```
[Unloaded] → [Importing] → [Valid] → [Adjusted] → [Exporting] → [Exported]
```

---

### 2. Glyph

**Responsibility**: Individual character representation with adjustable metrics

```javascript
{
  // Identity
  glyphName: string,            // Unique identifier within font
  unicode: string,              // Unicode code point (if applicable)
  character: string,            // Rendered character
  
  // Original metrics (immutable, from opentype.js)
  originalMetrics: {
    advanceWidth: number,       // Total horizontal advance
    leftBearing: number,        // Space before glyph
    rightBearing: number,       // Space after glyph
    xMin: number, xMax: number, // Bounding box
    yMin: number, yMax: number
  },
  
  // Current metrics (affected by adjustments)
  currentMetrics: {
    advanceWidth: number,
    leftBearing: number,
    rightBearing: number
  },
  
  // Visual properties
  contours: Array,              // Path data (from opentype.js)
  
  // Adjustment tracking
  hasAdjustments: boolean,
  adjustments: Adjustment[]     // Linked adjustments
}
```

**Constraints**:
- `advanceWidth > 0` always
- `leftBearing + rightBearing + glyphWidth < advanceWidth` (to prevent overlaps)

**Relationships**:
- Belongs to Font
- Referenced by Adjustment

---

### 3. TextSample

**Responsibility**: User-provided text for rendering and analysis

```javascript
{
  // Identity
  id: string,
  
  // Content
  text: string,                 // User-entered or default text
  language: string,             // Language hint (e.g., 'en', 'pt')
  
  // Display settings
  fontSize: number,             // Points (default: 16)
  lineHeight: number,           // Points (default: fontSize * 1.5)
  letterSpacing: number,        // Points (default: 0)
  textAlign: 'left' | 'center' | 'right',
  
  // Rendering context
  fontId: string,               // Reference to Font
  backgroundColor: string,      // hex color
  textColor: string,            // hex color (default: #000000)
  
  // Timestamps
  createdAt: Date,
  lastUpdatedAt: Date
}
```

**Default Sample**:
```
"The quick brown fox jumps over the lazy dog. "
"Pack my box with five dozen liquor jugs. "
"ABCDEFGHIJKLMNOPQRSTUVWXYZ "
"abcdefghijklmnopqrstuvwxyz "
"0123456789"
```

**Relationships**:
- References Font (via fontId)
- Input to AnalysisResult generation

---

### 4. AnalysisResult

**Responsibility**: Output from rhythm, heatmap, or confusion analysis

```javascript
{
  // Identity
  id: string,
  type: 'heatmap' | 'rhythm' | 'confusion',
  
  // Input metadata
  textSampleId: string,
  fontId: string,
  analyzedAt: Date,
  
  // Heatmap-specific
  heatmapData: {
    type: 'heatmap',
    densityMap: number[],       // Density per line (0-1)
    visualization: Canvas,      // Rendered heatmap
    summary: {
      averageDensity: number,
      densityVariance: number,
      peakDensity: number
    }
  },
  
  // Rhythm-specific
  rhythmData: {
    type: 'rhythm',
    characterSpacing: {
      values: number[],         // Spacing per character pair
      mean: number,
      stdDev: number,
      min: number,
      max: number
    },
    lineSpacing: {
      values: number[],         // Spacing per line pair
      mean: number,
      stdDev: number
    },
    rhythmScore: number,        // 0-100, higher = more consistent
    issues: {
      description: string,
      affectedRanges: Array<{ start: number, end: number }>
    }
  },
  
  // Confusion-specific
  confusionData: {
    type: 'confusion',
    confusablePairs: Array<{
      pair: [string, string],   // e.g., ['0', 'O']
      similarity: number,       // 0-1
      recommendation: string,
      severity: 'low' | 'medium' | 'high'
    }>,
    summary: {
      totalPairs: number,
      criticalIssues: number    // High severity pairs
    }
  }
}
```

**Relationships**:
- References TextSample and Font
- Input to Recommendation generation

---

### 5. Adjustment

**Responsibility**: A user-made or recommended modification to glyph metrics

```javascript
{
  // Identity
  id: string,
  glyphName: string,
  
  // Source
  source: 'manual' | 'recommendation',
  recommendationId?: string,    // If from Recommendation
  
  // Adjustment data
  changes: {
    leftBearing?: number,       // Delta from original
    advanceWidth?: number,
    rightBearing?: number
  },
  
  // Metadata
  description: string,          // Why this adjustment
  appliedAt: Date,
  undoable: boolean,
  
  // Validation
  isValid: boolean,             // Passes constraint checks
  validationErrors: string[]
}
```

**Constraints**:
- Changes must result in valid metrics (no overlaps)
- Adjustments are cumulative (new adjustment builds on previous state)

**Relationships**:
- Targets Glyph
- May originate from Recommendation

---

### 6. Recommendation

**Responsibility**: Automated suggestion for glyph adjustments

```javascript
{
  // Identity
  id: string,
  
  // Source
  analysisId: string,           // AnalysisResult that generated this
  
  // Recommendation content
  glyphName: string,
  adjustmentType: 'spacing' | 'width' | 'kerning' | 'weight',
  suggestedValue: number,
  reason: string,               // Why this adjustment is recommended
  
  // Impact assessment
  priority: 1 | 2 | 3,          // 1 = critical, 3 = optional
  affectsGlyphs: string[],      // Other glyphs impacted by this change
  
  // User interaction
  status: 'pending' | 'accepted' | 'rejected',
  appliedAt?: Date
}
```

**Relationships**:
- Generated from AnalysisResult
- Targets Glyph
- May create Adjustment when accepted

---

### 7. ExportJob

**Responsibility**: Request to save modified font to disk

```javascript
{
  // Identity
  id: string,
  
  // Export parameters
  fontId: string,
  format: 'otf' | 'ttf' | 'ufo',
  
  // Output
  filename: string,             // e.g., "MyFont-Optimized.otf"
  outputPath?: string,          // System path (set after save dialog)
  
  // Status
  status: 'pending' | 'processing' | 'complete' | 'failed',
  progress: number,             // 0-100
  error?: string,
  
  // Result
  exportedAt?: Date,
  fileSize?: number,
  checksum?: string             // For verification
}
```

**Relationships**:
- Exports Font with applied Adjustments

---

## State Management Design

### Application State Structure

```javascript
{
  // Current font
  font: Font | null,
  
  // Text samples (user can create multiple)
  textSamples: TextSample[],
  activeSampleId: string | null,
  
  // Analysis results (cached)
  analysisResults: AnalysisResult[],
  
  // Adjustments (accumulated during session)
  adjustments: Adjustment[],
  adjustmentHistory: Adjustment[],     // For undo/redo
  adjustmentHistoryIndex: number,
  
  // Recommendations
  recommendations: Recommendation[],
  
  // UI state
  ui: {
    currentView: 'import' | 'preview' | 'analysis' | 'adjustments' | 'export',
    theme: 'light' | 'dark',
    isAnalyzing: boolean,
    progress: number,
    errorMessage: string | null,
    notifications: Array<{ id, type, message, duration }>
  },
  
  // Export state
  exportJob: ExportJob | null
}
```

### State Transitions

**Import Font**:
```
Initial → FontImporting → FontLoaded → Ready for Analysis
```

**Analyze Text**:
```
TextEntered → AnalysisRunning → AnalysisComplete → RecommendationsReady
```

**Adjust Glyph**:
```
GlyphSelected → AdjustmentInput → AdjustmentValidated → PreviewUpdated
```

**Export**:
```
ExportRequested → FormatSelected → FileSaveDialog → ExportProcessing → Exported
```

---

## Validation Rules

### Font Validation
- Must be valid OTF/TTF format
- Must contain at least 1 glyph
- Must be < 50MB (file size check)

### TextSample Validation
- Must be non-empty string
- fontSize >= 8, <= 144 points
- lineHeight >= fontSize, <= 3 * fontSize

### Adjustment Validation
```javascript
const validateAdjustment = (glyph, adjustment) => {
  const newAdvance = glyph.advanceWidth + (adjustment.advanceWidth || 0);
  const newLeft = glyph.leftBearing + (adjustment.leftBearing || 0);
  const newRight = glyph.rightBearing + (adjustment.rightBearing || 0);
  
  // Constraints
  if (newAdvance <= 0) throw new Error('Advance width must be positive');
  if (newLeft + glyphWidth + newRight > newAdvance) {
    throw new Error('Adjustment creates overlap');
  }
  
  return true;
};
```

---

## Relationships Diagram

```
Font
  ├─→ Glyphs (collection)
  ├─→ TextSamples (references)
  └─→ AnalysisResults (references)

TextSample
  ├─→ Font (reference)
  └─→ AnalysisResults (input)

AnalysisResult
  ├─→ TextSample (reference)
  ├─→ Recommendations (generates)
  └─→ Adjustments (input for suggestions)

Recommendation
  ├─→ AnalysisResult (source)
  ├─→ Glyph (targets)
  └─→ Adjustment (creates when accepted)

Adjustment
  ├─→ Glyph (modifies)
  ├─→ Recommendation (may originate from)
  └─→ Adjustments (history chain)

ExportJob
  └─→ Font (exports with adjustments applied)
```

---

## Notes on Implementation

- **Immutability**: Original font data (from opentype.js) remains immutable; adjustments create new derived state
- **Undo/Redo**: Adjustment history stored as array; moving `adjustmentHistoryIndex` forward/backward
- **Performance**: Large fonts (>5,000 glyphs) analyzed via Web Worker; main thread remains responsive
- **Session-only**: All data ephemeral; no database persistence per constitution
- **Type Safety**: Props validation via PropTypes in React components; custom validation functions for domain rules

---

**Status**: Data Model Complete  
**Date**: 2025-12-02  
**Next**: API Contracts
