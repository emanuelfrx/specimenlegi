# Quick Start Guide: Typography Analysis Tool

**Duration**: 5 minutes  
**Audience**: Developers setting up the project  
**Prerequisites**: Node.js 18+, git

---

## Installation

### 1. Clone and Install Dependencies

```bash
# Clone repository
git clone <repo-url>
cd typography-analysis

# Install Node dependencies
npm install
```

### 2. Start Development Server

```bash
# Terminal 1: Start Vite dev server (frontend)
npm run dev

# Terminal 2: Start Electron (desktop app)
npm run electron

# OR combined (if configured in package.json)
npm run start
```

**Expected Output**:
- Vite server running at `http://localhost:5173`
- Electron app window opens with tool UI

---

## File Structure Overview

```
typography-analysis/
├── public/                    # Static assets
├── src/
│   ├── main.jsx              # Electron entry
│   ├── App.jsx               # Root React component
│   ├── components/           # React components
│   │   ├── FontImport.jsx    # Import font dialog
│   │   ├── TextPreview.jsx   # Text rendering
│   │   ├── HeatmapViewer.jsx # Heatmap display
│   │   └── ... (others)
│   ├── services/             # Business logic
│   │   ├── fontLoader.js
│   │   ├── renderEngine.js
│   │   ├── heatmapGenerator.js
│   │   ├── rhythmAnalyzer.js
│   │   ├── confusionDetector.js
│   │   ├── adjustmentManager.js
│   │   └── exportManager.js
│   ├── utils/                # Helpers
│   │   ├── constants.js
│   │   ├── validation.js
│   │   └── logger.js
│   └── state/                # State management
│       └── appState.js
├── electron/                 # Electron main process
│   ├── main.js
│   ├── preload.js
│   └── handlers/
├── package.json
├── vite.config.js
└── electron-builder.config.js
```

---

## Key Workflows

### Workflow 1: Load a Font

1. Click **"Import Font"** button
2. Select an OTF, TTF, or UFO file from your system
3. App parses font and displays metadata (glyph count, format, etc.)
4. Font preview shows with default sample text

**Code**: `src/components/FontImport.jsx` → `src/services/fontLoader.js`

---

### Workflow 2: Render Text Sample

1. Enter custom text in the **"Text Sample"** input
2. Preview updates in real-time
3. Adjust font size, line height, or letter spacing with sliders
4. Toggle light/dark mode to see contrast variations

**Code**: `src/components/TextPreview.jsx` → `src/services/renderEngine.js`

---

### Workflow 3: Analyze Typography

1. Click **"Analyze"** button to run all analyses
   - Generates texture heatmap
   - Calculates rhythm metrics
   - Detects character confusion pairs
2. Results display in separate tabs
3. System auto-generates optimization recommendations

**Code**: 
- `src/services/heatmapGenerator.js`
- `src/services/rhythmAnalyzer.js`
- `src/services/confusionDetector.js`

---

### Workflow 4: Manual Adjustments

1. Click on a glyph in the preview or select from list
2. Adjust **left bearing**, **advance width**, or **right bearing**
3. Preview updates immediately
4. Undo/redo available via buttons or Ctrl+Z / Ctrl+Y

**Code**: `src/services/adjustmentManager.js` → `src/components/AdjustmentPanel.jsx`

---

### Workflow 5: Export Optimized Font

1. Choose export format (OTF or TTF)
2. Click **"Export Font"**
3. OS file dialog opens
4. Select save location and confirm
5. Success notification shows file path

**Code**: `src/services/exportManager.js` → `src/components/ExportDialog.jsx`

---

## Development Tips

### Adding a New Analysis Type

1. Create service: `src/services/myAnalysis.js`
   ```javascript
   export async function analyzeMyMetric(font, textSample) {
     // Return result object
     return { success: true, data: {...} };
   }
   ```

2. Create component: `src/components/MyAnalysisViewer.jsx`
   ```jsx
   export function MyAnalysisViewer({ results }) {
     return <div>...</div>;
   }
   ```

3. Integrate in `App.jsx`
   ```jsx
   const [myAnalysis, setMyAnalysis] = useState(null);
   // Call service and display component
   ```

### Using Web Workers for Heavy Computation

1. Create worker: `public/workers/myWorker.js`
   ```javascript
   self.onmessage = (e) => {
     const result = expensiveCalculation(e.data);
     self.postMessage(result);
   };
   ```

2. Use in service:
   ```javascript
   const worker = new Worker('/workers/myWorker.js');
   worker.postMessage(data);
   worker.onmessage = (e) => {
     resolve(e.data);
   };
   ```

### Debugging

**Browser DevTools** (in Electron):
- Press `Ctrl+Shift+I` (Windows/Linux) or `Cmd+Option+I` (macOS)
- Inspect components, check console for errors

**Electron Main Process**:
- Add `--debug` flag to `npm run electron`
- Use Node Inspector (`chrome://inspect`)

**Logging**:
- Use `src/utils/logger.js` for structured logs
- Logs available in browser console and Electron logs

---

## Common Tasks

### Run Linting
```bash
npm run lint
```

### Build for Distribution
```bash
npm run build
```

Outputs packaged Electron apps for Windows, macOS, Linux.

### Run Formatting
```bash
npm run format
```

Auto-formats code with Prettier.

---

## Testing Workflow (Manual)

1. **Import Test**: Load `public/fonts/Roboto.otf` (included)
2. **Render Test**: Type sample text, verify preview updates
3. **Analysis Test**: Run all analyses, check results display
4. **Adjustment Test**: Adjust a glyph, verify preview changes
5. **Export Test**: Export to OTF, verify file is valid
   - Open in FontLab, Glyphs, or RoboFont
   - Verify adjustments applied correctly

---

## Performance Targets

| Operation | Target | Check With |
|-----------|--------|-----------|
| Font import | < 2s | Browser dev tools (Network tab) |
| Preview render | < 500ms | Browser dev tools (Performance tab) |
| Heatmap generation | < 3s | Progress indicator |
| Rhythm analysis | < 5s | Progress indicator |
| Confusion analysis | < 5s | Progress indicator |

**Note**: Large fonts (>5,000 glyphs) may exceed targets. Use Web Workers and show progress indicators.

---

## Documentation Structure

```
specs/001-typography-analysis/
├── spec.md                     # User requirements
├── plan.md                     # Architecture & roadmap
├── research.md                 # Technology decisions
├── data-model.md               # Entity definitions
├── contracts/                  # API specifications
│   ├── font-loader.contract.md
│   ├── render-engine.contract.md
│   ├── analysis.contract.md
│   ├── adjustment.contract.md
│   └── export.contract.md
├── quickstart.md               # This file
└── checklists/
    └── requirements.md         # Spec validation
```

---

## Next Steps

1. **Phase 2**: Generate `tasks.md` with implementation task breakdown
2. **Development**: Implement components and services per task list
3. **Testing**: Manual testing per QA checklist (no automated tests per constitution)
4. **Deployment**: Build and package for distribution
5. **Monitoring**: Track performance and errors in production

---

## Getting Help

- **API Contracts**: See `contracts/*.md` for service interfaces
- **Data Model**: See `data-model.md` for entity structures
- **Architecture**: See `plan.md` for system design
- **Spec**: See `spec.md` for user requirements

---

**Last Updated**: 2025-12-02  
**Version**: 1.0
