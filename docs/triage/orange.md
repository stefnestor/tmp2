---
hide:
  - navigation
  - toc
template: triage.html
---


!!! abstract "summary" 
    
    Resource utilization appears nominal throughout, only fluctuating with daily traffic volume.

    !!! quote ""

        (Opt) Both Elasticsearch and Kibana's Proxy Log response times suggest no significant change in overall traffic nor response times for the last month. However, there are now low volume Kibana `bsearch` requests taking +5s. Elasticsearch _maybe_ has time-correlative `_msearch` spikes but it's hard to guarantee if those are related.

    ??? quote "nominal utilization"

        !!! example "CAT alternatives"

            The Elasticsearch API pages for CAT ("compact and aligned text") prefix with warning

            > ⚠️ CAT APIs are only intended for human consumption using the command line or Kibana console. They are not intended for use by applications. For application consumption, use ...

            For mapping to backing JSON-response API's, see [Elasticsearch CAT Alternatives](https://medium.com/@stefnestor/elasticsearch-cat-alternatives-315f72ea6d5e). 

            Note that since Elasticsearch diagnostics are not stateful (aka they pull API's in quick succession but do not capture the singular state of all of the the databases' APIs all at once), then results between CAT APIs and sister JSON-response APIs will not be exact.


        CAT Nodes ([v8](https://www.elastic.co/docs/api/doc/elasticsearch/v8/operation/operation-cat-nodes), [v7](https://www.elastic.co/guide/en/elasticsearch/reference/7.16/cat-nodes.html)) would appear

        ```
        GET _cat/nodes?v&h=n,nodeId,i,v,role,m,d,dup,hp,cpu,load_1m,load_5m,load_15m,iic,sfc,sqc,scc&s=n
        ```

        which stores into Elasticsearch diagnostic under `./cat/cat_nodes.txt` ([ref](https://github.com/elastic/support-diagnostics/blob/main/src/main/resources/elastic-rest.yml#L71-L76)) and returned with

        ```txt
        n                     nodeId i            v      role   m      d  dup hp cpu load_1m load_5m load_15m iic sfc sqc scc
        tiebreaker-0000000002 e6Cw   10.42.10.175 7.16.1 mv     - 44.9gb 0.00 55   0    1.98    1.60     1.28   0   0   0   0
        instance-0000000000   fjnZ   10.42.0.53   7.16.1 himrst * 89.8gb 0.18 33   0    1.04    1.31     1.17   0   0   0   0
        instance-0000000001   VcYo   10.42.13.38  7.16.1 himrst - 89.8gb 0.19 20   2    0.95    1.45     1.66   0   0   0   0
        ```

        !!! example ""

            CAT Nodes compiles from 
              Node Info ([v8](https://www.elastic.co/docs/api/doc/elasticsearch/v8/operation/operation-nodes-info), [v7](https://www.elastic.co/guide/en/elasticsearch/reference/7.16/cluster-nodes-info.html), [ref](https://github.com/elastic/support-diagnostics/blob/main/src/main/resources/elastic-rest.yml#L259-L263))
            , Cluster State ([v8](https://www.elastic.co/docs/api/doc/elasticsearch/v8/operation/operation-cluster-state), [v7](https://www.elastic.co/guide/en/elasticsearch/reference/7.16/cluster-state.html), [ref](https://github.com/elastic/support-diagnostics/blob/main/src/main/resources/elastic-rest.yml#L183-L186))
            , and Node Stats ([v8](https://www.elastic.co/docs/api/doc/elasticsearch/v8/operation/operation-nodes-stats), [v7](https://www.elastic.co/guide/en/elasticsearch/reference/7.16/cluster-nodes-stats.html), [ref](https://github.com/elastic/support-diagnostics/blob/main/src/main/resources/elastic-rest.yml#L269-L273)).

            ```bash
            $ jq -rc '.nodes|input.master_node as $master|input.nodes as $ns|to_entries[]|{id:.key, ip:.value.ip, host:.value.host, name:.value.name, roles:.value.roles, is_master:(if .key == $master then true else false end), disk_avail:$ns[.key].fs.total.free, heap_percent:$ns[.key].jvm.mem.heap_used_percent, cpu:$ns[.key].os.cpu.percent, loads:$ns[.key].os.cpu.load_average}' nodes.json cluster_state.json nodes_stats.json
            ```

            which emits

            ```json
            {"id":"e6CwioOYT2mpLkUaugPLaQ","ip":"10.42.10.175","host":"10.42.10.175","name":"tiebreaker-0000000002","roles":["master","voting_only"],"is_master":false,"disk_avail":"44.9gb","heap_percent":55,"cpu":0,"loads":{"15m":1.28,"1m":1.98,"5m":1.60}}
            {"id":"fjnZjXPbTWa7jRsNY5YggA","ip":"10.42.0.53","host":"10.42.0.53","name":"instance-0000000000","roles":["data_content","data_hot","ingest","master","remote_cluster_client","transform"],"is_master":true,"disk_avail":"89.8gb","heap_percent":33,"cpu":0,"loads":{"15m":1.17,"1m":1.04,"5m":0.31}}
            {"id":"VcYoXNv5T92qsQRyN7hIhA","ip":"10.42.13.38","host":"10.42.13.38","name":"instance-0000000001","roles":["data_content","data_hot","ingest","master","remote_cluster_client","transform"],"is_master":false,"disk_avail":"89.8gb","heap_percent":20,"cpu":2,"loads":{"15m":1.66,"1m":0.95,"5m":1.45}}
            ```


        This emits some disk statistics, but to expand further use CAT Allocation ([v8](https://www.elastic.co/docs/api/doc/elasticsearch/v8/operation/operation-cat-allocation), [v7](https://www.elastic.co/guide/en/elasticsearch/reference/7.16/cat-allocation.html)) would appear

        ```json
        GET _cat/allocation?v
        ```

        which stores into Elasticsearch diagnostic under `./cat/cat_allocation.json` ([ref](https://github.com/elastic/support-diagnostics/blob/main/src/main/resources/elastic-rest.yml#L26-L30)) and returned with 

        ```txt
        shards disk.indices disk.used disk.avail disk.total disk.percent host        ip          node
            17       89.5mb   175.3mb     89.8gb       90gb            0 10.42.13.38 10.42.13.38 instance-0000000001
            17       88.4mb   168.7mb     89.8gb       90gb            0 10.42.0.53  10.42.0.53  instance-0000000000
        ```

        !!! example ""

            CAT Allocation compiles from Node Stats ([v8](https://www.elastic.co/docs/api/doc/elasticsearch/v8/operation/operation-nodes-stats), [v7](https://www.elastic.co/guide/en/elasticsearch/reference/7.16/cluster-nodes-stats.html), [ref](https://github.com/elastic/support-diagnostics/blob/main/src/main/resources/elastic-rest.yml#L269-L273)).

            ```bash
            $ cat nodes_stats.json | jq -rc '.nodes[]|{shards:.indices.shard_stats.total_count,disk_indices:.indices.store.size,disk_avail:.fs.total.free, disk_avail_bytes: .fs.total.free_in_bytes, disk_total:.fs.total.total, disk_total_bytes: .fs.total.total_in_bytes, host:.host, ip:.ip, node:.name}|.+{disk_used_bytes: (.disk_total_bytes-.disk_avail_bytes)}|.+{disk_percent: (.disk_used_bytes/.disk_total_bytes|round)}'
            ```

            which emits

            ```json
            {"shards":17,"disk_indices":"89.5mb","disk_avail":"89.8gb","disk_avail_bytes":96452792320,"disk_total":"90gb","disk_total_bytes":96636764160,"host":"10.42.13.38","ip":"10.42.13.38:19778","node":"instance-0000000001","disk_used_bytes":183971840,"disk_percent":0}           
            {"shards":17,"disk_indices":"88.4mb","disk_avail":"89.8gb","disk_avail_bytes":96459583488,"disk_total":"90gb","disk_total_bytes":96636764160,"host":"10.42.0.53","ip":"10.42.0.53:19058","node":"instance-0000000000","disk_used_bytes":177180672,"disk_percent":0}
            {"shards":0,"disk_indices":"0b","disk_avail":"44.9gb","disk_avail_bytes":48317509632,"disk_total":"45gb","disk_total_bytes":48318382080,"host":"10.42.10.175","ip":"10.42.10.175:19881","node":"tiebreaker-0000000002","disk_used_bytes":872448,"disk_percent":0}
            ```

            This did include `tiebreaker-0000000002` which does not normally host data as the CAT Allocation API filters to [data nodes](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html). 


    🚧 Suspected but never caught Node Hot Threads showing problem BUT when reproducing IF find then include [threads streamlit parser](https://github.com/elastic/support/tree/master/agentTools/streamlit).
    
