# Install ELK for Centralized logging (on Ubuntu)

Centralized logging can be very useful when attempting to identify problems with your servers or applications, as it allows you to search through all of your logs in a single place. It is also useful because it allows you to identify issues that span multiple servers by correlating their logs during a specific time frame. Here we cover how to install Elasticsearch , Logstash and Kibana on Ubuntu, then how to add more filters to structure your log data and how to use Kibana for a efficient log monitoring.
Sample ELK architecture (reference image from google) :
<img src="https://user-images.githubusercontent.com/12294956/46274128-f2ca7480-c575-11e8-9a50-d333495d6c25.png">

# 1. Install JDK (on central ELK system)
<ul>
<li>update the package index
<pre><code>sudo apt-get update</code></pre></li>
<li>Then, check if Java is not already installed
<pre><code>java -version </code></pre></li>
<li>If “The program java can be found in the following packages” then install jre
<pre><code>sudo apt-get install default-jre</code></pre></li>
<li>Install jdk
<pre><code>sudo apt-get install default-jdk</code></pre></li>
</ul>

# 2. Install and configure Elasticsearch (on central ELK system) <br>
Elasticsearch is a distributed, open source search and analytics engine, designed for horizontal scalability, reliability, and easy management.
<ul>
<li>Download Elasticsearch from :</br>
<a href="https://download.elastic.co/elasticsearch/elasticsearch/elasticsearch-1.4.4.zip">elasticsearch-1.4.4.zip</a> </li>
<li>Unzip it :
<pre><code>unzip elasticsearch-1.4.4.zip</code></pre></li>
<li>Configure Elasticsearch ( Change the ES port and cluster name ) :
<ul><li>Goto es_home/config/elasticsearch.yml edit the  " cluster.name: YOUR_CLUSTER_NAME " . </li>
	<li>Also change the port " http.port: 9800 " to any other port(9800 is just an example).</li></ul>
</li>
<li>Run elasticsearch :</br>
<pre><code>bin/elasticsearch</code></pre></li>
<li>ElasticSearch is now running, and can be checked from :
<pre><code>curl 'http://localhost:9800/_search?pretty'</code></pre>  OR  </br>
browse <a href="http://localhost:9800/_search?pretty">http://localhost:9800/_search?pretty</a>
</li></ul>

# 3. Install Logstash </br>
Logstash is a flexible, open source, data collection, parsing and enrichment pipeline designed to efficiently process a growing list of log, event, and unstructured data sources for distribution into a variety of outputs, including Elasticsearch.
<ul>
<li>Install and configure Logstash Forwarder/shipper
<li>Install and configure Logstash Indexer</li></ul>
<ul>
<li>Download and setup Logstash on both client and central/server machine from :</br>
<a href="https://download.elastic.co/logstash/logstash/logstash-1.4.2.zip">logstash-1.4.2.zip</a> </li>
<li>Unzip it :
<pre><code>unzip logstash-1.4.2.zip</code></pre></li>
<li>Configuration of Logstash shipper and indexer is a bit different :
<ul><li>Create a log-shipper.conf file in each shipper logstash home directory( cd logstash-1.4.2/)</li>
	<li>Similarly create a log-indexer.conf file in one and only indexer logstash home directory( cd logstash-1.4.2/)</li>
</ul></li>
<li>Run logstah using the the above created .conf file :</br>
On logstash shippers  : 
<pre><code>bin/logstash -f log-shipper.conf</code></pre></br>
Sample log-shipper.conf content :</br>
<pre><code>input {
file {
  # path => "path-to-server-logs-OR-application-logs"
  # path => "path-to-sys-logs" 
  path => "/tomcat_home/logs/catalina.out" 
  type => "client1_serverlog"
 }
}
filter {
 if [type] == "client1_serverlog" { 
grok { 
#can also use custom grok pattern by adding entry(key and corresponding regex) into grok-pattern under logstash_home/patterns and using the same key here.
# pattern => "%{COMBINEDAPACHELOG}"
 match => { "message" => "%{COMBINEDAPACHELOG}" } 
}
 }
}
output {
 redis { host => "central-elk-system-IP" data_type => "list" key => "redis-unique-key-name" }
 stdout { codec => rubydebug }
}</code></pre> OR</br>


