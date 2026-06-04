# Developer Guide - ITR-1 Chat UI

## Quick Start for Developers

### Setup
```bash
cd itr-chat
npm install
npm run dev
```

### File Structure Overview
```
src/
├── components/       # React components
├── utils/           # Utility functions
├── App.jsx          # Main app coordinator
└── main.jsx         # Entry point
```

## Key Development Concepts

### 1. Schema-to-Questions Pipeline

**Flow**: Schema → Parser → Questions Array → Chat UI

```javascript
// schemaParser.js
export function parseSchema(schema) {
  // Extracts definitions from JSON Schema
  // Creates flat question list with:
  // - id: JSON path (e.g., 'ITR.ITR1.PersonalInfo.PAN')
  // - label: Human-readable name
  // - type: Data type (string, integer, etc.)
  // - inputType: UI control (text, number, select, date)
  // - validations: pattern, min, max, enum
  return questions;
}
```

### 2. Form Data Structure

FormData mirrors the exact ITR-1 schema structure:

```javascript
{
  ITR: {
    ITR1: {
      PersonalInfo: {
        PAN: "ABCDE1234F",
        DOB: "1990-01-01",
        ...
      },
      ITR1_IncomeDeductions: {
        GrossSalary: 600000,
        ...
      },
      ...
    }
  }
}
```

### 3. Validation Strategy

**Two-Level Validation**:

1. **Per-Field** (Real-time):
```javascript
const validation = validateField(question, userAnswer);
if (!validation.valid) {
  showError(validation.error);
}
```

2. **Full Schema** (Before Download):
```javascript
const { valid, errors } = validateFullSchema(schema, formData);
if (valid) {
  downloadJSON(formData);
}
```

### 4. State Management

```javascript
// App.jsx state
const [schema, setSchema] = useState(null);        // JSON Schema
const [questions, setQuestions] = useState([]);     // Question list
const [formData, setFormData] = useState({});       // User answers
const [validationErrors, setValidationErrors] = useState([]);
```

**State Flow**:
1. Load schema → Parse to questions
2. User answers → Update formData
3. FormData changes → Auto-save to localStorage
4. FormData changes → Re-render charts

### 5. Deep Path Manipulation

For nested object updates:

```javascript
import { deepGet, deepSet } from './utils/deepPath';

// Get value
const pan = deepGet(formData, 'ITR.ITR1.PersonalInfo.PAN');

// Set value (creates intermediate objects)
deepSet(formData, 'ITR.ITR1.PersonalInfo.PAN', 'ABCDE1234F');
```

## Component Architecture

### ChatBot Component

**Responsibilities**:
- Orchestrate question flow
- Validate each answer
- Update formData
- Handle edit mode

**Key Methods**:
```javascript
askQuestion(index)      // Show next question
handleAnswer(value)     // Process user reply
handleEdit(questionId)  // Enable edit mode
```

**State**:
```javascript
chatHistory            // Array of messages
currentQuestionIndex   // Position in question list
editingQuestionId      // Current edit target (if any)
```

### UserInput Component

**Dynamic Input Rendering**:
```javascript
switch (question.inputType) {
  case 'select':     return <select>...
  case 'number':     return <input type="number">...
  case 'date':       return <input type="date">...
  case 'textarea':   return <textarea>...
  default:           return <input type="text">...
}
```

**Props**:
- `question`: Question object with validation rules
- `onSubmit`: Callback with user answer
- `initialValue`: For edit mode

### Dashboard Components

**IncomePie**: Pie chart from `ITR1_IncomeDeductions`
**TaxBar**: Bar chart from `ITR1_TaxComputation` + `TaxPaid`
**ProgressBar**: Completion percentage

All components receive `formData` and extract needed values using `deepGet`.

## Adding New Features

### Example: Add a New Input Type

1. **Update `determineInputType` in schemaParser.js**:
```javascript
function determineInputType(question) {
  if (question.format === 'email') return 'email';
  // ... existing logic
}
```

2. **Add case in UserInput.jsx**:
```javascript
case 'email':
  return <input type="email" {...commonProps} />;
```

### Example: Add New Chart

1. **Create component** in `components/Dashboard/`:
```javascript
// DeductionChart.jsx
export default function DeductionChart({ formData }) {
  const deductions = deepGet(formData, 'ITR.ITR1.DeductUndChapVIA');
  // ... chart logic
}
```

2. **Add to App.jsx**:
```javascript
import DeductionChart from './components/Dashboard/DeductionChart';

// In dashboard-panel
<DeductionChart formData={formData} />
```

### Example: Add Conditional Questions

In `schemaParser.js`:

```javascript
export function filterQuestionsByConditions(questions, formData) {
  const optOut = deepGet(formData, 'ITR.ITR1.FilingStatus.OptOutNewTaxRegime');
  
  if (optOut === 'N') {
    // New tax regime - hide deduction schedules
    return questions.filter(q => !q.id.includes('Schedule80'));
  }
  
  return questions;
}
```

## Utility Functions Reference

### schemaParser.js
- `parseSchema(schema)`: Convert schema to questions
- `getRequiredQuestions(questions)`: Filter required only
- `filterQuestionsByConditions(questions, formData)`: Conditional logic

### ajvHelpers.js
- `validateField(question, value)`: Single field validation
- `validateFullSchema(schema, data)`: Complete validation
- `formatErrors(errors)`: User-friendly error messages
- `checkFormCompleteness(questions, formData)`: Progress check

