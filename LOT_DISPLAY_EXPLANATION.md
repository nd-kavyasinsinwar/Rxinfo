# How LoT (Line of Therapy) is Displayed - Complete Explanation

## Overview
LoT (Line of Therapy) shows an **individual patient's treatment sequence** - the chronological progression of medications/therapies they've received over time. It's visualized as a **Sankey diagram** showing the flow from one treatment line to the next.

---

## 📊 Complete Data Flow

### **Step 1: Data Fetching** 
**Location:** `ventricle/app/evidence/page.tsx` (lines 158-208)

**When it triggers:**
- When user navigates to Evidence page
- When `patientId` is available in URL parameters
- Automatically re-fetches if `patientId` or `diagnosisCode` changes

**API Call:**
```javascript
GET http://10.91.20.207:8050/api/patients/lot?patient_id={patientId}&diagnosis_code={diagnosisCode}
```

**Request Headers:**
- `X-Api-Version: v1`
- `accept: application/json`

**What it fetches:**
- All treatment lines for a specific patient
- Optionally filtered by diagnosis code
- Returns multiple diagnoses, each with their own LoT sequence

---

### **Step 2: API Response Structure**

**Response Format:**
```json
{
  "message": "Successfully processed",
  "result": [
    {
      "diagnosis_code": "G309",
      "diagnosis_name": "Alzheimer's disease, unspecified",
      "lot": [
        {
          "patientid": 142815,
          "linename": "vitamin e",
          "linenumber": 1,
          "linestart": "2023-08-04T00:00:00Z",
          "lineend": "2023-08-24T00:00:00Z"
        },
        {
          "patientid": 142815,
          "linename": "donepezil + vitamin e",
          "linenumber": 2,
          "linestart": "2023-08-24T00:00:00Z",
          "lineend": "2023-09-03T00:00:00Z"
        },
        // ... more lines
      ]
    },
    // ... more diagnoses
  ],
  "metrics": { "time_taken": 15116 },
  "request_id": "30b109bf470801634634c89a6c8143d7"
}
```

**Key Fields:**
- `diagnosis_code`: ICD-10 code (e.g., "G309")
- `diagnosis_name`: Full diagnosis name
- `lot[]`: Array of treatment lines
  - `linename`: Medication/treatment name
  - `linenumber`: Sequential line number (1, 2, 3...)
  - `linestart`: When treatment started (ISO date)
  - `lineend`: When treatment ended (ISO date)

---

### **Step 3: Data Selection**
**Location:** `ventricle/app/evidence/page.tsx` (lines 195-198)

**Logic:**
```javascript
const arr: LoTData[] = json?.result || [];
// If diagnosis code provided, find that specific diagnosis
// Otherwise, show the first diagnosis
const diag = diagnosisCode 
  ? arr.find(d => d.diagnosis_code === diagnosisCode) 
  : arr[0];
setLotDiagnosis(diag || null);
```

**What happens:**
- If user selected a specific diagnosis → show that diagnosis's LoT
- If no diagnosis selected → show first diagnosis's LoT
- Stores selected diagnosis in state: `lotDiagnosis`

---

### **Step 4: Data Processing for Visualization**
**Location:** `ventricle/app/components/LotAnalysisView.tsx` (lines 27-102)

**Function:** `processLoTDataForPlotlySankey()`

**Processing Steps:**

#### **4.1. Sort by Line Number**
```javascript
const sorted = [...lot].sort((a, b) => {
  const aNum = a.linenumber || a.line_number || 0;
  const bNum = b.linenumber || b.line_number || 0;
  return aNum - bNum; // Ascending: 1, 2, 3, 4...
});
```
- Ensures treatments are in chronological order

#### **4.2. Create Unique Node Labels**
```javascript
const createNodeLabel = (entry, index) => {
  const name = entry?.linename?.trim() || '';
  const lineNum = entry?.linenumber || (index + 1);
  return `${name} (L${lineNum})`; // e.g., "vitamin e (L1)"
};
```
- Combines medication name with line number
- Prevents cycles when same medication appears multiple times
- Example: "vitamin e (L1)" vs "vitamin e (L4)" are different nodes

