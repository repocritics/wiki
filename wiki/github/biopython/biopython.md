# biopython/biopython

> Python library of parsers, data structures, and algorithm wrappers for computational molecular biology.

[GitHub repo](https://github.com/biopython/biopython) ·
[Official website](https://biopython.org/) ·
License: Biopython License Agreement / BSD-3-Clause (dual)[^1]

## Overview

Biopython is a collection of Python tools for computational molecular biology and bioinformatics, maintained by an international volunteer association since 1999[^2]. The repository was converted from CVS and has been on GitHub since 2009. It is one of the oldest still-active scientific Python libraries, predating NumPy's 1.0 and the modern PyData stack, and it shows: the codebase mixes decades of API conventions, from CamelCase class methods to more recent PEP 8 additions.

The library's center of gravity is *parsing and representing* biological data — sequences (`Bio.SeqIO`), alignments (`Bio.Align`), phylogenetic trees (`Bio.Phylo`), 3D macromolecular structures (`Bio.PDB`), and the file formats that carry them (FASTA, GenBank, FASTQ, PDB/mmCIF, Newick, Stockholm, and dozens more). It also wraps external command-line tools and remote databases (NCBI Entrez, BLAST) rather than reimplementing them. This makes Biopython a broad, shallow toolkit: it touches almost every bioinformatics task but is rarely the fastest or most specialized option for any single one.

The defining tension is age versus stability. Biopython is trusted precisely because it changes slowly and preserves backward compatibility, but that same conservatism means the API carries deprecated corners, inconsistent naming, and design decisions from an era before type hints, dataclasses, or the current NumPy. Users get a dependable, citable[^3] foundation at the cost of ergonomics that feel dated next to newer libraries.

## Getting Started

```bash
pip install biopython
```

Pre-compiled binary wheels have been published on PyPI since release 1.70, so installation normally needs no C compiler[^2]. NumPy is the only required runtime dependency.

```python
from Bio import SeqIO
from Bio.Seq import Seq

# Parse a FASTA file and work with sequences
for record in SeqIO.parse("sequences.fasta", "fasta"):
    print(record.id, len(record.seq))
    print(record.seq.reverse_complement())

# Build a sequence directly and translate DNA -> protein
dna = Seq("ATGGCCATTGTAATGGGCCGCTGAAAGGGTGCCCGATAG")
print(dna.translate())        # MAIVMGR*KGAR*
```

Fetching records from NCBI requires identifying yourself (Entrez enforces this):

```python
from Bio import Entrez, SeqIO

Entrez.email = "you@example.com"   # required by NCBI usage policy
with Entrez.efetch(db="nucleotide", id="NM_000518",
                   rettype="gb", retmode="text") as handle:
    record = SeqIO.read(handle, "genbank")
print(record.description)
```

## Architecture / How It Works

Biopython is a single top-level package, `Bio`, with a large set of loosely coupled submodules. There is no central engine; each subpackage is largely independent and can be imported in isolation.

- **`Bio.Seq` / `Bio.SeqRecord`** — the core value types. `Seq` is a string-like object with biological operations (complement, translate, transcribe); `SeqRecord` wraps a `Seq` with an id, description, and annotations. Nearly every other module produces or consumes these.
- **`Bio.SeqIO` / `Bio.AlignIO`** — the parser dispatch layer. Both expose a uniform `parse()` / `read()` / `write()` interface over a registry of per-format reader and writer classes. Adding a format means adding a module, not changing the interface. This is the most-used and most-polished part of the library.
- **`Bio.Align`** — alignment representations plus the `PairwiseAligner`, a pure-Python/C implementation of Needleman–Wunsch and Smith–Waterman that replaced the long-deprecated `Bio.pairwise2`.
- **`Bio.PDB`** — a `Structure → Model → Chain → Residue → Atom` (SMCRA) object hierarchy for macromolecular structures, with parsers for legacy PDB and mmCIF formats.
- **`Bio.Phylo`** — tree objects and I/O for Newick, Nexus, phyloXML, NeXML.
- **`Bio.Entrez` / `Bio.Blast`** — HTTP clients for NCBI web services, returning file handles that the I/O modules parse.

Performance-critical inner loops (pairwise alignment, some cluster and structure math) are implemented as C extension modules compiled at build time; the rest is pure Python. NumPy underpins the numeric arrays in `Bio.PDB`, `Bio.Align`, and `Bio.motifs`.

Because subpackages evolved independently over 25 years, conventions differ between them: some use CamelCase methods, some snake_case; some raise exceptions, some return `None`; parser strictness varies. The uniform façade of `SeqIO` hides this well, but reaching into a less-traveled module (`Bio.SCOP`, `Bio.KEGG`, `Bio.PopGen`) exposes the underlying unevenness.

## Production Notes

**It is a library, not a pipeline framework.** Biopython gives you parsers and data structures; it does not manage workflows, provenance, or parallelism. Large-scale analyses are built by composing Biopython with a workflow engine (Snakemake, Nextflow) and often faster specialized tools.

**Format parsers are fragile by nature.** The maintainers explicitly warn that text formats like BLAST and GenBank change upstream and break parsing; parser fixes sometimes land in git ahead of releases[^2]. Pin your Biopython version and test against your actual data files rather than assuming a format "just works."

**Deprecations are real and documented.** The `DEPRECATED.rst` file tracks removed and discouraged modules[^4]. Notable removals over the years include the old application command-line wrappers (`Bio.Application` and the `Bio.Align.Applications` / `Bio.Blast.Applications` wrappers), which were deprecated in favor of calling tools via `subprocess` directly. Code written against tutorials older than a few years will hit `BiopythonDeprecationWarning` or outright `ImportError`. Read `NEWS.rst` before upgrading across minor versions.

**Experimental code ships in stable releases.** Since 1.61, beta-quality modules emit `BiopythonExperimentalWarning`. These APIs can change without the usual compatibility guarantees — do not build long-lived pipelines on warning-flagged modules.

**Entrez etiquette matters.** NCBI rate-limits and can block clients that omit `Entrez.email` or exceed request rates. For heavy querying, set `Entrez.api_key` and batch requests; scraping NCBI through Biopython without throttling risks an IP ban.

**Python version churn.** Biopython tracks CPython aggressively — recent releases require Python 3.10+ and test against 3.14 and PyPy3[^2]. Long-lived environments pinned to old Python will be stuck on old Biopython.

## When to Use / When Not

**Use when:**
- You need to read or write bioinformatics file formats reliably (FASTA, GenBank, FASTQ, PDB, mmCIF, Newick, Stockholm).
- You want a single, well-cited dependency covering sequences, alignments, structures, and phylogenetics for scripts and teaching.
- You are gluing together NCBI queries, parsing, and light manipulation in Python.

**Avoid when:**
- You need maximum throughput on large datasets — dedicated tools (samtools, minimap2, MMseqs2) or `pysam` for BAM/VCF are faster.
- You work primarily with high-throughput sequencing formats (BAM, CRAM, VCF, BED) — Biopython's coverage there is thin; reach for pysam, pyfaidx, or scikit-bio.
- You want a modern, fully type-hinted, consistent API — the age of the codebase is felt directly.

## Alternatives

- pysam/pysam — Python bindings to htslib; the standard for BAM/CRAM/VCF/tabix work Biopython does not cover well.
- biocore/scikit-bio — newer, NumPy/pandas-native primitives for sequences and diversity stats; smaller format coverage.
- BioJulia/BioSequences.jl — use when you want Biopython-style capabilities with Julia's performance for heavy sequence work.
- open2c/bioframe — use for genomic-interval (BED-style) operations expressed as pandas DataFrames.
- rust-bio/rust-bio — use when parsing/algorithm performance is the bottleneck and you can drop to Rust.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.00 | 2000-10 | First public release of the Biopython Project[^2]. |
| 1.43 | 2007-03 | `Bio.SeqIO` introduced as the unified sequence I/O interface. |
| 1.54 | 2010-05 | `Bio.Phylo` added for phylogenetics. |
| 1.70 | 2017-07 | Pre-compiled PyPI wheels for Linux/macOS/Windows[^2]. |
| 1.74 | 2019-07 | `Bio.Align.PairwiseAligner` matured; `pairwise2` on the way out. |
| 1.78 | 2020-09 | Alphabet system removed — a long-signposted breaking change. |
| 1.80 | 2022-11 | Legacy `Bio.Application` command-line wrappers deprecated. |
| 1.85 | 2025 | Continued Python 3.13/3.14 support, ongoing deprecation cleanup[^4]. |

## References

[^1]: The repository ships the Biopython License Agreement with newer contributions dual-licensed under the 3-Clause BSD License; GitHub's SPDX detector reports the license as non-standard (NOASSERTION). https://github.com/biopython/biopython/blob/master/LICENSE.rst
[^2]: Biopython README and project website. https://github.com/biopython/biopython/blob/master/README.rst · https://biopython.org/
[^3]: Cock, P.J.A. et al. "Biopython: freely available Python tools for computational molecular biology and bioinformatics." Bioinformatics 25(11):1422–3, 2009. https://doi.org/10.1093/bioinformatics/btp163
[^4]: DEPRECATED.rst and NEWS.rst track removed modules and per-release API breakages. https://github.com/biopython/biopython/blob/master/DEPRECATED.rst · https://github.com/biopython/biopython/blob/master/NEWS.rst

## Tags

python, bioinformatics, genomics, sequence-analysis, phylogenetics, protein-structure, file-parsers, ncbi, scientific-computing, library
