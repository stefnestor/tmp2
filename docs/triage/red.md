---
hide:
  - navigation
  - toc
template: triage.html
---

!!! question "Sev1 initial email"
	> Hello team,
	> 
	> We upgraded two weeks ago from 7.15 to 7.16. Post upgrade system seemed to work as expected. 
	> However, starting yesterday, our main index pattern which loads alias `hogwarts` takes 
	> multiple seconds to load within Discover EVEN for only last 15mins or only ≤2000 hits, including 
	> during off hours. Core production application monitoring goes through this alias which never
	> loads within browser timeout which means we're flying blind and/or down! Confirmed 
	> other Kibana index patterns are not impacted.
	> 
	> Please fix ASAP!
	> 
	> Shibin

!!! abstract "summary" 

	Environment is ECH Deployment (`7.16.1`) which is `status:green`. 

	??? quote "version `7.16.1`"
		
		Elasticsearch API's "Root" aka "Get Cluster Info" endpoint ([v8](https://www.elastic.co/docs/api/doc/elasticsearch/v8/operation/operation-info), [v7](https://www.elastic.co/guide/en/elasticsearch/reference/7.16/rest-api-root.html)) which as ran from [DevTools](https://www.elastic.co/guide/en/kibana/current/console-kibana.html) would appear

		```json
		GET /
		```

		which stores into [Elasticsearch diagnostic](https://www.elastic.co/guide/en/elasticsearch/reference/current/diagnostic.html) under `version.json` ([ref](https://github.com/elastic/support-diagnostics/blob/main/src/main/resources/elastic-rest.yml#L343-L346)) and returned with

		```json
		{
		  "name" : "instance-0000000000",
		  "cluster_name" : "ecbcf72b2f6d4dfe92aac109bd6188f9",
		  "cluster_uuid" : "pakutsm-Qn6iizPqGLRQdg",
		  "version" : {
		    "number" : "7.16.1",
		    "build_flavor" : "default",
		    "build_type" : "docker",
		    "build_hash" : "5b38441b16b1ebb16a27c107a4c3865776e20c53",
		    "build_date" : "2021-12-11T00:29:38.865893768Z",
		    "build_snapshot" : false,
		    "lucene_version" : "8.10.1",
		    "minimum_wire_compatibility_version" : "6.8.0",
		    "minimum_index_compatibility_version" : "6.0.0-beta1"
		  },
		  "tagline" : "You Know, for Search"
		}
		```

		Third-party tool [JQ](https://jqlang.github.io/jq/) which does JSON preprocessing allows you to simplify these results to the version number

		```bash
		$ cat version.json | jq -r '.version.number'
		7.16.1
		```

	??? quote "status `green`"

		Cluster Health Status ([v8](https://www.elastic.co/docs/api/doc/elasticsearch/v8/operation/operation-cluster-health), [v7](https://www.elastic.co/guide/en/elasticsearch/reference/7.16/cluster-health.html)) would appear
		
		```json
		GET _cluster/health
		```

		which stores into Elasticsearch diagnostic under `cluster_health.json` ([ref](https://github.com/elastic/support-diagnostics/blob/main/src/main/resources/elastic-rest.yml#L166-L168)) and returned with 

		```json
		{
		  "cluster_name": "ecbcf72b2f6d4dfe92aac109bd6188f9",
		  "status": "green",
		  "timed_out": false,
		  "number_of_nodes": 3,
		  "number_of_data_nodes": 2,
		  "active_primary_shards": 17,
		  "active_shards": 34,
		  "relocating_shards": 0,
		  "initializing_shards": 0,
		  "unassigned_shards": 0,
		  "delayed_unassigned_shards": 0,
		  "number_of_pending_tasks": 0,
		  "number_of_in_flight_fetch": 0,
		  "task_max_waiting_in_queue_millis": 0,
		  "active_shards_percent_as_number": 100
		}
		```

		!!! example "filter result to `status`"
		
			If querying live against the cluster, admins can filter-in to `status` with Elasticsearch API's `filter_path` ([ref](https://www.elastic.co/guide/en/elasticsearch/reference/current/common-options.html#common-options-response-filtering))

			```json
			GET _cluster/health?filter_path=status
			```

			From an Elasticsearch diagnostic, you can reproduce this with third-party tool [JQ](https://jqlang.github.io/jq/) which does JSON preprocessing.

			```bash
			$ cat cluster_health.json | jq -rc '{status}'
			{"status":"green"}
			```
			

		The [Health (Report)](https://www.elastic.co/guide/en/elasticsearch/reference/current/health-api.html) did not yet exist so the Elasticsearch diagnostic will not include `internal_health.json` ([ref](https://github.com/elastic/support-diagnostics/blob/main/src/main/resources/elastic-rest.yml#L228-L235)). 

	A [browser network HAR log](https://www.elastic.co/blog/generating-browser-har-file-kibana-troubleshooting) did not emit anything obviously helpful. Kibana [Troubleshooting Discover Load](https://www.elastic.co/blog/troubleshooting-guide-common-issues-kibana-discover-load#3.-load-search) confirms that Elasticsearch's "Query time" for last 15min for `hogwarts` is ~6s and total time is ~6.5s. 
