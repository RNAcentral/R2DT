# Usage

Specify the input file in [FASTA format](https://en.wikipedia.org/wiki/FASTA_format) containing one or more RNA sequences as well as the path where the output files will be created (the folder will be created if it does not exist).

```
r2dt.py draw <input.fasta> <output_folder>
```

For example:

```
r2dt.py draw examples/examples.fasta temp/examples
```

R2DT will automatically select the best matching template and visualise the secondary structures.

## Output files

`r2dt.py draw` produces a folder called `results` with the following subfolders:

- `svg`: RNA secondary structure diagrams in SVG format
- `fasta`: input sequences and their secondary structure in dot-bracket notation
- `tsv`: a file `metadata.tsv` listing sequence ids, matching templates, and template sources
- `thumbnail`: secondary structure diagrams displayed as outlines in SVG format
- `json`: RNA secondary structure and its layout described using [RNA 2D JSON Schema](https://github.com/LDWLab/RNA2D-data-schema/)

## Advanced options

### Skipping ribovore filters

In some cases R2DT may not generate a diagram for a sequence because [ribovore](https://github.com/ncbi/ribovore) detects one or more [unexpected features](https://github.com/ncbi/ribovore/blob/master/documentation/ribotyper.md#unexpectedfeatures), such as having hits on both strands or having too many hits in the same sequence. You can use `--skip_ribovore_filters` to ignore these warnings and attempt to generate a secondary structure diagram anyway.

For example, the following command will produce no results because the sequence is close to a palindrome:

```
r2dt.py draw examples/ribovore-qc-example.fasta temp/examples
```

However, the following command generates a [valid diagram](./tests/examples/skip-ribovore-filters/URS0000001EB3-RF00661.colored.svg):

```
r2dt.py draw --skip_ribovore_filters examples/ribovore-filters.fasta temp/examples
```

Please note that this option should be used with caution as sequences with unexpected features often result in poor diagrams.

### Manually selecting template category

If the RNA type of the input sequences is known in advance, it is possible to bypass the classification steps and achieve faster performance.

* CRW templates (5S and SSU rRNA)
    ```
    r2dt.py crw draw examples/crw-examples.fasta temp/crw-examples
    ```

* RiboVision LSU and SSU rRNA templates
    ```
    r2dt.py ribovision draw_lsu examples/lsu-examples.fasta temp/lsu-examples
    r2dt.py ribovision draw_ssu examples/ribovision-ssu-examples.fasta temp/ssu-examples
    ```

* Rfam families
    ```
    r2dt.py rfam draw RF00162 examples/RF00162.example.fasta temp/rfam-example
    ```

* RNAse P
    ```
    r2dt.py rnasep draw examples/rnasep.fasta temp/rnasep-example
    ```

* tRNAs (using GtRNAdb templates)
    ```
    # for tRNAs, provide domain and isotype (if known), or use tRNAScan-SE to classify
    r2dt.py gtrnadb draw examples/gtrnadb.E_Thr.fasta temp/gtrnadb
    r2dt.py gtrnadb draw examples/gtrnadb.E_Thr.fasta temp/gtrnadb --domain E --isotype Thr
    ```

### Manually selecting templates

It is possible to select a specific template and skip the classification step altogether.

1. Get a list of all available templates and copy the template id:
    ```
    r2dt.py list-models
    ```

In addition, all models are listed in the file [models.json](./data/models.json).

2. Specify the template (for example, `RNAseP_a_P_furiosus_JB`):
    ```
    r2dt.py draw --force_template <template_id> <input_fasta> <output_folder>
    ```

    For example:

    ```
    r2dt.py draw --force_template RNAseP_a_P_furiosus_JB examples/force/URS0001BC2932_272844.fasta temp/example
    ```
### Constraint-based folding for insertions

If a structure contains insertions that are not present in the R2DT template files, the --constraint flag will allow their folding to be de-novo predicted using the RNAfold algorithm.

There are currently three constraint folding modes available. R2DT will automatically predict which folding mode is best for a given molecule, but the mode can also be manually overridden using the --fold_type parameter. There are three options for fold_type.

* Let R2DT pick a fold_type
    ```
    r2dt.py draw --constraint <input_fasta> <output_folder>
    ```
* Fold insertions (along with adjacent unpaired nucleotides) one at a time. Recommended for large RNAs.
    ```
    r2dt.py draw --constraint --fold_type insertions_only <input_fasta> <output_folder>
    ```
* Run entire molecule through RNAfold at once. Base pairs predicted from the template are used as constraints for prediction.
    ```
    r2dt.py draw --constraint --fold_type full_molecule <input_fasta> <output_folder>
    ```
* Run entire molecule through RNAfold at once. Both conserved single-stranded regions and base pairs predicted from the template are used as constraints for prediction.
    ```
    r2dt.py draw --constraint --fold_type all_constraints_enforced <input_fasta> <output_folder>
    ```
* Prevent certain nucleotides from base pairing. This will only work for base pairs that are de-novo predicted.
The exclusion file should contain a string the same length as the input sequence composed of '.'s and 'x's. Positions with '.'s are allowed to base pair,
positions with 'x's are not.
Example string: 'xxxx..............xx..............x............xx'
    ```
    r2dt.py draw --constraint --exclusion <exclusion_file> <input_fasta> <output_folder>
    ```

### Other useful commands

* Print R2DT version
    ```
    r2dt.py version
    ```

* Run [all tests](./tests/tests.py)
    ```
    r2dt.py test
    ```

    or

    ```
    python3 -m unittest
    ```

* Run a [single test](./tests/tests.py)
    ```
    python3 -m unittest tests.tests.TestRibovisionLSU
    r2dt.py test TestRibovisionLSU
    ```

* Run tests and keep the results (useful when updating Traveler for example)
    ```
    R2DT_KEEP_TEST_RESULTS=1 r2dt.py test
    R2DT_KEEP_TEST_RESULTS=1 python3 -m unittest tests.tests.TestRnasep
    r2dt.py test TestRnasep
    ```

* Classify example sequences using Ribotyper
    ```
    perl /rna/ribovore/ribotyper.pl -i data/cms/crw/modelinfo.txt -f examples/pdb.fasta temp/ribotyper-test
    ```

* Generate covariance models and modelinfo files
    ```
    python3 utils/generate_cm_library.py
    r2dt.py generatemodelinfo <path to covariance models>
    ```

* Precompute template library locally (may take several hours):
    ```
    r2dt.py setup
    ```

* Run R2DT with Singularity
    ```
    singularity exec --bind <path_to_cms>:/rna/r2dt/data/cms r2dt r2dt.py draw sequence.fasta output
    ```

* Convert a SVG diagram to a JSON file containing the paths per nucleotide and an ordinal numbering. Note that this *assumes* that the input pdb id is formatted like: `<PDB>_<Entity>_<chain>`, ie `1S72_1_0`.
    ```
    svg2json.py <pdb-id> diagram.svg <pdb-id>.json
    ```