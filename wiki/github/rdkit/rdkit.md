# rdkit/rdkit

> The default open-source cheminformatics toolkit: a C++ core with Python-first bindings that most of pharma and molecular ML quietly depends on.

[GitHub repo](https://github.com/rdkit/rdkit) ·
[Official website](https://www.rdkit.org) ·
[License: BSD-3-Clause](https://github.com/rdkit/rdkit/blob/master/license.txt)

## Overview

RDKit is a cheminformatics library: it parses chemical structures (SMILES, SMARTS, MOL/SDF, InChI), builds an in-memory molecule graph, and provides the operations computational chemistry is built from — substructure search, canonicalization, fingerprints, ~200 molecular descriptors, 2D depiction, 3D conformer generation, force fields (UFF, MMFF94), and reaction transforms. Development began around 2000 at Rational Discovery; Greg Landrum open-sourced it under BSD in June 2006 and has been the primary maintainer since[^1].

The 3.5k GitHub stars badly undercount its footprint. RDKit is the de facto open toolkit inside pharma and biotech, the chemistry layer under DeepChem, datamol, Open Force Field tooling, and effectively every "ML for molecules" paper's preprocessing script; distribution runs through conda-forge and PyPI, not GitHub. (GitHub also reports the repo language as "HTML" because of committed documentation — the core is C++, wrapped for Python via Boost.Python, plus SWIG-generated Java/C# wrappers and an emscripten JavaScript build.)

The defining tension: chemistry correctness over API ergonomics. RDKit enforces a chemical model — sanitization, aromaticity perception, valence checking — and refuses input that violates it, which is exactly what you want in a compound-registration system and exactly what surprises ML engineers who just want tensors out of SMILES strings. The API shows its Boost.Python age: C++-ish naming, `None` returns instead of exceptions, and errors logged to C++ streams rather than Python logging.

## Getting Started

```bash
conda install -c conda-forge rdkit   # recommended by upstream
# or
pip install rdkit                    # official wheels
```

```python
from rdkit import Chem
from rdkit.Chem import AllChem, Descriptors

mol = Chem.MolFromSmiles("CC(=O)Oc1ccccc1C(=O)O")  # aspirin
if mol is None:                 # parse failure returns None — it does NOT raise
    raise ValueError("bad SMILES")

Descriptors.MolWt(mol)          # 180.16
mol.GetNumAtoms()               # 13 — heavy atoms only; Hs are implicit
fp = AllChem.GetMorganFingerprintAsBitVect(mol, radius=2, nBits=2048)

patt = Chem.MolFromSmarts("c1ccccc1")   # SMARTS substructure query
mol.HasSubstructMatch(patt)             # True
print(Chem.MolToSmiles(mol))            # canonical SMILES
```

## Architecture / How It Works

The core object is the molecule graph (`ROMol` read-only / `RWMol` editable in C++; a single `Mol` class in Python). Parsing is two-phase: build the graph, then **sanitize** — perceive aromaticity, kekulize, check valences, assign stereochemistry. Sanitization is where chemically dubious input dies; it can be disabled (`sanitize=False`) and re-run selectively, which is the standard escape hatch for dirty vendor SDF files.

Notable subsystems:

- **Fingerprints** — Morgan (ECFP-equivalent), RDKit topological, MACCS keys, atom pairs, topological torsions. The newer `rdFingerprintGenerator` API supersedes the older per-fingerprint functions.
- **Conformers** — ETKDG, distance geometry augmented with experimental torsion preferences[^2]; ETKDGv3 improved small rings and macrocycles[^3]. Embedding can fail (returns `-1`) — production code must handle it.
- **2D depiction** — `MolDraw2D` (SVG/Cairo), with Schrödinger's open-sourced CoordGen library integrated for publication-style 2D layout.
- **Reactions** — SMARTS-based reaction transforms (`RunReactants`) for library enumeration and forward-synthesis pipelines.
- **PostgreSQL cartridge** — substructure/similarity search as SQL operators inside Postgres[^4]; how most groups ship RDKit search to non-Python users.
- **MinimalLib** — the emscripten/CFFI build (RDKit.js) exposing a curated subset in browsers; substantially narrower than the Python API.

Releases are calendar-versioned, roughly twice a year (YYYY.03 / YYYY.09), a cadence held since the 2013 move to GitHub[^5].

## Production Notes

**Silent `None` returns.** `MolFromSmiles` and friends return `None` on failure and log the reason to the C++ error stream, invisible to Python `logging`. Pipelines that skip the `None` check corrupt downstream data; capture diagnostics with `rdkit.RDLogger` or `rdBase.LogToPythonLogger()`.

**Canonical SMILES are not stable across versions.** The canonicalization algorithm changes between releases. Never use canonical SMILES as a permanent database key across RDKit upgrades — use InChI/InChIKey for identity and regenerate SMILES per-version.

**Sanitization rejects real-world files.** Pentavalent nitrogens (nitro groups drawn uncharged), odd metal valences, and aromaticity-perception mismatches are routine in vendor catalogs. Budget for a standardization pass (`rdMolStandardize`) and a quarantine path for failures rather than assuming 100% parse rates.

**Python-loop overhead dominates.** The C++ core is fast, but iterating millions of molecules from Python is bound by wrapper overhead. Use bulk APIs (`DataStructs.BulkTanimotoSimilarity`), `ForwardSDMolSupplier` / `MultithreadedSDMolSupplier` for streaming large SDF files, and multiprocessing for descriptor jobs. `PandasTools` embeds mol objects in DataFrames and bloats memory at scale.

**Serialization footgun.** Pickling a mol drops its properties by default; call `Chem.SetDefaultPickleProperties(Chem.PropertyPickleOptions.AllProps)` or persist properties separately.

**Packaging.** Conda-forge is the reference channel; official PyPI wheels (`rdkit`, replacing the community `rdkit-pypi` package around 2022) cover common platforms but exclude the PostgreSQL cartridge, and mixing conda and pip RDKit in one environment invites native-library conflicts.

**Upgrades.** Twice-yearly releases deprecate steadily (old fingerprint calls vs. generators, drawing APIs), and chemistry-level behavior — aromaticity edge cases, canonical atom ordering — can shift results between versions. Pin the version per project and re-validate cached descriptors/fingerprints on bump.

## When to Use / When Not

**Use when:**
- You need chemistry-aware processing in Python: parsing, standardization, substructure/similarity search, descriptors, depiction.
- You are featurizing molecules for ML — RDKit fingerprints and descriptors are the field's de facto baseline.
- You want substructure search inside PostgreSQL via the cartridge.
- License matters: BSD-3-Clause with no commercial tier.

**Avoid when:**
- You mainly need file-format conversion breadth — RDKit reads a handful of formats well; Open Babel reads over a hundred.
- You are on the JVM or .NET as a first-class citizen — the SWIG wrappers are functional but second-tier next to the Python API.
- You need quantum chemistry, MD simulation, or docking — RDKit stops at classical force-field cleanup; pair it with dedicated engines.
- You need guaranteed-stable canonical identifiers across tool versions without adding an InChI layer.

## Alternatives

- openbabel/openbabel — use instead when the job is file-format conversion across 100+ formats or a CLI-first workflow.
- cdk/cdk — Chemistry Development Kit; use for JVM-native cheminformatics.
- epam/Indigo — C++ toolkit with strong rendering and reaction handling; use when Java/.NET bindings are primary.
- Actelion/openchemlib — pure-Java toolkit; use when you need zero native dependencies on the JVM.
- datamol-io/datamol — not a replacement: a pythonic wrapper over RDKit itself; use when you want RDKit power with a friendlier ML-pipeline API.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | ~2000 | Development begins at Rational Discovery[^1]. |
| — | 2006-06 | Open-sourced under BSD on SourceForge[^1]. |
| 2013.09 | 2013 | Repo on GitHub; calendar versioning begins (YYYY.03 / YYYY.09)[^5]. |
| — | 2015 | ETKDG conformer generation published (Riniker & Landrum)[^2]. |
| — | 2020 | ETKDGv3: improved macrocycle and small-ring conformers[^3]. |
| — | ~2022 | Official `rdkit` wheels on PyPI supersede community `rdkit-pypi`. |
| 2026.03 | 2026 | Current release line; twice-yearly cadence continues[^5]. |

## References

[^1]: RDKit documentation, "An overview of the RDKit" (history section). https://www.rdkit.org/docs/Overview.html
[^2]: Riniker, S.; Landrum, G. A. "Better Informed Distance Geometry." J. Chem. Inf. Model. 2015. https://doi.org/10.1021/acs.jcim.5b00654
[^3]: Wang, S. et al. "Improving Conformer Generation for Small Rings and Macrocycles." J. Chem. Inf. Model. 2020. https://doi.org/10.1021/acs.jcim.0c00025
[^4]: RDKit PostgreSQL cartridge documentation. https://www.rdkit.org/docs/Cartridge.html
[^5]: RDKit release notes. https://github.com/rdkit/rdkit/releases

## Tags

c-plus-plus, python, cheminformatics, chemistry, drug-discovery, molecular-modeling, fingerprints, smiles, machine-learning, scientific-computing, postgresql-extension
