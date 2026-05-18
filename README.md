# `MED_PY` — python pipeline for prescribing data collection and processing in Genes & Health

<img src="./images/GNH%20TD%20Logo.png" alt="G&H Team Data logo" width=25%>
<!-- <img width="191" height="20" alt="image" src="https://github.com/user-attachments/assets/e5814924-f5a7-41d4-b634-e7e519a3fbfe" /> -->

## Author(s) & Contributor(s)

* Stuart Rison
* Mike Samuels
* Daniel Stow
* Sarah Finer

## Summary

The Genes & Health (G&H) `MED_PY` pipeline extracts and processes prescribing data from G&H raw phenotypic data.

At present, this is limited to prescribing data from North East London (NEL) primary care records.

Prescribed medications are extracted, Quality Controlled and de-dupilcated.  In order to failitate prescribing phenotype creation and analysis, wehere possible prescriptions are associated with a BNF code and Virtual Therapeutic Moiety (VTM).  To account for single pill combination medications (e.g. Sevikar HCT = Olmesartan + Amlodipine + Hydrochlorothiazide); prescrbing data are provided as per person/prescription (one row per pesron prescption with up to 3 VTM in columns) or as per person/VTM (one row per issue).  In the later case, Sevikar HCT would exists as three separate rows, one for Olmesartan, one for Amlodipine and one for Hydrochlorothiazide.

**MED_PY is under beta-development and this page is a place holder.  A (QMUL sandbox-1) beta-release of MED_PY data is expected in May 2026 and an official release within a month of the beta-release.**

We anticipate adding secondary care data sources in the future.

## Process

Two script manage the creation of MED_PY

1. `medication_parser`: collects, QCs and de-duplicates all issued presctions ("ord" files)
2. `medication_analyser`: links the output of `medication_parser` to the BNF code / VTM lookup table; creates heuristics based treatment length approximations; generates regenie-ready prescribing-phenotype files (BNF as phenotype, VTM as phenotype)

The above require access to a BNF_CODE <-> SNOMED_CODE <-> VTM look-up table.  This table uses the NHS Business Services Authority (NHSBSA)'s BNF SNOMED mapping resources (https://www.nhsbsa.nhs.uk/prescription-data/understanding-our-data/bnf-snomed-mapping).  This is handles by three scripts:
1. `NHSBSA_BNF_SNOMED_collector`: downloads all historical NHSBSA BNF-SNOMED mapping files
2. `NHSBSA_BNF_SNOMED_compiler`: concatenates and deduplicates downloaded BNF BSA mapping files
3. `nhsbsa_vtm_extender`: uses the compiled NHS BSA files and maximally extends the mapping potential for example by extracting putative VTMs for SNOMED_CODEs without a NHS BSA assigned VTM.

## Phenotype (prescribing) data

The pipeline imports G&H prescribing data in `.../library-red/phenotypes_rawdata/DSA__Discovery_7CCGs/`: 

./2022_04_Discovery/GNH_bhr-phase2-outfiles_merge/GNH_bhr_medications_ord_output_dataset_20220412.csv
./2022_04_Discovery/GNH_thwfnech-phase2-outfiles_merge/GNH_thwfnech_medications_ord_output_dataset_20220423.csv
./2022_12_Discovery/GNH_bhr-phase2-outfiles_merge/gh2_medications_ord_dataset_20221207.csv
./2022_12_Discovery/GNH_thwfnech-phase2-outfiles_merge/cohort_gh2_medications_ord_output_dataset_20221207.csv
./2023_01_Discovery/gh3_medication_ord.csv
./2023_03_Discovery/gh3_medication_ord.csv
./2023_11_Discovery/gh3_medication_ord.csv
./2024_07_Discovery/gh3_medication_ord.csv
./2024_12_Discovery/gh3_medication_ord.csv
./2026_04_Discovery/gh3_medication_ord.csv

## An important note re: prescribing data in G&H

The notion that `ord` files contains repeat ("ordinary") presctiptions and that `stmt` files contains acute (short term medications and treatments) is _incorrect_. The former contain any script issue to a pharmacy (i.e. prescribed by the GP) and the latter merely records that a medication was added to an individual's record (but not the issue of such a medication which is shown in "ord")

