# SetRank

## Introduction

This repo includes all the benchmark datasets, source code, evaluation toolkit, and experiment results for SetRank framework developed for entity-set-aware literature search. 

## Necessary Steps

### Code Fixes

```shell
pip3 install -r requirements.txt
```

```python
#/code/SetRank/autoSetRank_TREC.py
#Line 343
-  rankings = all_docno_rankings[query_id]
+  rankings = all_docno_rankings[int(query_id)-1]
#Line 403
-  if args.mode == "query": # save results only for query level aggregation
+  if args.agglevel == "query": # save results only for query level aggregation

#/code/SetRank/setRank_ESR.py    
#Line 40 and 45
-   if len(line) != 3:
+   if len(line) < 1 or len(line) > 3:
-   type = line[2]
+   type = line[2] if len(line) > 2 else "."

#/code/SetRank/setRank_TREC.py
-   parser.add_argument('-output', required=False, default="../results/trec/setrank.run",
+   parser.add_argument('-output', required=False, default="../../results/trec/setrank.run",
```

```shell
mklink /D TREC_BIO TREC-BIO
```

### Elastic Search Setup

Retrieve Elasticsearch from

```shell
https://www.elastic.co/downloads/past-releases/elasticsearch-5-4-0
```

and add the following lines to the config

```yml
#/elasticsearch-5.4.0/config/elasticsearch.yml
script.inline: true
script.ingest: true
```

then the elasticsearch engine can be started in the bin folder with either: 

```
elasticsearch
elasticsearch.bat
elasticsearch.sh
```

### Datasets

You can retrieve the datafiles from: 

```
#S2-CS
https://allenai.org/data/esr
#TREC-BIO
https://drive.google.com/file/d/1J6O9vm4wok4sEpV-1mVMWiNZ0_XlK7Xf/view
```

Download and extract the datasets and add the following datafiles to the correct folders:

```
#To /data/S2-CS/
s2.trec
s2_doc.json
s2_query.json
s2_query.manual_annotation

#To /data/TREC-BIO
trec_doc.json
```

#### Query Difference

In the datasets, only the complete search queries are provided, so you can only rerun the expirements from table 3 with them.

If you want to rerun the expirements from table 4 and table 5, you need cherrypick the ESQ from the provided query files (or you use the queryfiles provided by us). you can search for $$"1,"$$ lines in s2_query.json /trec_query.json and copy all lines which either contain 0 of these values, as the ana list should only contain 0 or 1 term.

### Creating Indexes

For each of the baseline algorithms it is necessary to create an index in elasticsearch, this can be done with the following: 

```
#baselines /code/baselines/S2-CS /code/baselines/TREC-BIO
python3 create_index.py -sim bm25
python3 index_data.py -sim bm25
#-sim options are: bm25, lm_dir, lm_jm, ib

#SetRank /code/SetRank/
#TREC-BIO
python3 create_index_TREC.py
python3 index_data_TREC.py

#S2-CS
python3 create_index_ESR.py
python3 index_data_ESR.py
```

It is necessary to generate an Index for each algorithm and dataset you want to use. Therefore you need to index the data five times (4x baselines, 1x SetRank) for each dataset, if you want to rerun the complete experiment. 

### Executing the scripts 

Baselines

```
#/code/baselines/S2-CS /code/baselines/TREC-BIO
#2 Choices,either
python3 script.py
#or
python3 search_data.py -sim bm25 -mode both
#-mode has the following options: word, entity, both
```

If you want to run script.py, it is necessary to first index all baselines. The execution of the scripts will produce files of the following format: "bm25_both.run" and will save them in the same folder the scripts are saved in.

SetRank

For this script there are several options to reexecute them, for example the following, taken by the example values from table7 by Jiaming Shen:

