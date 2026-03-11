# Project Structure

## Repository Layout

```
GAtor/
├── gator/                    # Main Python package
│   ├── GAtor_master.py       # Entry point
│   ├── core/                 # Core GA engine
│   │   ├── run_GA.py         # Main GA loop
│   │   ├── user_input.py     # Configuration parser
│   │   ├── file_handler.py   # File/path management
│   │   ├── data_tools.py     # Energy ranking & output
│   │   ├── check_conf.py     # Configuration validator
│   │   ├── output.py         # Logging utilities
│   │   └── activity.py       # Activity heartbeat
│   ├── optimization/         # Energy calculators
│   │   ├── MACE.py           # MACE MLIP
│   │   ├── UMA.py            # UMA (FAIRChem) MLIP
│   │   ├── AIMNET2.py        # AIMNet2 MLIP
│   │   ├── FHI_aims.py       # FHI-aims DFT
│   │   ├── VASP.py           # VASP DFT
│   │   └── rigid_press.py    # Rigid-body pressure optimizer
│   ├── selection/            # Parent selection strategies
│   ├── crossover/            # Crossover operators
│   ├── mutation/             # Mutation operators
│   ├── fitness/              # Fitness evaluation
│   ├── comparison/           # Duplicate detection
│   ├── clustering/           # Structure clustering
│   ├── structures/           # Structure data model
│   ├── initial_pool/         # Initial pool construction
│   ├── analysis/             # Post-run analysis tools
│   ├── utilities/            # Helper modules
│   ├── property/             # Property calculators (Hab)
│   ├── cgenarris/            # PyGenarris C extension
│   └── external_libs/        # Bundled dependencies
├── ibslib/                   # Structure I/O library
├── example/                  # Example configurations
├── test/                     # Test cases
├── doc/                      # This documentation
├── setup.py                  # Package installer
├── pyproject.toml            # Build system config
└── requirements.txt          # Python dependencies
```

## Runtime Directory Layout

When GAtor runs, it creates the following structure in your working directory:

```
working_directory/
├── ui.conf                              # Your configuration file
├── structures.json                      # Input structures (if using prepare_initial_pool)
├── mol.in / mol_1.in, mol_2.in          # Extracted molecule geometry
├── initial_pool/                        # Optimized initial pool structures
│   ├── struct_001.json
│   ├── struct_002.json
│   └── ...
├── tmp/                                 # Runtime data (main output directory)
│   ├── pool_0/                          # Structure pool for stoichiometry 0
│   │   ├── <struct_id>.json             # Individual structure files
│   │   └── ...
│   ├── energy_hierarchy_*.dat           # Ranked energy table
│   ├── num_IP_structs.dat               # Initial pool count
│   ├── initial_pool_space_groups.json   # Space group analysis
│   ├── conf/                            # Replica configuration files
│   ├── adaptation_candidate.json        # Adaptive selection data
│   └── unique_mol_dict.json             # Conformer pool
├── output/                              # Replica log files
│   ├── <replica_id>.log
│   └── ...
├── GAtor.log                            # Master log
├── GAtor.err                            # Error log
└── save.json                            # Checkpoint for restart
```

## Key File Formats

| File | Format | Description |
|------|--------|-------------|
| `ui.conf` | INI (ConfigParser) | GA configuration |
| `structures.json` | JSON (ASE) | Input crystal structures |
| `*.json` (in pool) | JSON (GAtor) | Individual structure with properties |
| `mol.in` | XYZ-like (ASE) | Reference molecule geometry |
| `energy_hierarchy_*.dat` | Fixed-width text | Ranked structures table |
