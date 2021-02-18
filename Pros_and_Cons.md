#Grafana:
Pros:
    Has many templates and plug-ins
    Supports/compatible with a variety of data sources, including Prometheus and Graphite, etc
    Has customizable dashboards with customizable alerts and notifications
    Has a built in user control and authentication mechanism
    Has a diverse set of features, including snapshots, data annotations, etc
Cons:
    Data Collection and storage must be set up separately
    Grafana can be complex due to the wide variety of features and interfaces
    Because Grarana was built to perform time series analytics, its functionality may be limited with other types of reporting

#Prometheus
Pros:
    Has a powerful, self contained monitoring and alert solution
    It’s fully functional and highly reliable even when other services on your network or in the cloud are unable
    It integrates well with Grafana
Cons:
    Graphing features are primarily used for ad-hoc queries and debugging
    Prometheus would be functionally redundant in some ways if you want to visualize existing log information that’s already stored in a functioning database
