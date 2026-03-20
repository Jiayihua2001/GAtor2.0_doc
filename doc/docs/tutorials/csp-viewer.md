# Tutorial 7: CSP Landscape Viewer

An interactive React-based web application for exploring crystal structure prediction results.

---

## Features

- **Energy landscape** — ΔE vs density scatter plots, colored by space group
- **PXRD comparison** — Simulated vs experimental patterns with similarity scores
- **Structure browser** — Sortable/filterable table with CIF export
- **Ancestry visualization** — Trace evolutionary history of any structure

## Setup

### 1. Create a React Project

```bash
npx create-react-app csp-viewer
cd csp-viewer
```

### 2. Install Dependencies

```bash
npm install recharts lucide-react
```

### 3. Add the Viewer Component

```bash
cp /path/to/GAtor/examples/06_post_analysis/csp-viewer-v2.jsx src/CSPViewer.jsx
```

### 4. Update `src/App.js`

```jsx
import CSPViewer from './CSPViewer';

function App() {
  return <CSPViewer />;
}

export default App;
```

### 5. Launch

```bash
npm start
```

Opens at `http://localhost:3000`.

---

## Preparing Data

Copy the relevant output files from your GAtor run:

```bash
# On the HPC cluster
cd /path/to/gator/run
tar czf gator_results.tar.gz \
    tmp/energy_hierarchy_*.dat \
    tmp/exp_match_record.dat \
    structures/ \
    match/

# On your local machine
scp user@hpc:/path/to/gator_results.tar.gz .
tar xzf gator_results.tar.gz
```

The viewer reads standard GAtor output files:

| File | Content |
|---|---|
| `energy_hierarchy_*.dat` | Ranked structures with energy, volume, lattice params, SG |
| `exp_match_record.dat` | GA step, match status, RMSD, energy |
| `match/*.cif` | Matched structure files for 3D viewing |

---

## Using the Viewer

| Tab | What You Can Do |
|---|---|
| **Energy Landscape** | Scatter plot of ΔE vs density; hover for details, click to select |
| **PXRD Comparison** | Overlay simulated vs experimental PXRD; view VC-PWDF score |
| **Structure Browser** | Sort/filter all structures by energy, SG, or GA iteration; export CIF |
| **Ancestry** | Tree view of parent-child relationships with operator annotations |

### Typical Analysis Session

1. **Overview** — Open energy landscape to see the full distribution
2. **Identify candidates** — Look for low-energy structures in the correct space group
3. **PXRD validation** — Check which structures match the experimental pattern
4. **Ancestry** — Trace back the best matches to understand how the GA found them
5. **Export** — Download structures as CIF for further analysis

---

## Tips

!!! tip "Publication Figures"
    The Python scripts (`plt_convergence.py`, `plt_pool.py`) produce higher-resolution figures for publications. Use the viewer for interactive exploration.

!!! tip "Large Runs"
    For runs with thousands of structures, use the filter controls to narrow down to the most interesting subset.
