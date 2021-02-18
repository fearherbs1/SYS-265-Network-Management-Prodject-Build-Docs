# Grafana:  
## Pros:    
   * Has many templates and plug-ins  
   * Supports/compatible with a variety of data sources, including Prometheus and Graphite, etc  
   * Has customizable dashboards with customizable alerts and notifications  
   * Has a built in user control and authentication mechanism  
   * Has a diverse set of features, including snapshots, data annotations, etc  
## Cons:  
   * Data Collection and storage must be set up separately  
   * Grafana can be complex due to the wide variety of features and interfaces  
   * Because Grarana was built to perform time series analytics, its functionality may be limited with other types of reporting  

# Prometheus:  
## Pros:  
   * Has a powerful, self contained monitoring and alert solution  
   * It’s fully functional and highly reliable even when other services on your network or in the cloud are unable  
   * It integrates well with Grafana  
## Cons:  
   * Graphing features are primarily used for ad-hoc queries and debugging  
   * Prometheus would be functionally redundant in some ways if you want to visualize existing log information that’s already stored in a functioning database  


# Do We Recommend it? 

* After working with this stack I can say that we recommend this software. The fast way of deploying monitoring on hosts with windows exporter and node exporter makes it easy to scale when more machines are added. There is also a large database of premade dashboards on the grafana community site. This makes it easy to display data without messing with too many queries. The alerting is also very good. It has tons of possibilities with all the data Prometheus brings in and it can even alert to something like discord.
