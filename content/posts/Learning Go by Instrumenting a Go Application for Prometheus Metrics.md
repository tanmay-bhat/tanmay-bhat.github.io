---
layout: post
title: Learning Go by Instrumenting a Go Application for Prometheus Metrics
date: 2024-10-18
tags: ["Go", "Prometheus", "Datadog"]
---

### A Beginner's Perspective
Before we dive into the details, I recently started learning Go. This article is just a beginner's perspective on combining Go learning and building a simple Prometheus metrics exporter.

I had a requirement to build this exporter because, at my workplace, we use [Datadog's SLO](https://www.datadoghq.com/product/service-level-objectives/) product alongside RUM monitoring. However, since all our other analytics and metrics are in Prometheus, I built this to consolidate all SLOs in one place.

### Datadog API Response Example

For an example request `curl -X GET "https://api.datadoghq.com/api/v1/slo/${slo_id}/history"`, the response is as follows:

```json
{
  "data": {
    "overall": {
      "name": "Example SLO",
      "sli_value": 0.99
      ...
    },
    "slo": {
      "target_threshold": 0.95,
      "timeframe": "7d"
      ...
    },
    ...
  }
}

```

### Exporter Requirements

For simplicity, let's focus on creating a Prometheus exporter that:

1. Parses the Datadog API response
2. Creates Prometheus metrics for the following example data:

```
datadog_slo_uptime{rolling_timeframe="<>" slo_name="<>" threshold="<>" window="<>"}

datadog_api_error_total{api_call="<>",status_code="<>"}
```

### Importing Required Client Packages

As we need clients of Datadog and Prometheus, let's import both:
```go
import (
    "github.com/Datadog/datadog-api-client-go/v2/api/datadog"
	"github.com/Datadog/datadog-api-client-go/v2/api/datadogV1"
	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/push"
)
```

### Constructing Structs to Match API Response

As the data we need is inside the data field, in the API response, let's create a struct to parse them later on.

A struct in Go is a container to group similar or related data together, somewhat similar to classes in other object-oriented languages.

Let's create a struct called `Response` which will contain `Data` of type `Data` struct and to match the overall structure of the API response :


```go
type Response struct {
    Data Data `json:"data"`
}
```

We specified `json:"data"` because that's the way of [mapping](https://pkg.go.dev/encoding/json) the actual JSON response's data field to this object.

As `overall` and `slo` are nested fields of `data`, let's create a struct called `Data` and add these two inside that:

```go
type Data struct {
    Overall Overall `json:"overall"`
    Slo     Slo     `json:"slo"`
}
```

Now, let's create structs called `Overall` and `Slo` which will contain the actual data we need:

```go
type Overall struct {
    Name string  `json:"name"`
    SLI  float64 `json:"sli_value"`
}

type Slo struct {
    Threshold float64 `json:"target_threshold"`
    Timeframe string  `json:"timeframe"`
}
```

### Global Variables and Metric Declarations

We declare the following variables:

```go
var (
    API_KEY       = os.Getenv("DD_API_KEY")
    APP_KEY       = os.Getenv("DD_APP_KEY")
    SLO_ID        = os.Getenv("DD_SLO_ID")
    PROM_ENDPOINT = os.Getenv("PROMETHEUS_ENDPOINT")

    ctx       context.Context
    api       *datadogV1.ServiceLevelObjectivesApi
    apiClient *datadog.APIClient

    // Declare Prometheus metrics for monitoring Datadog SLO
    DataDogSLOGauge = prometheus.NewGaugeVec(prometheus.GaugeOpts{
        Namespace: "datadog",
        Name:      "slo_uptime",
        Help:      "History details of a Datadog SLO"},
        []string{"slo_name", "threshold", "window", "rolling_timeframe"},
    )
    // Declare Prometheus metrics for monitoring Datadog API Errors
    DataDogAPIErrorCounter = prometheus.NewCounterVec(prometheus.CounterOpts{
        Namespace: "datadog",
        Name:      "api_error_total",
        Help:      "Total Error count on requests to Datadog API"},
        []string{"api_call", "status_code"},
    )

    logger *log.Logger
)
```

### Implementing the Exporter

Now that we have our structs defined to match the Datadog API response, let's implement the core functionality of our exporter.

**Initializing the Datadog Client**

First, we initialize the Datadog client. We'll create a function called `InitDataDogClient()`:

```go
func InitDataDogClient() error {
    ctx = context.WithValue(context.Background(), datadog.ContextAPIKeys,
        map[string]datadog.APIKey{
            "apiKeyAuth": {Key: API_KEY},
            "appKeyAuth": {Key: APP_KEY},
        },
    )
    configuration := datadog.NewConfiguration()
    configuration.RetryConfiguration.EnableRetry = true // defaults to 3 retries
    apiClient = datadog.NewAPIClient(configuration)
    api = datadogV1.NewServiceLevelObjectivesApi(apiClient)
    return nil
}
```
Context in Go is a way to manage timeouts, cancellation, and passing data like API keys across multiple functions or API boundaries.
In the above example, context is used to pass the Datadog API credentials along with API requests.

The above function also checks if the necessary environment variables are set, creates a context with the API keys, and initializes the Datadog client and then create an instance of `NewServiceLevelObjectivesApi` API and set it to `api` variable.

**Fetching SLO Data**

Next, let's implement the `GetSloData` function to fetch SLO data from Datadog:

```go
func GetSloData(sloDataID string, daysWindow int) (sliValue float64, sloName string, threshold float64, rollingTimeframe string, error error) {
    if sloDataID == "" {
        logger.Fatal("sloDataID environment variable is not set")
    }

    fromTime := time.Now().AddDate(0, 0, -daysWindow).Unix()
    toTime := time.Now().Unix()

    resp, r, err := api.GetSLOHistory(ctx, sloDataID, fromTime, toTime)
    if err != nil {
        statusCode := 0
        if r != nil {
            statusCode = r.StatusCode
        }
        DataDogAPIErrorCounter.WithLabelValues("GetSLOHistory", strconv.Itoa(statusCode)).Inc()
        return 0, "", 0, "", fmt.Errorf("error when calling `ServiceLevelObjectivesApi.GetSLOHistory`: %v", err)
    }

	responseContent, err := json.Marshal(resp)
	if err != nil {
		return 0, "", 0, "", fmt.Errorf("error marshaling response: %v", err)
	}

    var response Response
    err = json.Unmarshal(responseContent, &response)
    if err != nil {
        logger.Printf("Error unmarshaling JSON: %s\n", err)
        return
    }

    sli = response.Data.Overall.SLI
    slo = response.Data.Overall.Name
    threshold = response.Data.Slo.Threshold
    rollingTimeframe = response.Data.Slo.Timeframe

    return sli, slo, threshold, rollingTimeframe, nil
}
```
This function takes the SLO ID and the number of days to look back as parameters. It then makes a request to the Datadog API, parses the response, and returns the relevant SLO data.


1. API Request
    `api.GetSLOHistory` is called with the provided SLO ID and time window.
    - Returns three values:
        - `resp` (SLOHistoryResponse struct): Contains the API response data.
        - `r` (_nethttp.Response): The actual HTTP response.
        - `err` (error): Any error that occurred during the request.
2. Error Handling & Metric Update
    If an error occurs, update the `DataDogAPIErrorCounter` metric:
    - Set the `api_call` label to `"GetSLOHistory"`.
    - Convert the HTTP status code (`r.StatusCode`) to a string using `strconv.Itoa()` and set it as the `status_code` label. This is needed as labels can only be of type `string` in Prometheus.
    - Increment the metric value using `Inc()`.
3. Response Parsing
    - As we know that `resp` is of type SLOHistoryResponse which is a struct, convert it into a JSON using `json.Marshal(resp)` and store the result in `responseContent`.
    - Declare a `Response` struct variable, `response`.
    - Unmarshal the JSON data into the response struct using `json.Unmarshal(responseContent, &response)`.
      - It takes JSON data and a pointer as arguments. In our case, we passed a pointer to the response variable `(&response)`.
4. Extract Relevant SLO Data
    - Access the parsed data using the response struct, e.g., `sliValue = response.Data.Overall.SLI`

Note: Prometheus labels are always sorted alphabetically.

**Main Function**

Now, let's implement our `main()` function to tie everything together:

```go
func main() {
    logger = log.New(os.Stdout, "", log.Ldate|log.Ltime|log.Lshortfile)
    err := InitDataDogClient()
    if err != nil {
        logger.Printf("Failed to initialize Datadog client: %v\n", err)
        return
    }
    logger.Printf("Datadog client initialized successfully\n")

    // Use a custom registry for consistency and to drop default go_.*  metrics
    registry := prometheus.NewRegistry()
    registry.MustRegister(DataDogSLOGauge, DataDogAPIErrorCounter)

    days := []int{7, 30, 90}

    // Fetch SLO data for different time windows
    for _, day := range days {
        logger.Printf("Fetching SLO history for %d days\n", day)
        sliValue, sloName, threshold, rollingTimeframe, err := GetSloData(SLO_ID, day)

        if err != nil {
            logger.Printf("Error getting SLO data: %v\n", err)
        } else {
            g := DataDogSLOGauge.WithLabelValues(
                sloName,
                fmt.Sprintf("%.2f", threshold),
                fmt.Sprintf("%dd", day),
                rollingTimeframe,
            )
            g.Set(sliValue)
        }
    }

    // Collect all the metrics and push to Prometheus endpoint
    pusher := push.New(PROM_ENDPOINT, "datadog-slo-exporter").Gatherer(registry)

    if err := pusher.Push(); err != nil {
        logger.Printf("Error pushing metrics to VictoriaMetrics: %v\n", err)
    } else {
        logger.Printf("Metrics pushed to VictoriaMetrics successfully\n")
    }
}
```

I set the model to `push` instead of Prometheus's default `pull` since we don't need to run this as a service. The best way to run this is as a cronjob.

### Key Concepts in Prometheus Client Library
- Collector : 
A collector is a part of an exporter that represents a set of metrics. It may be a single metric if it is part of direct instrumentation, or many metrics if it is pulling metrics from another system.
In our case, we used two collector types, Gauge and Counter.
- Registry : 
A registry is a central place to store and manage multiple collectors. It keeps track of all registered metrics and their values.
Prometheus client by default has a registry which collects go related metrics, for our simple use case, it's overkill, hence we create a new registry and register our metrics to that.
```go
	registry := prometheus.NewRegistry()
	registry.MustRegister(DataDogSLOGauge, DataDogAPIErrorCounter)
```
- Gatherer : 
A gatherer is an interface responsible for collecting metrics from registered collectors. It gathers metrics from all collectors in a registry and prepares them for scraping by Prometheus.
- As we use a custom registry called `registry`, while pushing the metric, we specify that.
```go
push.New(PROM_ENDPOINT, "datadog-slo-exporter").Gatherer(registry)
```

### Using Prometheus Pull (Alternative to Push)

```go
import (
        "net/http"
        "github.com/prometheus/client_golang/prometheus/promhttp"
)
func main() {
        http.Handle("/metrics", promhttp.Handler(registry))
        http.ListenAndServe(":2112", nil)
}
```

---
### References
- https://docs.datadoghq.com/api/latest/service-level-objectives/#get-an-slos-history
- https://github.com/prometheus/client_golang
- https://github.com/Datadog/datadog-api-client-go
- https://prometheus.io/docs/practices/instrumentation/

Complete code can be found here : https://github.com/tanmay-bhat/datadog-slo-exporter