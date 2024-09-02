# Merging multiple indices into one

If there are too many indices we can merge them into a single index by using ```_reindex``` endpoint:
```sh
curl -XPOST -H "Content-Type: application/json" -d '{"source": { "index" : "some-index-pattern-*"}, "dest" : {"index" : "new-index"}}' "http://<host>>:9200/_reindex"
```

It is better to check if the enw index has nonzero document count:
```sh
curl -s -XGET "http://<host>:9200/_cat/count/new-index?h=count
```

And then we can delete the source indices

Here is a simple script to perform merging daily indices for 2023 yr into monthly (index names are defined in index_name file)
```sh
#!/bin/bash

YEAR=2023
MONTHS="01 02 03 04 05 06 07 08 09 10 11 12"
re='^[0-9]+$'
NOWMON=`date +"%Y-%m"`
HOST=myelasic01

while IFS= read -r index; do
    for m in $MONTHS; do
        if [ "$YEAR-$m" == $NOWMON ]; then
           echo "SKIPPING CURENT MONTH $NOWMON for index $index"
           continue
        fi
        INDEX=$index
        echo -n "Listing daily indexes from $index for $YEAR-$m ..."
	    curl -XGET "http://$HOST:9200/_cat/indices/$INDEX-$YEAR-$m-*?h=index" > clist 2>1
        if ! [ -s clist ]; then
           echo "not found"
           continue
        fi
        if [ -s clist ]; then
	    echo "Found unmerged data for $INDEX-$YEAR-$m: merging"
	    curl -XPOST -H "Content-Type: application/json" -d '{"source": { "index" : "$INDEX-$YEAR-$m-*"}, "dest" : {"index":"$INDEX-$YEAR-$m"}}' "http://$HOST:9200/_reindex"
            echo ".... done. Waiting"
            sleep 1
            echo "Checking merged index document count"
	    DCOUNT=$(curl -s -XGET "http://$HOST:9200/_cat/count/$INDEX-$YEAR-$m?h=count")
	    if [[ $DCOUNT =~ $re ]] && [ $DCOUNT != "0" ]; then
		echo "document count is $DCOUNT: removing daily indices"
		while IFS= read -r line; do
		    echo $line
	    	    curl -XDELETE "http://$HOST:9200/$line"
		done < "clist"
	    fi
	fi
    done
done < "index_names"
```