#### **4.3. Build Node Map**
```javascript
// Add "Start" node (index 0)
nodeMap.set('Start', 0);
labels.push('Start');

// Add all medication nodes
sorted.forEach((entry, index) => {
  const nodeLabel = createNodeLabel(entry, index);
  if (!nodeMap.has(nodeLabel)) {
    nodeMap.set(nodeLabel, labels.length);
    labels.push(nodeLabel);
  }
});
```
- Creates a map: `{ "Start": 0, "vitamin e (L1)": 1, "donepezil (L2)": 2, ... }`
- Each node gets a unique index for Plotly

#### **4.4. Build Flow Connections**
```javascript
// Start → First medication
source.push(0); // Start node
target.push(nodeMap.get(firstNode)); // First medication node
value.push(2); // Flow weight

// Each line → Next line
for (let i = 0; i < sorted.length - 1; i++) {
  source.push(nodeMap.get(fromNode)); // Current line
  target.push(nodeMap.get(toNode));   // Next line
  value.push(2); // Flow weight
}
```

**Result Structure:**
```javascript
{
  label: ["Start", "vitamin e (L1)", "donepezil (L2)", ...],
  source: [0, 1, 2, ...],      // Source node indices
  target: [1, 2, 3, ...],      // Target node indices
  value: [2, 2, 2, ...]        // Flow weights
}
```

**What this means:**
- `source[0] = 0, target[0] = 1` → Flow from "Start" to "vitamin e (L1)"
- `source[1] = 1, target[1] = 2` → Flow from "vitamin e (L1)" to "donepezil (L2)"
- Creates sequential flow: Start → L1 → L2 → L3 → ...

---

### **Step 5: Visualization with Plotly Sankey**
**Location:** `ventricle/app/components/LotAnalysisView.tsx` (lines 183-209)

**Chart Configuration:**

#### **5.1. Data Structure**
```javascript
const data = [{
  type: 'sankey',
  orientation: 'h',        // Horizontal flow
  arrangement: 'perpendicular', // Sequential layout
  node: {
    pad: 20,               // Space between nodes
    thickness: 25,         // Node width
    label: plotlyData.label, // ["Start", "vitamin e (L1)", ...]
    color: nodeColorArray  // Blue shades for each node
  },
  link: {
    source: plotlyData.source, // [0, 1, 2, ...]
    target: plotlyData.target, // [1, 2, 3, ...]
    value: plotlyData.value,   // [2, 2, 2, ...]
    color: linkColorArray      // Blue gradient for flows
  }
}];
```

#### **5.2. Layout Configuration**
```javascript
const layout = {
  width: chartWidth,       // Dynamic: min 1200px, 200px per node
  height: 420,
  font: { size: 11, family: 'Inter' },
  paper_bgcolor: 'transparent',
  margin: { l: 20, r: 20, t: 20, b: 20 }
};
```

#### **5.3. Dynamic Width Calculation**
```javascript
const nodeCount = plotlyData.label.length;
const chartWidth = Math.max(1200, nodeCount * 200);
```
- More treatment lines = wider chart
- Minimum 1200px for readability
- Each node gets ~200px width

---

### **Step 6: Rendering**
**Location:** `ventricle/app/components/LotAnalysisView.tsx` (lines 227-275)

**Component States:**

1. **Loading State:**
   ```jsx
   if (isLoading) {
     return <div>Loading animation...</div>;
   }
   ```

2. **No Data State:**
   ```jsx
   if (!diagnosis || !diagnosis.lot?.length) {
     return <div>No Line of Therapy data available</div>;
   }
   ```

3. **Chart Rendering:**
   ```jsx
   <Plot
     data={data}
     layout={layout}
     config={{ displayModeBar: false, responsive: false }}
   />
   ```

**Visual Result:**
- Horizontal Sankey diagram
- Nodes: Vertical bars labeled with medication names
- Links: Flowing ribbons connecting sequential treatments
- Colors: Blue gradient (matches brand colors)
- Scrollable: Horizontal scroll for long sequences

---

### **Step 7: User Interaction**

**Navigation:**
1. User clicks "LoT Analysis" in sidebar
2. `activeView` state changes to `'lot'`
3. Component renders `LotAnalysisView`