```
#/code/SetRank
setRank_ESR.py -query ../../data/S2-CS/s2_query.json -output ../../results/s2/setRank_tuned.run -params title:20.0,abstract:5.0,keyphrase:16.0,title_ana:20.0,abstract_ana:5.0,keyphrase_ana:16.0,bodytext_ana:1.0,title_mu:1000.0,abstract_mu:1000.0,keyphrase_mu:1000.0,title_ana_mu:1000.0,abstract_ana_mu:1000.0,keyphrase_ana_mu:1000.0,bodytext_ana_mu:1000.0,entity_lambda:0.7,type_interaction:1.0,consider_entity_set:1.0,consider_word_set:1.0,consider_type:1.0,word_dependency:1.0
#OR
python setRank_TREC.py -query ../../data/TREC-BIO/trec_query.json -output ../../results/trec/setRank_tuned.run -params title:20.0,abstract:5.0,title_ana:20.0,abstract_ana:5.0,title_mu:1000,abstract_mu:1000,title_ana_mu:1000.0,abstract_ana_mu:1000.0,entity_lambda:0.2,type_interaction:1.0,consider_entity_set:1.0,consider_word_set:1.0,consider_type:1.0,word_dependency:1.0
```

To run this script for word or entity, as mentioned in table 3, it is necessary to change the code a little bit, as in the setRank scripts it is possible to modify the fields which should be searched and you need to remove the lines containing either "entity_string" or "query_string". When you only have lines containing query_string you run "word" queries, and with only "entity_string" you are running entity queries.

```python
# /code/SetRank/setRank_ESR.py
# Lines 84-90
# Only executing "word queries"
{"match": {"title": {"query": query_string, "boost": field_weights["title"]}}},
{"match": {"abstract": {"query": query_string, "boost": field_weights["abstract"]}}},
{"match": {"keyphrase": {"query": query_string, "boost": field_weights["keyphrase"]}}},
#{"match": {"title_ana": {"query": entity_string, "boost": field_weights["title_ana"]}}},
#{"match": {"abstract_ana": {"query": entity_string, "boost": field_weights["abstract_ana"]}}},
#{"match": {"keyphrase_ana": {"query": entity_string, "boost": field_weights["keyphrase_ana"]}}},
#{"match": {"bodytext_ana": {"query": entity_string, "boost": field_weights["bodytext_ana"]}}}
```

This works similar for setRank_TREC.py.

### Training the model

To retrieve the best parameters, the paper also provides a script called "autoSetRank" which finds the best hyperparameters, with bruteforcing them.

To tune the parameters, run the following command (12h computation time):

```
python autoSetRank_TREC.py -output rankings_reproduction 
```

Then you can use the tuned results to rank the given queries:

```
python autoSetRank_TREC.py -load_pre_saved_rankings 1 -pre_saved_rankings rankings_reproduction -mode rank -query ../../data/TREC-BIO/trec_query.json -agglevel query -output filename
```

You can also use the printed parameters of autoSetRank to tune setRank as done above.

## Evaluation Tool

The **./pytrec_eval/** folder includes the original evaluation toolkit [pytrec_eval](https://github.com/cvangysel/pytrec_eval) and our customized scripts for performing model evaluation.

You may first follow the instructions in **./pytrec_eval/README.md** to install this packages and then conduct the model evaluation using following commands:

```
$ cd ./pytrec_eval/examples
$ ./eval.sh ../../results/s2/setRank.run setRank ## first argument is path to run file and the result save files
```

## Experiment Results

The **./results/** folder includes all the experiment results reported in our paper. Specifically, each file with suffix _.run_ is the model output ranking files; each file with suffix _.eval.tsv_ is the query-specific evaluation result. Notice that in the paper, we only report the NDCG@{5,10,15,20}, while here we releases the experiment results in terms of other metrics, including MAP@{5,10,15,20} and success@{1,5,10}. 


## Citation 

If you use the datasets or model implementation code produced in this paper, please refer to our SIGIR paper:

```
@inproceedings{JiamingShen2018ess,
  title={Entity Set Search of Scientific Literature: An Unsupervised Ranking Approach},
  author={Jiaming Shen, Jinfeng Xiao, Xinwei He, Jingbo Shang, Saurabh Sinha, and Jiawei Han},
  publisher={ACM},
  booktitle={SIGIR},
  year={2018},
}
```

Furthermore, if you use the pytrec_eval toolkit, please also consider citing the original paper:

```
@inproceedings{VanGysel2018pytreceval,
  title={Pytrec\_eval: An Extremely Fast Python Interface to trec\_eval},
  author={Van Gysel, Christophe and de Rijke, Maarten},
  publisher={ACM},
  booktitle={SIGIR},
  year={2018},
}
```