### deepPath.js
- `deepGet(obj, path, defaultValue)`: Safe nested get
- `deepSet(obj, path, value)`: Create & set nested value
- `deepHas(obj, path)`: Check path existence
- `deepDelete(obj, path)`: Remove nested value

### storage.js
- `saveDraft(data)`: Save to localStorage
- `loadDraft()`: Load from localStorage
- `clearDraft()`: Delete saved data
- `downloadJSON(data, filename)`: Trigger download
- `encryptDraft(data, password)`: Basic encryption (demo)
- `decryptDraft(encrypted, password)`: Basic decryption (demo)

## Debugging Tips

### Enable Debug Logging
```javascript
// In App.jsx or ChatBot.jsx
useEffect(() => {
  console.log('FormData updated:', formData);
}, [formData]);
```

### Inspect localStorage
```javascript
// Browser console
localStorage.getItem('itr1_draft_v1')
```

### Test Validation
```javascript
// Browser console
import { validateField } from './utils/ajvHelpers';

const question = { 
  label: 'PAN', 
  type: 'string', 
  pattern: '[A-Z]{5}[0-9]{4}[A-Z]',
  required: true 
};

console.log(validateField(question, 'ABCDE1234F'));
```

### Check Schema Parsing
```javascript
// Browser console
const questions = parseSchema(schema);
console.table(questions.map(q => ({
  id: q.id,
  label: q.label,
  type: q.type,
  required: q.required
})));
```

## Performance Optimization

### Current Optimizations
1. **Debounced Auto-Save**: 1-second delay before saving to localStorage
2. **Lazy Chart Rendering**: Only render charts when dashboard visible
3. **Memoization**: React components use functional updates

### Future Optimizations
1. **Virtual Scrolling**: For very long chat histories
2. **Web Workers**: Move schema parsing to background thread
3. **IndexedDB**: For larger data storage (>5MB)
4. **Code Splitting**: Lazy-load dashboard components

## Common Patterns

### Adding a New Section

```javascript
// 1. Identify section in schema
const newSection = schema.definitions.NewSection;

// 2. Parser will auto-extract if in ITR1.properties
// 3. Questions appear in chat automatically

// 4. Add chart if needed
function NewSectionChart({ formData }) {
  const data = deepGet(formData, 'ITR.ITR1.NewSection');
  // ... visualization
}
```

### Custom Validation

```javascript
// In ajvHelpers.js
export function validateField(question, value) {
  // Existing validations...
  
  // Add custom logic
  if (question.id.includes('PAN')) {
    if (!isValidPAN(value)) {
      return { valid: false, error: 'Invalid PAN checksum' };
    }
  }
  
  return { valid: true, error: null };
}
```

### Formatting Display Values

```javascript
// Currency formatting
const formatted = value.toLocaleString('en-IN', {
  style: 'currency',
  currency: 'INR'
});

// Date formatting
const formatted = new Date(value).toLocaleDateString('en-IN');
```

## API Reference

### ChatBot Props
```typescript
interface ChatBotProps {
  questions: Question[];
  formData: object;
  onUpdateFormData: (data: object) => void;
  onComplete?: () => void;
}
```

### Question Object
```typescript
interface Question {
  id: string;              // JSON path
  name: string;            // Field name
  label: string;           // Display label
  type: string;            // Schema type
  inputType: string;       // UI control type
  required: boolean;
  description?: string;
  pattern?: string;
  enum?: any[];
  minimum?: number;
  maximum?: number;
  minLength?: number;
  maxLength?: number;
  default?: any;
  example?: string;
}
```

## Troubleshooting

### Schema Not Loading
1. Check `public/ITR-1_2025_Main_V1.1.json` exists
2. Verify JSON is valid (use JSONLint)
3. Check browser console for fetch errors

### Validation Failing
1. Check AJV error messages in console
2. Verify formData structure matches schema
3. Use `validateFullSchema` to see all errors

### Charts Not Updating
1. Ensure formData is being updated correctly
2. Check that chart components receive new formData prop
3. Verify data extraction paths in chart components

### localStorage Full
1. Clear old drafts: `localStorage.clear()`
2. Reduce formData size by removing empty objects
3. Consider IndexedDB for larger datasets

## Best Practices

1. **Always validate user input** before updating formData
2. **Use deepGet/deepSet** for nested access
3. **Keep components pure** - avoid side effects in render
4. **Handle errors gracefully** - show user-friendly messages
5. **Test with real schema** - edge cases emerge
6. **Document validation rules** - help users understand requirements

## Security Considerations

1. **Never store passwords** in localStorage
2. **Sanitize all user input** before display
3. **Use HTTPS** in production
4. **Don't log sensitive data** (PAN, Aadhaar)
5. **Implement CSP headers** to prevent XSS
6. **Use proper encryption** for draft export (replace demo implementation)

## Next Steps

1. Add comprehensive unit tests
2. Implement E2E tests with Playwright
3. Add accessibility audit with axe-core
4. Create Storybook for component documentation
5. Set up CI/CD pipeline
6. Add error boundary for graceful error handling
7. Implement proper state management (if app grows)
8. Add internationalization (i18n)

---

**Questions?** Review the code comments or check the main README.md for high-level documentation.