**Scrolling:**
- If chart is wider than viewport → horizontal scrollbar appears
- Scrollbar styled with custom CSS (visible thumb, grab cursor)
- Page-level scrolling also enabled for LoT view

**Responsive Behavior:**
- Chart width adapts to number of treatment lines
- Minimum width ensures readability
- Works on different screen sizes

---

## 🎯 Key Points to Explain

### **What is LoT?**
- **Line of Therapy** = Sequential treatments a patient has received
- Shows progression: Treatment 1 → Treatment 2 → Treatment 3...
- Each "line" represents a medication or combination used during a time period

### **Why Sankey Diagram?**
- **Visual Flow:** Shows how patient moved from one treatment to next
- **Sequential Nature:** Perfect for showing chronological progression
- **Individual Patient Focus:** Unlike typical Sankey (cohort distribution), this shows ONE patient's journey

### **Data Structure:**
- **Multiple Diagnoses:** Patient can have LoT for different conditions
- **Chronological Order:** Lines sorted by `linenumber` (1, 2, 3...)
- **Time Periods:** Each line has start/end dates (not shown in chart, but in data)

### **Visualization Details:**
- **Nodes:** Each treatment line (e.g., "vitamin e (L1)")
- **Links:** Flows showing progression from one line to next
- **Start Node:** Artificial starting point for visualization
- **Unique Labels:** Same medication in different lines = different nodes (prevents cycles)

### **Technical Implementation:**
- **Plotly.js:** Industry-standard visualization library
- **React Hooks:** `useMemo` for efficient data processing
- **Dynamic Sizing:** Chart width adapts to data
- **Error Handling:** Graceful fallbacks for missing data

---

## 📝 Example Walkthrough

**Scenario:** Patient with Alzheimer's disease

1. **API Returns:**
   - Diagnosis: "Alzheimer's disease, unspecified" (G309)
   - 21 treatment lines

2. **Processing:**
   - Sorted: L1, L2, L3... L21
   - Nodes created: "Start", "vitamin e (L1)", "donepezil + vitamin e (L2)", ...
   - Flows: Start→L1, L1→L2, L2→L3, ... L20→L21

3. **Visualization:**
   - 22 nodes (Start + 21 treatments)
   - 21 flows (connections between lines)
   - Chart width: ~4200px (22 nodes × 200px)
   - Horizontal scrollbar appears

4. **User Sees:**
   - Flowing diagram showing treatment progression
   - Can scroll horizontally to see all treatments
   - Each treatment clearly labeled with line number

---

## 🔧 Configuration Points

**API Endpoint:** `http://10.91.20.207:8050/api/patients/lot`

**Query Parameters:**
- `patient_id` (required): Patient identifier
- `diagnosis_code` (optional): Filter by specific diagnosis

**Chart Settings:**
- Minimum width: 1200px
- Width per node: 200px
- Height: 420px
- Node colors: Blue gradient (#125AD3, #003183, ...)
- Link colors: Blue with opacity (rgba)

**Responsive Breakpoints:**
- Standard: 450px container width
- Small screens: 344px container width

---

## 🐛 Error Handling

1. **No Patient ID:** Shows "No data" message
2. **API Failure:** Sets `lotDiagnosis` to `null`, shows empty state
3. **No LoT Data:** Shows "No Line of Therapy data available"
4. **Invalid Data:** Shows "Not enough data to display flow diagram"
5. **Rendering Errors:** Try-catch blocks prevent crashes

---

## 📊 Summary

**Complete Flow:**
```
User clicks "LoT Analysis" 
  → useEffect triggers API call
  → GET /patients/lot?patient_id=X
  → API returns diagnosis array with LoT data
  → Select diagnosis (first or by code)
  → Process data: sort, create nodes, build flows
  → Convert to Plotly format
  → Render Sankey diagram
  → User sees flowing treatment sequence
```

**Key Technologies:**
- **Frontend:** React, Next.js, Plotly.js
- **Data Processing:** JavaScript/TypeScript
- **API:** RESTful endpoint
- **Visualization:** Sankey diagram (Plotly)

**Purpose:**
- Show individual patient's treatment history
- Visualize sequential progression
- Help clinicians understand treatment journey
- Support clinical decision-making

