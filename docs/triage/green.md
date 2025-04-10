---
hide:
  - navigation
  - toc
template: triage.html
---

This was 01840022 from 2025Mar. (Issue doesn't recreate on v8.17 so spun up v7.16)

!!! example "go further"
    
    Show slow logs, query profiler, query profiler TLDR.

???+ info "recreate"

    1. Create v7.16.1 cluster 
    1. DevTools to create alias
        ```json
        PUT _ilm/policy/cows
        { "policy": { "phases": {
        "hot": {
          "min_age": "0ms",
          "actions": {
            "rollover": { "max_docs": "500" },
            "forcemerge" : { "max_num_segments": 1 }
        }}
        }}}

        # setup
        PUT _index_template/superman
        { "index_patterns": ["hogwarts*"],
        "template": {  "settings": {
            "index.lifecycle.name": "cows",
            "index.lifecycle.rollover_alias": "hogwarts",
            "number_of_shards": 1,
            "number_of_replicas": 1
        }}}

        PUT hogwarts-000001
        { "aliases": { "hogwarts": { "is_write_index": true }}}
        ```
    2. Python script to load Harry Potter `hogwarts`-related data
        ```python
        from datetime import datetime
        from elasticsearch import Elasticsearch
        from elasticsearch import helpers as esh
        import json
        import lorem
        import requests
        import sys
        import random

        # v7.16
        elastic = Elasticsearch(
            "https://v716.es.us-central1.gcp.cloud.es.io"
            ,api_key="XXXXX=="
            )

        # freepublicapis > HP > https://www.freepublicapis.com/harry-potter-api 
        spells = json.loads( requests.get("https://hp-api.onrender.com/api/spells").content )
        people = json.loads( requests.get("https://hp-api.onrender.com/api/characters").content )

        def generate_docs(p,s,l):
            # https://elasticsearch-py.readthedocs.io/en/latest/quickstart.html
            for i in range(0,100):
                yield {
                     "name": p["name"]
                    ,"nicknames": p["alternate_names"]
                    ,"species": p["species"]
                    ,"gender": p["gender"]
                    ,"house": p["house"]
                    ,"patronus": p["patronus"]
                    ,"image": p["image"]

                    ,"spell": s["name"]
                    ,"spell_description": s["description"]
                    
                    ,"message": f"{l} "*1000
                    ,"@timestamp": datetime.now().astimezone()
                    ,"ingested_at": datetime.now().astimezone()
                    ,"rando": i

                    ,"_index": "hogwarts"
                }

        l = lorem.paragraph()
        path = random.choice([1,2,3,4])
        print(f"👋 {path}")
        if path==1:
            for p in people:
                for s in spells:
                    print(f"{p["name"]} is casting {s["name"]}")
                    print( esh.bulk(elastic, generate_docs(p,s,l) ) )
        elif path==2:
            for p in people:
                for s in reversed(spells):
                    print(f"{p["name"]} is casting {s["name"]}")
                    print( esh.bulk(elastic, generate_docs(p,s,l) ) )
        elif path==3:
            for s in reversed(spells):
                for p in people:
                    print(f"{p["name"]} is casting {s["name"]}")
                    print( esh.bulk(elastic, generate_docs(p,s,l) ) )
        elif path==4:
            for s in reversed(spells):
                for p in reversed(people):
                    print(f"{p["name"]} is casting {s["name"]}")
                    print( esh.bulk(elastic, generate_docs(p,s,l) ) )
        ```
    3. Create two Data Views for `hogwarts` (without runtime mapping) & `hogwarts*` (with)
        ```bash
        # https://www.elastic.co/guide/en/kibana/7.17/index-patterns-api-create.html 
        curl -X POST -u "elastic:XXXXX" https://v716.kb.us-central1.gcp.cloud.es.io/api/index_patterns/index_pattern -d '{"index_pattern": {"title": "hogwarts", "timeFieldName": "@timestamp" } }' -H "kbn-xsrf: true" -H "content-type:application/json"

        curl -X POST -u "elastic:XXXXXX" https://v716.kb.us-central1.gcp.cloud.es.io/api/index_patterns/index_pattern -d '{"index_pattern": {"title": "hogwarts*", "timeFieldName": "@timestamp", "runtimeFieldMap": {"@timestamp": {"type": "date" } } } }' -H "kbn-xsrf: true" -H "content-type:application/json"
        ```
    4. Load Discover and see load duraton difference.
