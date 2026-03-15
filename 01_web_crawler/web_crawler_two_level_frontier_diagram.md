# Web Crawler -- Two-Level Frontier Diagram

Below is a simple Mermaid diagram showing the two-level frontier design:

-   Per-host queues (one queue per host)
-   Global scheduler (min-heap by next allowed time)
-   Workers leasing tasks

``` mermaid
flowchart TB
    Worker --> Scheduler
    Scheduler --> HostAQueue
    Scheduler --> HostBQueue

    HostAQueue --> URL1
    HostAQueue --> URL2

    HostBQueue --> URL3

    Scheduler --> Worker
```
