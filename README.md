# Fact-Driven Health Information Retrieval: Integrating LLMs and Knowledge Graphs to Combat Misinformation (ECIR 2025)

This repository contains the code related to the ECIR 2025 short paper
'Fact-Driven Health Information Retrieval: Integrating LLMs and Knowledge Graphs
to Combat Misinformation'. The paper deals with retrieving health-related
information from Web pages by considering both their topical relevance and their
correctness (or factuality). Topical relevance is assessed with BM25, while
correctness is verified by comparing claims extracted from documents against a
health-related Knowledge Graph (KG).

This repository contains the code related to:

- The construction of a health-related KG (HKG).
- Indexing the collections used in the paper - i.e., TREC Health Misinformation
  (HM) 2020, 2021 and 2022 -, and first-stage retrieval with BM25.
- The extraction of health-related claims from documents in the shape of RDF
  triples.
- Verifying the claims extracted from documents against the HKG, and computing a
  correctness score for each document.
- Combining the BM25 scores with the correctness scores to obtain the final
  ranking.

The repository is organized as follows:

- `./hkg/`: A Python package that implements a few functions needed to match
  terms and RDF triples with the terms and triples in the HKG.
- `./misinfo-runs/`: The retrieval runs produced for the paper.
- `./prompts/`: The prompts fed to an LLM to extract claims in the shape of RDF
  triples.
- `./queries/`: The TREC Health Misinformation topics in TSV format.
- `./run-summaries/`: The outputs of the TREC evaluation scripts for the runs in
  `./misinfo-runs/`, containing the final scores.
