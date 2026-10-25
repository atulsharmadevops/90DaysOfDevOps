# PromQL Made Easy: The House-Sitter Analogy

PromQL (Prometheus Query Language) is the language you use to ask questions of the numbers Prometheus has been collecting from your systems, such as "how many errors per second is my API returning right now?"

## The analogy: a house-sitter with a notebook

Imagine your house has meters everywhere: an electricity meter, a water meter, a fridge thermometer. You hire a house-sitter (**Prometheus**) who walks around every 15 seconds, reads each meter, and writes it in a notebook with a timestamp and labels, like "water meter, kitchen, 10:00:15, reading 5,230".

**PromQL is how you ask the house-sitter questions about that notebook.**

### 1. Metric name and labels are the meter and its tag

```promql
http_requests_total{method="GET", status="500"}
```

"The requests meter, but only the one labelled GET with status 500." The curly braces are your filter.

### 2. Instant vector vs range vector: a single reading vs a page of readings

- `http_requests_total` is "what does the meter say right now?" (instant vector)
- `http_requests_total[5m]` is "show me every reading from the last 5 minutes" (range vector)

### 3. Counter vs gauge: odometer vs thermometer

- A **counter** only goes up, like a car odometer or `http_requests_total`. The raw number is rarely useful on its own.
- A **gauge** goes up and down, like a thermometer or memory usage. The raw number is useful as is.

### 4. `rate()` is the speedometer

An odometer says 80,000 km, which tells you little. But if you look at how far it moved over the last 5 minutes, you get your speed. That's `rate()`:

```promql
rate(http_requests_total[5m])
```

"Requests per second, averaged over the last 5 minutes." You'll write this query constantly.

### 5. `sum by (...)` is totals per room

If you have meters in many rooms and want one total per floor:

```promql
sum by (status) (rate(http_requests_total[5m]))
```

"Requests per second, grouped by status code."

### 6. Math between queries is the bill calculation

Dividing two things gives you ratios, like an error percentage:

```promql
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```

The `=~` means "matches this pattern", so `5..` catches 500, 502, 503, and so on.

## How to remember it

> **Pick the meter, take a page, find the speed, add up the rooms.**

1. **Pick the meter** (metric name and `{labels}`)
2. **Take a page** (`[5m]` range)
3. **Find the speed** (`rate()`)
4. **Add up the rooms** (`sum by (...)`)

Most real-world queries are just those four steps in that order:

```promql
sum by (service) (rate(http_requests_total{status="500"}[5m]))
```

Read it inside-out: pick the 500s, take the last 5 minutes, find the per-second rate, and total it per service.

## Quick self-test

1. What's the per-second rate of requests to `/login` over the last 10 minutes?
   **Answer:** `rate(http_requests_total{path="/login"}[10m])`
2. Why is `rate()` the wrong thing to use on a gauge like `memory_usage_bytes`?
   **Answer:** A thermometer isn't an odometer. There's no "distance traveled" to convert into speed, so just read the value directly.