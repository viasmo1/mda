# Exercise Nifi + ELK

## Objective

Using Nifi + ELK, represent a solution that shows in a map the crimes location of the following API:

https://data.cityofnewyork.us/Social-Services/311-Service-Requests-from-2010-to-Present/erm2-nwe9

 * Content of the json response: [json](content.json)

 * More info on how to query result: [api docs](https://dev.socrata.com/foundry/data.cityofnewyork.us/erm2-nwe9)

## Solution

### Running the docker-compose

* Run the docker compose file with Nifi, Elasticsearch and Kibana:

    ```sh
    docker-compose up -d
    ```

* Open the browser and check that all three containers are running:

    * Nifi: http://localhost:8080/nifi

        <img src="img/nifi.png" size=500px>

    * Elasticsearch: http://localhost:9200

        <img src="img/elastic.png" size=350px>

    * Kibana: http://localhost:5601

        <img src="img/kibana.png" size=500px>


### Dealing with the geo_point

* In Kibana, create the index *crimes* and map the *location* property with a geo_point data type:

    <img src="img/crimes_mapping.png" size=500px>

* As explained in the elasticsearch docs, the [geo_point](https://www.elastic.co/guide/en/elasticsearch/reference/current/geo-point.html) has to have an specific format. In this case, we will use the first one, a dictionary with the following structure:

    ```sh
    {
        "location": { 
            "lat": 41.12,
            "lon": -71.34
        }
    }
    ```

    * The *location* property in the [json](content.json) returned by the API has the following structure:

        ```sh
        {
            "location": { 
                "latitude": 41.12,
                "longitude": -71.34,
                "human_address": "{\"address\": \"\", \"city\": \"\", \"state\": \"\", \"zip\": \"\"}"
            }
        }
        ```

        Thus, we need to modify it with Nifi so that we get the structure required for the geo_point.


### Nifi fileflow

[Template](nifi+elk_NYC_crimes.xml)

<img src="img/nifi_flow.png" size=500px>

| Processor | Usage |
| --- | --- |
| InvokeHTTP | Connect to the API and gets the dataset |
| SplitJSON | Splits the returned JSON into JSON docs |
| ReplaceText | Replaces "latitude" for "lat" |
| ReplaceText | Replaces "longitude" for "lon" |
| JoltTransformJSON | Removes the "human_address" property inside "location" |
| EvaluateJsonPath | Gets the attribute "unique_key" from json and saves it into a FlowFile attribute named "unique_key" |
| PutElasticsearchHTTP | Uploads the resultings docs into Elasticsearch |

* Configuration:

    * InvokeHTTP (scheduling time = 120seconds)
    
        * TIP: you can use url filters to get specific results:

            * ?$limit=40000
            * ?$where=created_date between '2020-01-01T00:00:00.000' and '2021-01-15T00:00:00.000'
            * ?$where=created_date>'2020-01-01T00:00:00.000'
            * ?$limit=40000&$where=created_date>'2020-01-01T00:00:00.000'

        <img src="img/invokehttp_settings.png" size=400px>        

    * SplitJSON:

        <img src="img/splitjson_settings.png" size=400px> 

    * ReplaceText 1:

        <img src="img/replacetext1_settings.png" size=400px> 

    * ReplaceText 2:

        <img src="img/replacetext2_settings.png" size=400px> 

    * JoltTransformJSON ([web with interesting examples](https://community.cloudera.com/t5/Community-Articles/Jolt-quick-reference-for-Nifi-Jolt-Processors/ta-p/244350)):

        <img src="img/jolttransformjson_settings.png" size=400px> 

    * EvaluateJsonPath ([web with interesting examples](https://help.syncfusion.com/data-integration/processors/evaluatejsonpath)) ([another interesting example](https://stackoverflow.com/questions/51820430/apache-nifi-evaluatejsonpath-processor-jsonpath-expression-to-concatenate-2-att/51833973#51833973)):

        <img src="img/evaluatejsonpath_settings.png" size=400px> 

    * PutElasticsearchHTTP:

        <img src="img/putelasticsearchhttp_settings1.png" size=400px>
        <img src="img/putelasticsearchhttp_settings2.png" size=400px> 

### Visualizing NYC crimes data with Kibana

* Check that the docs have been uploaded into the index *crimes*. For that, we go into **Index Management** in Kibana.

    <img src="img/index_management.png" size=400px>

* Create an Index Pattern for the *crimes* index.

    <img src="img/crimes_indexpattern.png" size=400px>

* Create a new *Coordinate Map* visualization with the *crimes* index pattern and configure it as follows:

    * Metrics: *Unique count* of *unique_key*
    * Buckets: *Geo hash* of *location*

    <img src="img/crimes_map.png" size=400px>


**THAT'S IT! WE CAN NOW START ANALYZING OUR VISUALIZATION!**

For example, here's a visualization of the *last 3 days* crimes at the *BRONX* with *Open* status

<img src="img/visualization1.png" size=400px>

Or create a dashboard with other visualizations:

<img src="img/crimes_dashboard.png" size=600px>