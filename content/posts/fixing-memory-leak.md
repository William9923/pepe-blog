---
title: "Fixing a Memory Leak in Go Service"
date: 2023-03-19T00:05:59+07:00
draft: false
---

In this article, I will show the journey of identifying and resolving a memory leak issue in a Go service. The problem emerged when our team query service experienced a significant surge in network traffic, resulting in a continuous rise in memory and CPU usage. Our expectation was that after a prolonged and extensive query process, the memory and CPU usage would return to normal. However, this was not the case. We will delve into the investigation, root cause analysis, and solution to fix this memory leak issue.

## The Problem
Let me start the story by telling you how we encounter the problem. On a very nice day, we enjoy our amazing break time, and all of the sudden, our SRE tell us that one of our service has been reaching > 75% memory usage for the past few week. We notice that our query service memory and CPU did not decrease to the expected level after the high traffic load. 

To understand the cause of this anomaly, we turned to the PPROF tool for memory usage analysis. Using PPROF, we examined the memory usage within the application logic. Surprisingly, no memory leaks were detected at this level. However, when we monitored the actual memory usage, it continued to increase steadily. This discrepancy led us to investigate further and understand the underlying cause of the memory leak.

During our investigation, we discovered that the increasing memory usage was related to the golang Prometheus client library. This library stored all the monitored endpoints in a hashmap, and as the number of "unique" endpoints grew, it consumed a significant amount of memory resources.

![Prometheus Memory Leak](https://ibb.co/NmgkDym)

## Solution
So, how we solve this memory leak problem?

To mitigate the memory leak issue, we implemented the following practices:

1. Exclude dynamic variables:
    When monitoring inbound requests, avoid including dynamic variables such as endpoint parameters or query parameters. Instead, focus on monitoring requests based on the API path. For example, if the path is /ping?ping-name=hai, only monitor /ping as the endpoint.

2. Monitor registered endpoints:
    In an Echo server, register and monitor only the specific endpoints that have been defined and registered. By filtering out unregistered paths, we reduce the chances of monitoring excessive unique endpoints. Here are the example using Go Echo framework to collect all registered routes for the server, so prometheus can only monitor the registered endpoint...

```go
const (
    metricServiceName string = "service-metric"

    unregisteredPath string = "unregistered-path"
)

var registeredRoutes map[string]struct{} = make(map[string]struct{})

func InitRegisteredRoutes(routes []*echo.Route) {
    newRegisteredRoutes := make(map[string]struct{})
    for _, route := range routes {
   	 newRegisteredRoutes[route.Path] = struct{}{}
    }
    registeredRoutes = newRegisteredRoutes
}

func isRouteRegistered(path string) bool {
    _, ok := registeredRoutes[path]
    return ok
}
```

By implementing the suggested tips and tricks, we successfully addressed the memory leak issue in our Go service. The memory usage graph demonstrated a stable pattern, with memory and CPU usage returning to normal levels after completing a high-volume query process.

## Takeaway
This experience serves as a reminder to exercise caution when using external libraries. It is essential to thoroughly understand how the library functions and make informed decisions based on its usage guidelines. Blindly incorporating external dependencies can lead to unexpected issues such as memory leaks. By adopting best practices and being mindful of resource management, we can maintain the stability and efficiency of our software systems.
