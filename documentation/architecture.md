# System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        USER INTERFACE                           │
│                   React Dashboard (Port 3000)                   │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│  │  Stats   │  │ Defects  │  │  Charts  │  │  Table   │         │
│  │  Cards   │  │   Pie    │  │   Line   │  │ Recent   │         │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘         │
│                                                                 │
│             Auto-refresh every 5 seconds                        │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            │ HTTP GET Requests
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│                     API LAYER (FastAPI)                         │
│                    Backend (Port 8000)                          │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Dashboard Endpoints (NEW)                  │    │
│  │  • GET /api/stats                                       │    │
│  │  • GET /api/recent-detections                           │    │
│  │  • GET /api/defect-distribution                         │    │
│  │  • GET /api/performance-metrics                         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │         Prediction Endpoints (EXISTING)                 │    │
│  │  • POST /predict (base64)                               │    │
│  │  • POST /predict/file (multipart)                       │    │
│  │  • POST /debug/predict                                  │    │
│  │  • GET /health                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            │ Read/Write Operations
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│                      DATA STORAGE                               │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │ 
│  │         logs/predictions_YYYY-MM-DD.json                │    │
│  │  {                                                      │    │
│  │    "timestamp": "2025-12-18T10:30:00",                  │    │
│  │    "traditional_result": {...},                         │    │
│  │    "few_shot_result": {...},                            │    │
│  │    "processing_time": 0.142,                            │    │
│  │    "success": true                                      │    │
│  │  }                                                      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────┐
│                      ML MODEL LAYER                             │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │  YOLO Model  │  │  Traditional │  │   Few-Shot   │           │
│  │   (Masking)  │  │  Classifier  │  │  Prototypical│           │
│  │              │  │  (MobileNet/ │  │   Network    │           │
│  │  weights.pt  │  │   ResNet18)  │  │              │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘


DATA FLOW:
═════════

1. IMAGE UPLOAD → /predict or /predict/file
   ↓
2. YOLO Masking → Extract rail region
   ↓
3. ML Classification → Traditional + Few-Shot models
   ↓
4. LOG RESULT → logs/predictions_YYYY-MM-DD.json
   ↓
5. DASHBOARD POLL → API endpoints every 5 seconds
   ↓
6. AGGREGATE DATA → Read and process log files
   ↓
7. RETURN JSON → Formatted data for charts/tables
   ↓
8. UPDATE UI → React components re-render


INTEGRATION POINTS:
══════════════════

Frontend (React) ←→ Backend (FastAPI)
     │                      │
     │   HTTP GET /api/*    │
     │─────────────────────→│
     │                      │
     │    JSON Response     │
     │←─────────────────────│
     │                      │
     │   Every 5 seconds    │
     │                      │


Backend (FastAPI) ←→ Log Files
     │                      │
     │   Read aggregation   │
     │─────────────────────→│
     │                      │
     │   Write predictions  │
     │←─────────────────────│
     │                      │


User ───→ /predict ───→ Models ───→ Log ───→ Dashboard
         (Password)                  (JSON)   (Auto-refresh)


KEY FEATURES:
════════════

✓ No modification to existing endpoints
✓ Real-time updates (5 second polling)
✓ Separate concerns (prediction vs monitoring)
✓ Scalable architecture
✓ Error handling at each layer
✓ CORS enabled for cross-origin requests
