# H3 Core Logic Analysis — Ambulance Dispatch System

**Prepared by:** Kaipa Purandeswar Reddy  
**Source files reviewed:**  
- `h3api.h.in` — Public C API declarations  
- `h3Index.c` — Index encoding/decoding  
- `algos.c` — Grid traversal algorithms  
- `localij.c` — IJ coordinate space & distance  
- `faceijk.c` — Icosahedron face math  
- `h3-go/h3.go` — Go CGo bindings  
- `h3-go/h3_test.go` — Go test suite  

---

## 1. How the Three Core Functions Work

### 1.1 `latLngToCell(latLng, resolution) → Cell`

**C signature:**
```c
H3Error latLngToCell(const LatLng *g, int res, H3Index *out);
```
**Go binding:**
```go
func LatLngToCell(latLng LatLng, resolution int) (Cell, error)
```

**What it does internally (`faceijk.c`):**

1. Converts lat/lng (in radians internally) to a 3D point on the unit sphere.
2. Projects onto the nearest icosahedron face using face-centred gnomonic projection.
3. Converts the 2D face coordinates into a hierarchical IJK coordinate system.
4. Encodes the result as a 64-bit `H3Index` (the cell token).

**Key data type:**
```c
typedef uint64_t H3Index;   // also exposed as int64 in Go (Cell type)
```

**LatLng note:** The public API accepts **degrees** in Go (`LatLng{Lat, Lng float64}`). Internally the C library converts to radians. Do not pre-convert in Go code.

**Error to watch for:** `E_LATLNG_DOMAIN` (lat outside ±90°, lng outside ±180°) and `E_RES_DOMAIN` (resolution outside 0–15).

---

### 1.2 `gridDisk(origin, k) → []Cell`

**C signature:**
```c
H3Error gridDisk(H3Index origin, int k, H3Index *out);
// out must be pre-allocated to maxGridDiskSize(k)
```
**Go binding:**
```go
func GridDisk(origin Cell, k int) ([]Cell, error)
```

**What it does internally (`algos.c`):**

`gridDisk` is a thin wrapper over `gridDiskDistances`. The actual algorithm:

1. **Fast path — `gridDiskDistancesUnsafe`:** Rotates around the origin hexagon using direction arrays. This is O(k²) and very fast, but **fails near pentagons**.
2. **Safe fallback — `_gridDiskDistancesInternal`:** A recursive BFS-style traversal. Used automatically if the fast path fails (pentagon distortion area).

**Output size formula (from `algos.c`):**
```
maxGridDiskSize(k) = 3*k*(k+1) + 1
```
| k | cells returned |
|---|---|
| 0 | 1 (just origin) |
| 1 | 7 |
| 2 | 19 |
| 3 | 37 |

**Important:** Output array **may contain zeros** (`H3_NULL = 0`) when crossing a pentagon. The Go binding strips these by default (`cellsFromC(out, true, false)`).

**Pentagon caveat:** There are exactly 12 pentagon cells in H3 (one per icosahedron vertex, replicated at every resolution). If your ambulance or emergency is near one, `gridDisk` silently falls back to the safe algorithm — you don't need to handle this yourself, but be aware the output cell count may be slightly less than the formula predicts.

---

### 1.3 `gridDistance(a, b) → int`

**C signature:**
```c
H3Error gridDistance(H3Index origin, H3Index index, int64_t *distance);
```
**Go binding:**
```go
func GridDistance(a, b Cell) (int, error)
```

**What it does internally (`localij.c`):**

1. Converts both cells to **local IJ coordinates** anchored on `origin` using `cellToLocalIjk`.
2. Computes `ijkDistance` — the grid hop count in the hexagonal IJK space:
   ```
   distance = max(|i1-i2|, |j1-j2|, |k1-k2|)
   ```
   (Equivalent to the Chebyshev distance in the IJ plane after normalisation.)

**Failure modes to handle in dispatch code:**
- `E_PENTAGON` — path crosses a pentagon; distance is undefined.
- `E_FAILED` — cells are too far apart for the local IJ projection to work reliably (very large distances). At resolution 9 (typical city-level), this is rarely a problem.
- Cells at **different resolutions** → `E_RES_MISMATCH`. Always index ambulances and emergencies at the **same resolution**.

---