On logstash Indexer (on central ELK system)  : 
<pre><code>bin/logstash -f log-indexer.conf</code></pre></br>
Sample log-indexer.conf content :</br>
<pre><code>input {  
 redis { 
  host => "127.0.0.1" 
  data_type => "list" 
  key => "redis-unique-key-name" 
  #redis-unique-key-name is same what specified in the shipper logstash conf
  codec => json
 }
 #can also include multiple redis block with corresponding redis-unique-key-name for acceoting logs from corresponding shipper
}
output {
 stdout { }
 # If you can't discover using multicast, set the address explicitly
 # elasticsearch { bind_host => "127.0.0.1" }
 #network.host: "127.0.0.1" OR localhost
 elasticsearch { 
 }
}</code></pre></br></li>
</ul>

# 4. Install and configure Redis(on central ELK system)
<ul>
<li>Install Redis from apt-get :</br>
<pre><code>wget http://download.redis.io/releases/redis-2.8.9.tar.gz</code>
<code>tar xzf redis-2.8.9.tar.gz</code>
<code>cd redis-2.8.9</code>
<code>make</code>
<code>make test</code> </br></pre></li>
<li>
Copy both the Redis server and the command line interface in proper places, either manually using the following commands: </br>
<pre><code>sudo cp src/redis-server /usr/local/bin/</code>
<code>sudo cp src/redis-cli /usr/local/bin/</code>
Or just using <code>sudo make install</code> </br></pre>
Here I assume that /usr/local/bin is in your PATH environment variable so that you can execute both the binaries without specifying the full path.</br></li>
<li>Start redis :</br>
<code>$ redis-server</code></br>
check if redis is working :</br>
<pre><code>$ redis-cli ping</code><code>
PONG</code></pre>
</ul>

# 5. Install and configure Kibana (on central ELK system)<br>
Kibana is an open source data visualization platform that allows you to interact with your data through stunning, powerful graphics that can be combined into custom dashboards that help you share insights from your data far and wide.
<ul>
<li>Download Kibana from :</br>
<a href="https://download.elastic.co/kibana/kibana/kibana-4.0.2-linux-x64.tar.gz">kibana-4.0.2-linux-x64.tar.gz</a> </li>
<li>Untar it :
<pre><code>tar -zxvf kibana-4.0.2-linux-x64.tar.gz</code></pre></li>
<li>Configure Kibana ( Change the Kibana port and elasticsearch url pointer ) :
<ul><li>Goto kibana_home/config/kibana.yml edit the  ' elasticsearch_url: "http://localhost:9800"' to the central ES url. </li>
	<li>Also change the kibana port " port: 5800 " to any other port(5800 is just an example).</li></ul>
</li>
<li>Run Kibana :</br>
<pre><code>bin/kibana</code></pre></li>
<li>kibana is now running, and can be checked from browser</br>
<a href="http://localhost:5800/">http://localhost:5800/</a>
</li>
</ul>

# 6.Start the entire ELK setup</br>
ELK installation and configuration is done, now its time to start the entire ELK setup :
<ul>
<li>Start indexer and shipper logstash with their corresponding config files.</li>
<li>It will start the shipping of logs from various client machines(or shipper) to central logstash/indexer through redis pipeline</li>
<li>Creating separate indices for each shipper server/machine to separate their logs from each other</li>
<li>Start kibana and point it to each shipper indices from elasticsearch cluster</li>
<li>Creating different visualisations and dashboard outoff the structured logs from elasticsearch.</li>
<li>Also you can share the Iframe url of each kibana visualisation and even the entire dashboard url to anyone.</li>
<li>If interested you can apply some proxy on the kibana iframe url so as to restrict the public access.</li>
</ul>

# 7. Scaling the ELK setup </br>
You can scale your ELK setup to a production standard by using few of the below tips :
<ul>
<li>A machine with memory 64 GB of RAM is the ideal sweet spot, but 32 GB and 16 GB machines are also common.</li>
<li>If you need to choose between faster CPUs or more cores, choose more cores. Common clusters utilize two to eight core machineswith better heap size.</li>
<li>Have a setup of single cluster with muti-nodes</li>
<li>Addproxy/security on the kibana iframe url so as to restrict the public access.</li>
</ul>

Enjoy your central ELK setup.
