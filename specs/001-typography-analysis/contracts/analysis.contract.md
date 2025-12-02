# API Contract: Analysis Services

**Modules**: 
  - `src/services/heatmapGenerator.js` - Texture visualization
  - `src/services/rhythmAnalyzer.js` - Spacing analysis  
  - `src/services/confusionDetector.js` - Character confusion analysis

**Responsibility**: Analyze typography for rhythm, texture, and confusion issues  

---

## Heatmap Generator

### `generateHeatmap(canvas, textSample, font, renderLayout)`

**Purpose**: Create a visual density heatmap showing text texture

**Input**:
```javascript
{
  canvas: HTMLCanvasElement,
  textSample: {
    text: string,
    fontSize: number,
    lineHeight: number,
    letterSpacing: number
  },
  font: OpenTypeFont,
  renderLayout: {                // From measureTextLayout()
    lines: Array,
    totalWidth: number,
    totalHeight: number
  }
}
```

**Output**:
```javascript
{
  success: true,
  densityMap: number[],           // Density per line (0-1)
  visualization: canvas,
  summary: {
    averageDensity: number,       // 0-1
    densityVariance: number,      // 0-1
    peakDensity: number,          // Maximum line density
    peakDensityLine: number       // Which line has peak
  }
}
```

**Behavior**:
- Renders text on temporary canvas
- Extracts pixel luminance data
- Computes line-by-line density (inverse of luminance)
- Renders heatmap as grayscale visualization (dark = high density)
- Lighter areas indicate better rhythm (more whitespace)

**Performance**: < 3 seconds for typical samples

---

## Rhythm Analyzer

### `analyzeRhythm(font, textSample, renderLayout)`

**Purpose**: Quantify spacing consistency in character and line rhythm

**Input**:
```javascript
{
  font: OpenTypeFont,
  textSample: {
    text: string,
    fontSize: number,
    lineHeight: number,
    letterSpacing: number
  },
  renderLayout: {                 // From measureTextLayout()
    lines: Array<{
      glyphs: Array<{ advanceWidth, x, y }>
    }>
  }
}
```

**Output**:
```javascript
{
  success: true,
  characterSpacing: {
    values: number[],             // Advance width per glyph pair
    mean: number,
    stdDev: number,
    min: number,
    max: number,
    variance: number              // stdDev / mean (normalized)
  },
  lineSpacing: {
    values: number[],             // Vertical distance per line pair
    mean: number,
    stdDev: number,
    variance: number
  },
  rhythmScore: number,            // 0-100, higher = more consistent
  issues: [
    {
      description: string,
      severity: 'low' | 'medium' | 'high',
      affectedGlyphs: string[],
      suggestedFix: string
    }
  ],
  warnings: string[]
}
```

**Rhythm Score Calculation**:
```javascript
score = 100 * (1 - avgVariance)
  where avgVariance = (charVariance + lineVariance) / 2
```
- 90-100: Excellent rhythm
- 70-89: Good, minor issues
- 50-69: Moderate, significant issues
- <50: Poor rhythm, major adjustments needed

**Performance**: < 5 seconds for typical samples; may use Web Worker for large fonts

---

### `analyzeRhythmAsync(font, textSample, renderLayout)`

**Purpose**: Non-blocking version using Web Worker

**Input**: Same as `analyzeRhythm()`

**Output**: Promise that resolves to same output as `analyzeRhythm()`

**Behavior**:
- Offloads computation to Web Worker
- Main thread remains responsive
- UI can show progress indicator

---

## Confusion Detector

### `detectConfusion(font, language = 'en')`

**Purpose**: Identify visually confusable character pairs

**Input**:
```javascript
{
  font: OpenTypeFont,
  language: string               // 'en', 'pt', etc. (default: 'en')
}
```

**Output**:
```javascript
{
  success: true,
  confusablePairs: [
    {
      pair: [char1, char2],      // e.g., ['0', 'O']
      similarity: number,        // 0-1 (>0.7 = confusing)
      severity: 'low' | 'medium' | 'high',
      recommendation: string,
      glyphNames: [glyphName1, glyphName2]
    }
  ],
  summary: {
    totalPairs: number,
    criticalIssues: number,      // High severity
    addressedPairs: number       // Approved adjustments
  }
}
```

**Predefined Confusion Pairs** (en):
```
0/O, 1/l, 1/I, 2/Z, 5/S, 6/G, 8/B, 
il/1l, rn/m, cl/d, o/0, O/0, etc.
```

**Similarity Scoring**:
- Render both glyphs at same size
- Compute pixel-level Jaccard similarity
- Score > 0.7 = confusing (recommend adjustment)

**Severity Levels**:
- **High**: Similarity > 0.85 (critical accessibility issue)
- **Medium**: 0.75-0.85 (noticeable confusion potential)
- **Low**: 0.65-0.75 (minor, context-dependent)

**Performance**: < 5 seconds for full font

---

### `detectConfusionAsync(font, language = 'en')`

**Purpose**: Non-blocking version using Web Worker

**Input**: Same as `detectConfusion()`

**Output**: Promise resolving to same structure

---

## Common Error Handling

| Error | Cause | Recovery |
|-------|-------|----------|
| Invalid text sample | Empty text | Skip analysis, show message |
| Missing font | Font not loaded | Error state in UI |
| Canvas error | Canvas context unavailable | Fallback to non-visual analysis |
| Analysis timeout | Computation exceeds time limit | Allow user to cancel, show partial results |

---

## Usage Examples

```javascript
// Heatmap generation
import { generateHeatmap } from '../services/heatmapGenerator';

async function showHeatmap() {
  const layout = measureTextLayout(text, font, fontSize, lineHeight, letterSpacing);
  const result = await generateHeatmap(canvas, textSample, font, layout);
  
  displayHeatmapChart(result.summary);
}

// Rhythm analysis (async for large fonts)
import { analyzeRhythmAsync } from '../services/rhythmAnalyzer';

async function analyzeRhythm() {
  setIsAnalyzing(true);
  
  const result = await analyzeRhythmAsync(font, textSample, layout);
  
  displayRhythmReport(result);
  generateRecommendationsFromIssues(result.issues);
  
  setIsAnalyzing(false);
}

// Confusion analysis
import { detectConfusionAsync } from '../services/confusionDetector';

async function checkConfusion() {
  const result = await detectConfusionAsync(font, 'en');
  
  if (result.summary.criticalIssues > 0) {
    showWarning(`Found ${result.summary.criticalIssues} critical confusion issues`);
  }
  
  displayConfusionChart(result.confusablePairs);
}
```

---

**Version**: 1.0  
**Last Updated**: 2025-12-02
