## MongoDB - Elastic Data Sync Demo

### Overview

This is a demo for MongoDB+Elastic

Sync mongodb data to Elastic:

    https://www.linkedin.com/pulse/5-way-sync-data-from-mongodb-es-kai-hao

In this demo we will demonstrate:

- Use the **mongo-connector** method to sync the data from MongoDB to Elastic
- Perform some queries from Elastic

### Setup

Tear down

	 sudo docker ps -a | awk '{print $1}' | xargs --no-run-if-empty sudo docker rm -f

Prep (one time thing):
	
  	mkdir -p /data/mtools /data/mongodb /data/elastic
  	# fix for elasticsearch docker issue: https://github.com/docker-library/elasticsearch/issues/98
  	sudo sysctl -w vm.max_map_count=262144  
	
Start containers:

	 sudo docker run --name mongodb -p 27017:27017 -v /data/mongodb:/data/db -d mongo:3.4.0-rc5 --replSet mset

	 sudo docker run --name elastic -p 9200:9200 -p 9300:9300 --link mongodb:mongodb -v /data/elastic:/usr/share/elasticsearch/data -d elasticsearch

	 sudo docker run -v /data/mtools:/data/mtools  --link mongodb:mongodb -w /data/mtools -it stennie/ubuntu-mtools 

	

### Prep data 

Load

	# mgenerate -p 4 -n 100000 -c tracking --port 9999 tracking-template.json
  # Took 15 minutes to load 20M data on AWS

Create index

	db.tracking.ensureIndex({courier:1})
	db.tracking.ensureIndex({status:1})
	db.tracking.ensureIndex({dest:1})
	db.tracking.ensureIndex({origin:1})

### Run aggregation

Date.timeFunc(function(){
  
  db.tracking.aggregate( [
     {$match: {origin:"China"}},    
    {
      $facet: {
        "categorizedByStatus": [
          { $sortByCount: "$status" }
        ],
        "categorizedByDays": [        
          { $match: { days: { $exists: 1 } } },
          {
            $bucket: {
              groupBy: "$days",
              boundaries: [  1, 2, 3,4, 5,6,7,8,9,10 ],
              default: "Other",
              output: {
                "count": { $sum: 1 }
              }
            }
          }
        ],
        "categorizedByOrigin": [
          { $sortByCount: "$origin" }
        ]
      }
    }
  ])

}); // end timeFunc
	


When run with 100K documents, it took no time
1 million: 2500 ms
20 million: 



### Install Connector to Elastic



    sudo pip install mongo-connector
    sudo pip install elastic2-doc-manager
    mongo-connector -m localhost:27017 -t localhost:9200 -d elastic2_doc_manager


**Note about the mongo-connector:**

    First it will discover the mongo nodes by connecting to the seed node. However if the mongo is running in the container, the hostname configuration in the replset config may not be what is the one used in the mongo-connector parameter.  As such, one must manually config  a DNS resolve for the hostname in the replset config. i.e., 

      # cat /etc/hosts
      127.0.0.1   localhost 6dca744f8c34

    Note the hostname starts with 6dca












