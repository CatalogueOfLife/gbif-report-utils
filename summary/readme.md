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

There results of this are [available for download](https://download.gbif.org/tim/col/taxa_summary.tsv.gz).

#### Step 3: Analyse the results

_The following is based on the procedure above, using the GBIF Backbone [23 November 2022](https://hosted-datasets.gbif.org/datasets/backbone/2022-11-23/)_

There are many ways in which one might analyse this result. Some examples and summaries are given.

**EVERYTHING BELOW SHOULD BE SCRUTINIZED BEFORE DRAWING ANY CONCLUSIONS - IT MAY CONTAIN ERRORS**


1. `2,995,271` names in total for the `2,178,064,458` occurrences
    ```
    SELECT count(*), sum(occurrenceCount) FROM tim.taxa_summary;
    ```

2. `2,780,033` names and `2,172,337,244` occurrences when name starting with `BOLD:` or `SH[0-9]` are excluded
    ```
    SELECT count(*), sum(occurrenceCount) FROM tim.taxa_summary WHERE scientificName NOT LIKE 'BOLD%' AND scientificName NOT RLIKE('^SH[0-9]*');
    ```

3. `2,150,605` names (`77%`) from `2,060,937,274` occurrence (`95%`) can be organised using the COL alone, where COL provides the accepted name, understanding synonymy where necessary.

   ```
   SELECT 
     count(*) AS names, sum(occurrenceCount) AS occurrences
   FROM 
     tim.taxa_summary
   WHERE 
     (nameSource = 'Catalogue of Life Checklist' OR nameSource IS NULL) AND
     (acceptedNameSource = 'Catalogue of Life Checklist' OR acceptedNameSource IS NULL) AND
     scientificName NOT LIKE 'BOLD%' AND scientificName NOT RLIKE('^SH[0-9]*');     
   ```
