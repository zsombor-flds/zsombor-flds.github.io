---
title: >-
  Building an IoT Data Pipeline with DuckDB and Malloy.

date: '2023-05-04T06:46:37.118Z'
tags: ["DuckDB", "malloy"]
type: post
weight: 25
showTableOfContents: true
---

As many of you probably know, energy prices in the EU skyrocketed in 2022. My primary heating source is electric, so last winter, I wanted to understand my home electricity usage better.


That’s why I developed a data pipeline that allows me to monitor the electricity consumption of my home.

At work, I spend most of my Engineering related time interacting with Cloud Services, so to explore something different, I challenged myself to develop a data pipeline that only computes on the edge.

My first idea was to invest in smart plugs that enable the end user to pull the measured consumption data from the device, but at the time, I wasn’t lucky enough to find one that matched my preferences because most of them lacked a public API. In the past, I spent more time than I’d like to admit trying to reverse-engineer things like my [Nobo Hub](https://www.nobo.hu/en/products/nobo-eco-hub), but in this situation, it seemed too much of a hassle. And while I’m impressed by the flourishing smart home community, the direction of utilizing a home assistant such as [home-assistant.io](https://www.home-assistant.io/) didn’t appear worthwhile for me since I was only going for energy monitoring.

Fortunately, 2022 was the year I discovered DuckDB, and ever since, I have been looking for opportunities to get more out of it: whether it comes to building CLIs or bringing the “Cloud” to my laptop.

After a bit of tinkering, I came up with this simple architecture:

![](/images/1__i7OQZyPc0AmYqfoMx7B0__Q.png)

To sum it up, I set up an ESP8266 to read the measurement data from the Smart Meter via its serial port. The collected events were then parsed and sent to an API running on a Raspberry PI. Afterward, I persist the raw events as parquet files, and lastly, I utilize DuckDB with Malloy for transformation and analysis. In the next few paragraphs, I will walk you through each stage.

### Event Collection

Fortunately, the Smart Meter provided by my electricity supplier is equipped with a [DSMR](https://en.wikipedia.org/wiki/Open_metering_system) standard P1 port. DSMR stands for Dutch Smart Meter Requirements, a standard for smart meters created in the Netherlands. This standard enables end users to monitor their energy consumption/billing data.

I came across several projects that aimed to parse data from Smart Meters, such as [dsmr-reader](https://github.com/dsmrreader/dsmr-reader), but ultimately, none of them were compatible with the [particular Smart Meter](https://www.eon.hu/content/dam/eon/eon-hungary/documents/Muszaki-ugyek/Fogyasztasmerok-leirasa/sanxing-sx6x1/EON-leiras-az-ugyfelek-szamara-SX6X1-S12U16-S34U18-V010.pdf) I have. However, with some troubleshooting and online research, I found a datasheet that helped me parse the messages received via the serial interface. The raw data gained from the Serial Connection looked like [this](https://gist.github.com/zsombor-flds/665c34de506a52340a229e0b3659b246.).

I programmed a NodeMCU to collect the events from the Smart Meter. The app retrieves data from the P1 serial connection, then parses the relevant information from the raw event, and finally transfers it as a JSON object to a simple Python API. Additionally, I added a DHT22 sensor to measure outdoor temperature and humidity.

Example output from the ESP:
```json
{  
 "power\_timestamp": "230421091600S",  
 "power\_device\_name": "AUX103030299999",  
 "power\_breaker\_status": "000",  
 "power\_consumption": "009994.902\*kWh",  
 "l1\_phase\_voltage": "240.5\*V",  
 "l2\_phase\_voltage": "239.8\*V",  
 "l3\_phase\_voltage": "236.3\*V",  
 "l1\_phase\_current": "000\*A",  
 "l2\_phase\_current": "000\*A",  
 "l3\_phase\_current": "000\*A",  
 "humidity": 40.4,  
 "temperature": 7.6  
}
```

You can check out the PoC version of the event collection [here](https://gist.github.com/zsombor-flds/1a91e3f953c9d6be04f0bca12e9d74c2%29).

### Persist Service

![](/images/1__tQlg9qLnKpk8LZr0zhM20w.png)

To store data collected by the ESP8266, I created a simple Flask service with Python that persists the received data as partitioned parquet files. I chose this approach because it’s simple and maintainable.

```python
@app.route("/sensor", methods=\["POST"\])  
async def dummly\_handle\_request():  
  raw\_data = json.loads(request.data)     
  transformed = transform(data=raw\_data)  
  persist\_to\_parquet(data=transformed)  
  
  return "ok"
```

Since the update interval is between 1–10 seconds for the DSMR protocol, I didn’t worry about the throughput being too large. A simple HTTP service was enough to transform the serialized JSON objects into parquet. Moreover, since my plan was to deploy it on a Raspberry PI, I didn’t want to introduce more complexity and service costs with a regular streaming platform/ message broker like Red Panda. Realistically, a lightweight MQTT service would also have worked, and obviously, I would choose those options in a production environment.

```python
from deltalake.writer import write\_deltalake  
write\_deltalake('/raw/',  
  df,  
  mode='append',   
  partition\_by=\['year', 'month', 'day'\]  
)
```
### Transformation and exploration with DuckDB and Malloy

Once the data is available in date partitioned parquet structure, I used [DuckDB](http://duckdb.org) to query and explore the dataset directly. DuckDB is optimized for analytical workloads — therefore, it boasts exceptional query performance thanks to its columnar storage and vectorized query execution engine.   
I chose DuckDB because it’s lightweight and embeddable, which makes it an excellent fit for edge computing.

With DuckDB, reading from parquet files is as easy as:

```sql
select  
  measure\_timestamp,  
  power\_consumption\_khw,  
  outside\_temperature    
from read\_parquet('~/raw/year=\*/month=\*/day=\*/\*.parquet')  
limit 10;
```

After applying some basic wrangling of the collected data, I wanted to explore and visualize it to uncover patterns and insights. I used [Malloy](https://www.malloydata.dev/) to analyze and create charts to display energy consumption trends and check the correlation with outside temperature and humidity levels.

A little background info: Malloy is an innovative language designed to define data relationships and transformations. It serves as both a semantic modeling language and a structured querying language (pun intended), enabling users to run queries on relational databases. DuckDB is natively supported in Malloy, along with BigQuery, and Postgres. Therefore, it is a superb choice to create reusable explores against your data model.

I used the Visual Studio Code extension in a Remote Explorer to streamline the process of constructing queries and generating visualizations and dashboards well.

![](/images/1__3smjX9sIRfq01K7qfUZBLQ.png)

I let this pipeline run in the past five months to ensure I have enough data to make my observations. Through these visualizations, I identified several energy consumption peaks throughout the day that could be addressed by adjusting my home’s heating schedule. Armed with this knowledge, I was able to take steps to optimize my home’s energy usage and reduce consumption, cut down my electricity bill, and reduce my impact on the environment.

![](/images/1__i7xJV6oudPCDToNmphaEXA.png)

It was very instructive for me to build a data processing pipeline only using open-source technologies without touching cloud services or large on-prem infrastructures. Most of the prototype code is available [here](https://github.com/zsombor-flds/dsmr-data-flow). If you want to learn more about Malloy, check out this interactive [demo](https://github.dev/malloydata/try-malloy/airports.malloy).

In conclusion, I hope this blog post has provided you with valuable insights into the process of creating efficient IoT pipelines and demonstrated the impressive potential of DuckDB for enhancing local performance that rivals the Cloud.

Feel free to reach [out](http://hello@hiflylabs.com) if you’re interested in building a similar data pipeline or have any questions about my setup.