* **medications\_ord** for prescriptions issued to a pharamacy for dispensation (i.e. "ordered")  
* **medications\_stmt** to record the addition of a medication to a volunteer's medications list **\[NOT USED in MED_P\]**

## Input files

Beyond the `.ord` files listed above, the pipeline requires the extended BNF <-> SNOMED <-> VTM look-up tables

## Output files

TBC

## Locations/paths naming convention

1. We do not use relative paths.
2. We do not explicitly use the word FOLDER in the naming, so `MEGADATA_LOCATION`, not `MEGADATA_FOLDER_LOCATION`.
3. Locations and paths are in `UPPER_CASE`.
4. When referred to as `_LOCATION`, the variable contain a string with the path.
5. When referred to as `_PATH`, the variable is an `AnyPath` path object.
6. The folder order is "what it is" / "Where it's from" so, for example megadata/primary_care not primary_care/megadata; so `MEGADATA_PRIMARY_CARE_LOCATION` or `PROCESSED_DATASETS_PRIMARY_CARE_PATH`

## How the pipeline works

### Deduplicating prescribing data

The problem is that there is a huge amount of redundancy in the raw data:
* Prescriptions are duplicated between cuts
* Identical or near identical prescriptions are found in `ord` (repeat) and `stmt` (short-term medication and treatment) entries
* Different SNOMED codes for the same entity are captured by different cuts

So for one patient, there are **97 prescriptions assigned to a single date**!

**Solution**

The pragmatic solution is not to distinguish `ord` and `stmt`and to de-duplicate in stages.

#### Stage 1: De-duplicate same medication name issued to same patient on the same day

_Problem:_
<p><img src="./images/problem1.png" alt="G&H Team Data logo" width=100%>

_Solution:_
```python
.sort(by=[ pl.col.original_code.str.len_chars(), pl.col.CUT ], descending=True)
.unique(
    [
        "pseudo_nhs_number",
        "clinical_effective_date",
        "original_term",
    ]
)
```
_Explanation_

We sort on the length of the original code looking for longest SNOMED codes, this is because NHS SNOMED codes are typically longer that "international" SNOMED codes:

* 319283006 |Product containing precisely amlodipine besilate 5 milligram/1 each conventional release oral tablet (clinical drug)|
* 39732011000001102 |Product containing precisely amlodipine 5 milligram/1 each conventional release oral tablet 1 tablet tablet (clinical drug)| \[this one has children that are UK meds\]

We then pick the row from the most recent cut

_Outcome:_
<p><img src="./images/solution1.png" alt="G&H Team Data logo" width=100%>

97 rows --> 36 rows

#### Stage 2:  De-duplicate same `original_code` (SNOMED code), different `original_term`

_Problem_
<p><img src="./images/solution1.png" alt="G&H Team Data logo" width=100%>

_Solution:_
```python
.sort(by=[ pl.col.original_term.str.len_chars(), pl.col.CUT ], descending=True)
.unique(
    [
        "pseudo_nhs_number",
        "clinical_effective_date",
        "original_code",
    ]
)
```
_Explanation_

We sort on the length of the `original_term`, this is because the longer the `original_term` the more likely it is the be the fullest description.

_Outcome:_
<p><img src="./images/solution2.png" alt="G&H Team Data logo" width=100%>

36 rows --> 30 rows

#### Impact

In the case of the individual with 97 prescriptions, we go from 97 rows to 30 rows.
In the case of all individuals, we go from 35_107_228 rows to --> 23_122_567 rows --> 22_943_164 (65% of initial num rows).


<!--
## Phenotype data

The pipeline imports G&H phenotype data in `.../library-red/phenotypes_rawdata/`.  These data are from:

**DSA__Discovery_7CCGs**: Primary care data from the North East London ICS \[North East London: ~51,000 individuals with data\]

We anticipate adding secondary care data sources in the future.

## Input files

No input files at present.

## Output files

TBC
-->
