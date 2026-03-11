# FAQ & Troubleshooting

## General

??? question "How long does a typical GAtor run take?"
    It depends heavily on the energy calculator and system size:

    - **MLIP (MACE/UMA)**: 2-8 hours on 1-4 GPU nodes for 500-1000 structures
    - **DFT (FHI-aims)**: Days to weeks on 8-32 CPU nodes

??? question "How do I know when the GA has converged?"
    Monitor the `energy_hierarchy_*.dat` file. When the lowest energies plateau and new structures are consistently rejected as duplicates, the GA has likely explored the relevant region of the landscape.

??? question "Can I restart a run that was interrupted?"
    Yes. Set `restart_replicas = TRUE` in `[run_settings]`. GAtor will detect existing pool structures and continue. For initial pool preparation, set `restart = TRUE` in `[initial_pool]`.

??? question "How many initial structures do I need?"
    A good starting pool has 50-200 structures with diverse space groups and packing motifs. More is generally better, but diminishing returns set in around 200.

---

## Installation Issues

??? question "Error: Unable to detect MPI C compiler mpicc"
    Load your MPI module before installing: `module load cray-mpich` (or your system's equivalent), then retry `pip install -e .`

??? question "ImportError: No module named 'mace'"
    Install the MACE package: `pip install mace-torch`

??? question "ImportError: No module named 'fairchem'"
    Install FAIRChem: `pip install fairchem-core`

---

## Runtime Issues

??? question "CUDA out of memory"
    - Reduce `replicas_per_node` (fewer replicas sharing GPU memory)
    - Use a smaller model (e.g., MACE `medium` instead of `large`)
    - Ensure only one replica per GPU

??? question "Structures failing SR check"
    The specific radius (SR) check is rejecting structures with atoms too close together. Adjust:
    ```ini
    [cell_check_settings]
    specific_radius_proportion = 0.60    # Lower = more permissive
    ```

??? question "All structures rejected as duplicates"
    Your comparison tolerances may be too loose. Tighten them:
    ```ini
    [post_relaxation_comparison]
    ltol = 0.3
    stol = 0.3
    angle_tol = 5
    ```

??? question "GAtor hangs at 'Predicted unit cell volume'"
    This means the master process is waiting for the volume prediction to complete. Check that your molecule files (`mol.in` or `mol_1.in`, `mol_2.in`) exist and are valid.

??? question "Energy evaluations are failing"
    - Check that the optimization module executable is accessible
    - For FHI-aims: verify `path_to_aims_executable` and `control_in_directory`
    - For MLIPs: verify GPU availability with `python -c "import torch; print(torch.cuda.is_available())"`

??? question "What are qualifying duplicates?"
    When a structure is detected as a duplicate, GAtor may save it to a separate `duplicates/` directory if the RMSD between the new and matched structure is >= 0.2 A **and** the match involves the lowest-energy structure (pre-relaxation) or the new structure has global lowest energy (post-relaxation). These near-duplicates are tracked in `GA_duplicates.dat` and can be useful for analyzing the energy landscape around key minima.

??? question "Pool is not growing"
    Common causes:

    1. Energy thresholds too strict → increase `relative_energy_thresholds`
    2. Comparison too loose → all structures detected as duplicates → tighten tolerances
    3. SR check too strict → lower `specific_radius_proportion`
    4. Volume bounds too tight → widen `volume_upper_ratio` / `volume_lower_ratio`

---

## Performance Tips

??? tip "Speed up MLIP evaluations"
    - Use `run_on_gpu = TRUE` with GPU nodes
    - Match `replicas_per_node` to GPUs per node
    - Set `fmax = 0.05` for faster (but less accurate) relaxation
    - Reduce `steps` to 500-1000

??? tip "Reduce I/O bottlenecks"
    - Use a local scratch filesystem for the working directory if available
    - The shared pool uses file-based locking; fast I/O reduces contention

??? tip "Improve search quality"
    - Use more replicas (8-16) for better diversity
    - Enable clustering for fitness sharing
    - Use `Adaptive_tournament_selection` for automatic selection pressure tuning
    - Include diverse mutation types (translation + rotation + strain)
