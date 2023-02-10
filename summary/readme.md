## Summary

This procedure produces a CSV that documents which checklist (COL or otherwise) provided the taxon to which the occurrence record was organised in the GBIF backbone.
The intention is to help identify significant gaps in the COL for the purpose of organising GBIF data.

From the GBIF checklistbank, an export is made:

```
\copy (
  SELECT 
    u.id, 
    u.rank, 
    case when d.publisher=plaziKey() then plaziKey() else u.constituent_key end as source_key, 
    case when u.constituent_key=colKey() then srcu.constituent_key else null end as col_source_key,  
    d.title as source, 
    gsd.title as col_source
  FROM 
    name_usage u 
    LEFT JOIN dataset d ON d.key=u.constituent_key 
    LEFT JOIN name_usage srcu ON srcu.id=u.source_taxon_key 
    LEFT JOIN dataset gsd ON gsd.key=srcu.constituent_key 
  WHERE 
    u.dataset_key=nubKey() AND u.deleted IS NULL) to 'backbone_sources.tsv'
```

In Hive, we create ane load the table:

```
CREATE TABLE IF NOT EXISTS tim.backbone_source (
 id INT,
 rank STRING,
 sourceKey STRING,
 colSourceKey STRING,
 title STRING,
 colSource STRING)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t';

LOAD DATA LOCAL INPATH 'backbone_sources.tsv' INTO TABLE tim.backbone_source;
```