- `./trec-misinfo-resources/`: The official TREC resources for TREC HM 2020,
  2021, and 2022, including the topics and the evaluation scripts (source:
  <https://trec.nist.gov/data/misinfo.html>).
- Notebook/scripts `./1_1_create_graphs.ipynb` up to
  `./1_5_filter_llm-generated_abstract_triples.ipynb`: the code used to create
  the HKG.
- Scripts `./2_0_compute_english_whitelists.py` up to
  `./2_5_get_trec21-22_docs.py`: the code used to index the TREC HM collections
  and for the first-stage BM25 retrieval.
- Scripts `./3_1_doc_to_passages.py` up to `./3_5_reranking.py`: the code used
  to extract claims from documents, verify their correctness, and produce the
  final ranking.

The data related to the paper can be downloaded
[here](https://drive.google.com/file/d/1ej-Kn_K-7e4DIY0WEDlaue0sjoDQtsHF/view?usp=sharing),
including the files for the HKG and the extracted triples.

The rest of this README provides more information on how to run the code and
reproduce the experiments in the paper.

# Requirements

## Installing the `hkg` package

Clone this repository and move inside its root in a terminal. Create a virtual
environment, activate it, and install the `hkg` Python package contained in this
repository by running `pip install -e .`.

## A local copy of DBpedia

A local instance of Virtuoso Open Source loaded with the latest snapshot of the
DBpedia KG is recommended in order to avoid the rate limits of the public
DBpedia SPARQL endpoint.

```sh
git clone https://github.com/dbpedia/virtuoso-sparql-endpoint-quickstart.git
cd virtuoso-sparql-endpoint-quickstart
COLLECTION_URI=https://databus.dbpedia.org/dbpedia/collections/dbpedia-snapshot-2022-12 VIRTUOSO_ADMIN_PASSWD=YourSecretPassword docker compose up
```

For more information, see [Virtuoso SPARQL Endpoint
Quickstart](https://github.com/dbpedia/virtuoso-sparql-endpoint-quickstart).

## TREC Health Misinformation 2020, 2021 and 2022 resources

The resources for TREC HM 2020, 2021 and 2022 are included in this repository's
`./trec-misinfo-resources` folder for convenience.

In order to run the official TREC evaluation scripts, the extended version of
`trec_eval` (<https://github.com/lcschv/Trec_eval_extension>) needs to be
installed. To do so, run `make` inside the `Trec_eval_extension` directory,
which will create the `trec_eval` binary. An older version of `gcc` might be
needed to compile `trec_eval`, as otherwise running the latter might result in
segmentation faults. If this happens, install `gcc-4.8` and edit the
`Trec_eval_extension` Makefile at line 12 so that
```
CC       = gcc-4.8
```
and then run `make` inside the `Trec_eval_extension` folder.

Also download <https://github.com/trec-health-misinfo/Compatibility>, and set
the variables `trec_eval` and `compatibility` in the TREC evaluation scripts
(e.g., in `./trec-misinfo-resources/2020/scripts/run-2020-eval.sh`) accordingly.

## TREC Collections

Download the TREC Health Misinformation 2020 and 2021/22 collections (follow the
official instructions [here](https://trec-health-misinfo.github.io/)) and place
them in a `data` folder in the root of this repository. The folder structure
should be as follows:

```
data
├── ...
├── cc-news-2020
│   ├── 01
│   │   ├── CC-NEWS-20200101023937-00188.warc.wet.gz
│   │   └── ...
│   ├── 02
│   │   └── ...
│   ├── 03
│   │   └── ...
│   └── 04
│   │   └── ...
├── c4
│   └── en.noclean
│       ├── c4-train.00000-of-07168.json.gz
│       └── ...
└── ...
```


## Other files (`./data/` folder)

The outputs of the scripts (the HKG files, the LLM-generated triples, etc.) are
too large to be included in this repository. You can download a compressed
archive containing all of these files
[here](https://drive.google.com/file/d/1ej-Kn_K-7e4DIY0WEDlaue0sjoDQtsHF/view?usp=sharing)
and extract it inside the root of this repository (the size of the folder is
around 9 GB uncompressed).

The archive includes the following folders and files:

- `./data/correctness_scores/`
- `./data/graphs/`
- `./data/cc-news_en_whitelist.txt`
- `./data/main_graph_abstracts.jsonl`
- `./data/main_graph_abstracts_triples.jsonl`
- `./data/trec2020_first_stage_retrieved_docs.csv`
- `./data/trec2020_first_stage_retrieved_docs_passages.jsonl`
- `./data/trec2020_first_stage_retrieved_docs_passages_RDF_triples.jsonl`
- `./data/trec2021-22_first_stage_retrieved_docs.csv`
- `./data/trec2021-22_first_stage_retrieved_docs_passages.jsonl`
- `./data/trec2021_22_first_stage_retrieved_docs_passages_RDF_triples.jsonl`

# Reproducing the experiments

## 1 Creating the Health-related Knowledge Graph (HKG)

The HKG files are available in the downloadable `./data/` archive, inside the
`./data/graphs/` directory.

The HKG can also be constructed through the following notebooks and scripts.

### `./1_1_create_graphs.ipynb`

This notebook queries DBpedia to retrieve triples with predicate
`dbo:medication`, `dbp:prevention`, etc., and performs some basic preprocessing
and filtering. It also downloads the labels of all the diseases and drugs
present in DBpedia. The resulting graphs are the following.

- `all_diseases`: all entities of type `dbo:Disease` in DBpedia. The relevant
  predicates are: `dbo:abstract`, `owl:sameAs`, `rdf:type`, `rdfs:comment`,
  `rdfs:label`.
- `all_drugs`: all entities of type `dbo:Drug` in DBpedia. The relevant
  predicates are: `dbo:abstract`, `owl:sameAs`, `rdf:type`, `rdfs:comment`,
  `rdfs:label`.
- `main_graph`: the core HKG, including triples involving diseases and their
  treatments. The relevant predicates are: `owl:sameAs`, `rdf:type`,
  `rdfs:label`, `rdfs:comment`, `dbo:abstract`, `hkg:cause`, `hkg:complication`,
  `hkg:diagnosis`, `hkg:medication`, `hkg:prevention`, `hkg:risk`,
  `hkg:symptom`, and `hkg:treatment`.

The union of these graphs is referred to as the `full_graph`.

Outputs:

- `./data/graphs/csv/all_diseases.csv`
- `./data/graphs/csv/all_drugs.csv`
- `./data/graphs/csv/main_graph.csv`
- `./data/graphs/pickle/all_diseases.pickle`
- `./data/graphs/pickle/all_drugs.pickle`
- `./data/graphs/pickle/main_graph.pickle`
- `./data/graphs/ttl/all_diseases.ttl`
- `./data/graphs/ttl/all_drugs.ttl`
- `./data/graphs/ttl/main_graph.ttl`

### `./1_2_get_main_dbpedia_abstracts.py`

This script saves all DBpedia abstracts of the entities in the main graph as a
JSONL file.

Outputs:

- `./data/main_graph_abstracts.jsonl`

### `./1_3_compute_graph_dense_indices.py`

This script computes vector representations of the entities in the full graph
using the `abhinand/MedEmbed-small-v0.1` model and the
[`retriv`](https://github.com/AmenRa/retriv) Python library. These
representations are used to match entities in the HKG according to semantic
similarity.

Outputs:

- `~/.retriv/collections/full_graph_labels/`: the dense index of the entities in
  `full_graph`.

### `./1_4_graph_abstract_translation_to_rdf.py`

This script extracts triples from DBpedia abstracts using the model
`google/gemma-2-9b-it` and the prompt in `./prompts/prompt_abstract.txt`.

Outputs:

- `./data/main_graph_abstracts_triples.jsonl`

### `./1_5_filter_llm-generated_abstract_triples.ipynb`

The HKG is expanded by including the triples generated with the LLM from the
DBpedia abstracts, after some filtering.

Outputs:

- `./data/graphs/csv/full_graph_abstract_triples_filtered.csv`
- `./data/graphs/pickle/full_graph_abstract_triples_filtered.pickle`
- `./data/graphs/ttl/full_graph_abstract_triples_filtered.ttl`

Different versions of the HKG can now be loaded, for example using the `hkg`
package as follows:

```python
from hkg.graph_utils import load_named_graph

# The core HKG, containing entities linked by, e.g., hkg:treatment,
# hkg:medication and so on, and their labels and abstracts.
graph = load_named_graph("main_graph")

# full_graph =
# main_graph + all DBpedia entities of type dbo:Disease and dbo:Drug + their
# labels and abstracts.
graph = load_named_graph("full_graph")

# full_graph_abstract =
# full_graph + the triples extracted from DBpedia abstracts in the last step
graph = load_named_graph("full_graph_abstract")
```

Alternatively, [`pickle`](https://docs.python.org/3/library/pickle.html) can be
used to load the graphs saved in the `pickle` extension, or
[`rdflib`](https://github.com/RDFLib/rdflib) can be used to load the graphs
saved in the `ttl` format.

The loaded graph is an object of class
[`rdflib.Graph`](https://rdflib.readthedocs.io/en/stable/apidocs/rdflib.html#rdflib.graph.Graph).

## 2 Indexing the TREC collections and first-stage BM25 retrieval

### Indexing

The code used for indexing the TREC HM 2020 and 2021/22 collections with
[`pyserini`](https://github.com/castorini/pyserini/) is found in
`./2_1_index_collections.sh`. Before indexing the 2020 collection, a whitelist
of English documents is obtained in `./2_0_compute_english_whitelists.py`

### First-stage retrieval with BM25

The script `./2_2_convert_xml_topics_to_tsv.py` converts the XML TREC topics into
TSV files inside the `./queries` folder:

- `./queries/misinfo-2020-topics-description.tsv`
- `./queries/misinfo-2020-topics-title.tsv`
- `./queries/misinfo-2021-topics-description.tsv`
- `./queries/misinfo-2021-topics-query.tsv`
- `./queries/misinfo-2022-topics-query.tsv`
- `./queries/misinfo-2022-topics-question.tsv`

The first-stage retrieval with BM25 is then performed in
`./2_3_retrieval.sh`. The resulting runs are the following:

- `./misinfo-runs/adhoc/2020/run.misinfo-2020-title.bm25_en.txt`
- `./misinfo-runs/adhoc/2020/run.misinfo-2020-description.bm25_en.txt`
- `./misinfo-runs/adhoc/2021/run.misinfo-2021-query.bm25.txt`
- `./misinfo-runs/adhoc/2021/run.misinfo-2021-description.bm25.txt`
- `./misinfo-runs/adhoc/2022/run.misinfo-2022-query.bm25.txt`
- `./misinfo-runs/adhoc/2022/run.misinfo-2022-question.bm25.txt`

### Obtaining the text of the retrieved documents

The raw text of the documents in the collections is not stored in the Anserini
indices in order to save space. The scripts `./2_4_get_trec_2020_docs.py` and
`./2_5_get_trec21-22_docs.py` take care of obtaining the texts of the documents
retrieved in the first-stage runs. The texts are saved in the files
`./data/trec2020_first_stage_retrieved_docs.csv` and
`./data/trec2021-22_first_stage_retrieved_docs.csv`.

## 3 Estimating the correctness of the documents and re-ranking

### Extracting claims in the shape of RDF triples from the documents

Although this is an offline (query-independent) step, in order to save resources
the following operations were performed only on the first-stage retrieved
documents.

Passages are first extracted from the documents in the 2020 and 2021/22
collections in `./3_1_doc_to_passages.py`. The resulting files are:

- `./data/trec2020_first_stage_retrieved_docs_passages.jsonl`
- `./data/trec2021-22_first_stage_retrieved_docs_passages.jsonl`

Next, RDF triples are generated with an LLM from the document passages. This can
be done with the script `./3_3_generate_triples_from_list_of_docids.py`:

- The script takes as input a file with a list of IDs of documents from which to
  extract triples. These can be, for instance, the IDs of the top retrieved
  documents for a query, which can be computed with the script
  `./3_2_get_top_docs_per_qid.py`.
- For each document ID, the script creates a file containing the triples
  generated for that document. The location of these files can be specified as a
  command-line argument.

In this way, the script can be run multiple times in parallel.

The prompt used to generate RDF triples from documents can be found in
`./prompts/prompt_document.txt`.

The triples generated for the TREC 2020 documents are collected in
`./data/trec2020_first_stage_retrieved_docs_passages_RDF_triples.jsonl`.
The triples generated for the TREC 2021/22 documents are collected in
`./data/trec2021_22_first_stage_retrieved_docs_passages_RDF_triples.jsonl`.

### Verifying document claims

The correctness of the triples extracted from the documents is then verified
with the script `./3_4_compute_document_correctness_scores.py`.
The output files are the following:

- `./data/correctness_scores/trec2020_first_stage_retrieved_docs_passages_RDF_triples_full_graph_abstract_triple-level_correctness_scores.json`
- `./data/correctness_scores/trec2020_first_stage_retrieved_docs_passages_RDF_triples_full_graph_abstract_triple-level_correctness_scores_dense-10-match.json`
- `./data/correctness_scores/trec2021_22_first_stage_retrieved_docs_passages_RDF_triples_full_graph_abstract_triple-level_correctness_scores.json`
- `./data/correctness_scores/trec2021_22_first_stage_retrieved_docs_passages_RDF_triples_full_graph_abstract_triple-level_correctness_scores_dense-10-match.json`

All the correctness scores were computing using the version of the HKG
containing the triples extracted from DBpedia abstracts (`full_graph_abstract`).
The `dense-10-match` in the filenames refers to whether similarity-based
matching (using the dense index obtained in a previous step) was used to match
the terms in the LLM-generated triples with the terms in the HKG, with `10`
referring to the top 10 most similar terms.

### Re-ranking with respect to topicality and correctness

Finally, the documents are re-ranked based on both topicality and correctness
with the script `./3_5_reranking.py`. The resulting run files are named
according to the following template:

- `./misinfo-runs/adhoc/[year]/reranked_top500_topicality[t]_correctness[c].txt`

Where `[t]` and `[c]` refer to the topicality and correctness weights used for
re-ranking (e.g., `0.5` and `0.5` for the file
`./misinfo-runs/adhoc/2020/reranked_top500_topicality0.5_correctness0.5.txt`)

## 4 Evaluation

The results of running the TREC evaluation scripts on the runs in
`./misinfo-runs/adhoc/` are found in `./run-summaries/adhoc/`.