## 2. Mapping the Flow to the API

```
Ambulance Update
    ↓
LatLngToCell(ambulanceLat, ambulanceLng, resolution)
    → stores Cell token as the map key

Emergency Request
    ↓
LatLngToCell(emergencyLat, emergencyLng, resolution)
    ↓
GridDisk(emergencyCell, k=1)          // returns ~7 candidate cells
    ↓
for each cell in disk:
    ambulances = index[cell]          // O(1) lookup
    if len(ambulances) > 0:
        for each ambulance:
            dist, err = GridDistance(ambulanceCell, emergencyCell)
            // handle err (pentagon, mismatch)
        choose min(dist) → Dispatch
    else:
        k++; repeat (k=2 → 19 cells, k=3 → 37 cells, ...)
```

---

## 3. Resolution Choice

For urban ambulance dispatch in India, **resolution 9** is the recommended default:

| Resolution | Avg hex area | Avg edge length |
|---|---|---|
| 8 | ~0.74 km² | ~0.46 km |
| **9** | **~0.105 km²** | **~0.17 km** |
| 10 | ~0.015 km² | ~0.065 km |

Resolution 9 cells are roughly 300–400 m across — large enough to bucket multiple ambulances in a dense area, small enough to be meaningful for nearest-unit dispatch.

At res 9:
- `GridDisk(k=1)` covers ~1.2 km radius — sufficient for dense urban zones.
- `GridDisk(k=3)` covers ~2.5 km — good for semi-urban fallback.

---

## 4. Critical Implementation Notes

### 4.1 Always check H3Error

Every function returns `H3Error`. In Go this maps to a standard `error`. **Never silently discard it:**
```go
cell, err := h3.LatLngToCell(latLng, resolution)
if err != nil {
    // log + fallback, don't crash dispatch
}
```

### 4.2 gridDistance can fail — build a fallback

```go
dist, err := h3.GridDistance(ambulanceCell, emergencyCell)
if err != nil {
    // Fall back to Haversine (great-circle distance) as a proxy
    dist = haversineApprox(ambulanceLat, ambulanceLng, emergencyLat, emergencyLng)
}
```

`gridDistance` returns grid hops, not metres. For final ranking, if precision matters, use `greatCircleDistanceM` from h3api for the shortlisted top 2–3 ambulances.

### 4.3 H3Index is a value type — safe as a map key

`H3Index` is `uint64_t` (C) / `int64` (Go). It can be used directly as:
- A Go `map[h3.Cell][]AmbulanceID` key
- A Redis key (stringify with `h3ToString` or `.String()` in Go)
- A PostgreSQL `bigint` column

### 4.4 Ambulance moves frequently — update the index atomically

When an ambulance moves from cell A to cell B:
1. Remove from `index[cellA]`
2. Add to `index[cellB]`

Do both in one operation (mutex or Redis transaction) to avoid a race where an ambulance appears in neither cell during dispatch.

### 4.5 gridDisk output order is unspecified

The C code places cells "in no particular order." Do **not** assume the first element is the origin or closest — always run `gridDistance` to rank.

---

## 5. File-by-File Summary for the Maintainer

| File | Role in your system |
|---|---|
| `h3api.h.in` | Public API contract. `latLngToCell`, `gridDisk`, `gridDistance`, `maxGridDiskSize` are all declared here. All return `H3Error`. |
| `h3Index.c` | Bit-level encoding of the 64-bit cell token. Resolution (4 bits), base cell (7 bits), 15 digit levels (3 bits each). Not called directly. |
| `algos.c` | Implements `gridDisk` and `maxGridDiskSize`. The `3k(k+1)+1` formula is here. Contains the pentagon fallback logic. |
| `localij.c` | Implements `gridDistance` via IJ projection. Anchor-based — both cells must be "close" in grid terms. Source of `E_PENTAGON` / `E_FAILED` errors. |
| `faceijk.c` | Handles the icosahedron → IJ coordinate math used by `latLngToCell`. Internally called; not invoked directly. |
| `h3-go/h3.go` | Go CGo bindings. `LatLngToCell`, `GridDisk`, `GridDistance` are the three functions your dispatch loop calls. |
| `h3-go/h3_test.go` | Test suite. Reference for expected behaviour: valid cell checks, pentagon handling, resolution edge cases. |
