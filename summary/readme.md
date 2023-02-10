## Summary

This procedure produces a CSV that documents which checklist (COL or otherwise) provided the taxon to which the occurrence record was organised in the GBIF backbone.
The intention is to help identify significant gaps in the COL for the purpose of organising GBIF data.

#### Step 1: Load the backbone into Hive

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

In Hive, we create and load the table:

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

#### Step 2: Process the origin of occurrence data

Now we process the occurrence data into the master summary view which holds the 
classification, the origin of the accepted name and synonym if applicable along with 
the number of occurrences / datasets having the classification.

This process only looks for records that have been idenfitied to a species or more precise, ignoring records that match to higher taxa only. 

```
CREATE TABLE tim.taxa_summary ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' AS
SELECT
  o.taxonKey AS taxonID,
  name.rank AS taxonRank,
  o.acceptedTaxonKey AS acceptedTaxonID,
  accepted.rank AS acceptedTaxonRank,
  o.kingdom,
  o.family,
  o.scientificName,
  o.acceptedScientificName,
  name.sourceKey AS nameSourceKey,
  name.colSourceKey AS nameColSourceKey,
  name.title AS nameSource,
  name.colSource AS nameColSource,
  accepted.sourceKey AS acceptedNameSourceKey,
  accepted.colSourceKey AS acceptedNameColSourceKey,
  accepted.title AS acceptedNameSource,
  accepted.colSource AS acceptedNameColSource,
  count(*) AS occurrenceCount,
  count(DISTINCT o.datasetKey) AS occurrenceDatasetCount
FROM
  prod_h.occurrence o 
  JOIN tim.backbone_source name ON o.taxonKey=name.id
  JOIN tim.backbone_source accepted ON o.acceptedTaxonKey=accepted.id
WHERE o.speciesKey IS NOT NULL  
GROUP BY 
  o.taxonKey,
  name.rank,
  o.acceptedTaxonKey,
  accepted.rank,
  o.acceptedTaxonKey,
  o.kingdom,
  o.family,
  o.scientificName,
  o.acceptedScientificName,
  name.sourceKey,
  name.colSourceKey,
  name.title,
  name.colSource,
  accepted.sourceKey,
  accepted.colSourceKey,
  accepted.title,
  accepted.colSource
ORDER BY
  o.kingdom,
  o.family,
  o.scientificName,
  o.acceptedScientificName;
```
