# Clustering Module

Clustering groups structurally similar crystals together, enabling fitness sharing and diversity maintenance.

---

## Configuration

```ini
[modules]
clustering_module = cluster

[clustering]
clustering_algorithm = AffinityPropagation
feature_vector = RSF
```

---

## Clustering Algorithm

### Affinity Propagation

The default clustering algorithm. It automatically determines the number of clusters without requiring a predefined count.

```ini
[clustering]
clustering_algorithm = AffinityPropagation
```

---

## Feature Vectors

The feature vector defines how structures are represented for clustering similarity comparison.

### RSF (Radial Structure Function)

Encodes the radial distribution of atoms:

```ini
[clustering]
feature_vector = RSF
```

### RCD (Relative Coordinate Descriptor)

Encodes the relative positions and orientations of molecules:

```ini
[clustering]
feature_vector = RCD
```

---

## Fitness Sharing

When clustering is active, the fitness of each structure is divided by the number of members in its cluster:

$$F_{\text{shared}} = \frac{F}{\text{cluster\_size}}$$

This penalizes structures in densely populated regions of the energy landscape, encouraging the GA to explore diverse structural motifs rather than converging on a single basin.
