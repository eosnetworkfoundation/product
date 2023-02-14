# Prometheus Plugin Test Setup

* Install Prometeus 
  * wget https://github.com/prometheus/prometheus/releases/download/v2.42.0/prometheus-2.42.0.linux-amd64.tar.gz
  * tar xvzf prometeus-2.4.2.0.linux-amd64.tar.gz
* Configure Prometheus
  * Take note of your nodeos topology.  The endpoint for the nodeos prometheus_plugin will be on the http_plugin port at:
    /v1/prometheus/metrics. So a typical one node configuration
    will be http://127.0.0.1:8888/v1/prometheus/metrics.  If you are running via one of the test scrtips, the regression
    test scripts, the first node will run on port 8888, and each of the next will use the next port number, so 
    8888, 8889, 8890...
  * In the prometheus.yml file, add sections for each nodeos node for which you want to collect metrics, in the form of:
   
```YML
   - job_name: "nodeos-0"
     metrics_path: /v1/prometheus/metrics
     static_configs:
       - targets: ["127.0.0.1:8888"]
```

    targets should match the ip/port of the node that you are trying to target.
* Run Prometheus.  Go to the prometheus directory and run prometheus by executing './prometheus'.  By default,
  prometheus will use the local prometheus.yml file.  To get more detail, you can run with the -v option (for verbose).
* Run nodeos.  A simple way to test nodeos is to run one of the integration test scripts.  You will need to
  enable the prometheus_plugin.  You can do this by adding an extraNodeosArgs="--plugin prometheus_plugin" option
  to the cluster launch parameter.
* Query prometheus.  To query prometheus, bring up a web browser and go to the prometheus web page:
  "http://127.0.0.1:9090/".  This will bring up a web page that will allow you to view metrics collected
  from the nodeos instances.

   

