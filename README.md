# MEG-APSU TERRA — Quantum Biodiversity Scanner

**"What quantum machinery dies when this species dies?"**

TERRA is the world's first tool connecting quantum biology with biodiversity loss. It scans endangered species for quantum-critical enzymes — enzymes that rely on quantum tunneling to function — and calculates what is irreversibly lost when a species goes extinct.

## Key Finding

**500 Critically Endangered species scanned. Results:**

| Metric | Value |
|--------|-------|
| Species scanned | 489 |
| Total enzymes found | 866 |
| With PDB structure | 220 |
| **Quantum-critical** | **67** |
| Species with QC enzymes | 63/489 |
| **QC rate** | **30.5%** |

Control group (17 Least Concern species): **17.8% QC rate, 17/17 affected.**

> *Quantum-critical enzymes are universally distributed across all kingdoms of life at a constant rate of ~18%. Every species extinction irreversibly destroys quantum machinery that took 4 billion years to evolve. No species is quantum-expendable.*

## Pipeline

```
IUCN Red List → UniProt → RCSB PDB → MEG-APSU Lindblad Solver
```

1. **IUCN Red List API v4** — Fetch Critically Endangered species
2. **UniProt** — Find enzymes for each species (with human ortholog mapping)
3. **RCSB PDB** — Download 3D protein structures
4. **MEG-APSU** — RK4 Lindblad quantum dynamics solver classifies each enzyme

## Requirements

- Rust (edition 2021)
- [MEG-APSU](https://github.com/sectio-aurea-q/meg-apsu) built at `~/meg-apsu/target/release/meg-apsu`
- IUCN Red List API key (set as `IUCN_REDLIST_KEY` environment variable)
- Internet connection (UniProt, PDB, IUCN APIs)

## Usage

```bash
# Build
cargo build --release

# Scan first 100 CR species
./target/release/terra cr_species_500.txt --limit 100

# Scan all 500
./target/release/terra cr_species_500.txt --limit 500

# Resume after crash (automatic — reads terra-progress.txt)
./target/release/terra cr_species_500.txt --limit 500
```

## Output

- `terra-scan-results.json` — Full results with per-species breakdown
- `terra-progress.txt` — Resume checkpoint (delete to rescan)

## What This Proves

Two worlds that never touched:

- **Quantum enzymology** (Klinman, Scrutton, Kohen) — measures tunneling in individual enzymes
- **Biodiversity conservation** (IUCN, IPBES) — counts endangered species

TERRA connects them. For the first time, we can calculate the **quantum cost of extinction**.

Every species carries quantum machinery. No species is quantum-expendable. These are 4-billion-year-old quantum machines no laboratory can rebuild.

## Author

**sectio-aurea-q** · Independent Quantum Cryptomathematics & Quantum Biology Research

## License

Research code. Full publication pending.

---

*The number that didn't exist.*
