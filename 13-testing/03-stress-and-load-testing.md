# Stress and Load Testing

## Why Load Test

Unit and integration tests verify **correctness**. Load tests verify **performance under pressure**:

- Can the system handle 1,000 concurrent users?
- What happens when traffic spikes 10x?
- Where are the bottlenecks (CPU, memory, database, network)?
- At what point does the system break?

## Types of Performance Testing

| Type | Goal | Pattern |
|------|------|---------|
| **Load test** | Can it handle expected traffic? | Ramp up to N users, hold, ramp down |
| **Stress test** | Where does it break? | Push beyond expected limits |
| **Spike test** | Can it handle sudden surges? | Sudden jump to peak, then drop |
| **Soak test** | Are there memory leaks / degradation? | Moderate load for hours |
| **Breakpoint test** | What is the absolute limit? | Incrementally increase until failure |

## k6 (by Grafana Labs)

Most popular modern load testing tool. Write tests in **JavaScript**, run from CLI, integrates with Grafana dashboards.

### Installation

```bash
# Windows (Chocolatey)
choco install k6

# macOS
brew install k6

# Docker
docker run --rm -i grafana/k6 run - < script.js
```

### Basic load test

```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 50 },   // ramp up to 50 users
    { duration: '1m', target: 50 },    // hold at 50 users
    { duration: '30s', target: 0 },    // ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% of requests must be under 500ms
    http_req_failed: ['rate<0.01'],    // less than 1% failure rate
  },
};

export default function () {
  const res = http.get('https://api.myapp.com/api/products');
  
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
    'body has data': (r) => JSON.parse(r.body).length > 0,
  });

  sleep(1); // simulate user think time
}
```

```bash
k6 run load-test.js
```

### Stress test

```javascript
export const options = {
  stages: [
    { duration: '2m', target: 100 },   // ramp up
    { duration: '5m', target: 100 },   // hold
    { duration: '2m', target: 200 },   // push beyond normal
    { duration: '5m', target: 200 },   // hold at stress level
    { duration: '2m', target: 300 },   // push to breaking point
    { duration: '5m', target: 300 },   // hold
    { duration: '2m', target: 0 },     // ramp down
  ],
};
```

### Spike test

```javascript
export const options = {
  stages: [
    { duration: '10s', target: 10 },   // normal traffic
    { duration: '1s', target: 500 },   // sudden spike!
    { duration: '30s', target: 500 },  // hold spike
    { duration: '1s', target: 10 },    // back to normal
    { duration: '30s', target: 10 },   // recovery
  ],
};
```

### Testing authenticated endpoints

```javascript
import http from 'k6/http';
import { check } from 'k6';

export function setup() {
  // Runs once before the test — get auth token
  const loginRes = http.post('https://api.myapp.com/auth/login', JSON.stringify({
    email: 'loadtest@example.com',
    password: 'testpassword'
  }), { headers: { 'Content-Type': 'application/json' } });

  return { token: JSON.parse(loginRes.body).accessToken };
}

export default function (data) {
  const headers = { Authorization: `Bearer ${data.token}` };
  
  const res = http.get('https://api.myapp.com/api/orders', { headers });
  check(res, { 'status is 200': (r) => r.status === 200 });
}
```

### Testing POST / creating data

```javascript
export default function () {
  const payload = JSON.stringify({
    name: `Product ${__ITER}`,
    price: Math.random() * 100,
  });

  const res = http.post('https://api.myapp.com/api/products', payload, {
    headers: { 'Content-Type': 'application/json' },
  });

  check(res, {
    'created': (r) => r.status === 201,
  });
}
```

## Key Metrics

| Metric | Description | Healthy target |
|--------|-------------|----------------|
| **p95 response time** | 95th percentile latency | < 500ms for APIs |
| **p99 response time** | 99th percentile latency | < 1s |
| **Throughput (RPS)** | Requests per second | Depends on requirements |
| **Error rate** | % of failed requests | < 1% |
| **Concurrent users** | Active virtual users | Match production expectations |

## k6 Output and Visualization

```bash
# Output to terminal
k6 run script.js

# Output to JSON
k6 run --out json=results.json script.js

# Output to InfluxDB + Grafana
k6 run --out influxdb=http://localhost:8086/k6 script.js
```

### k6 Cloud (managed)

```bash
k6 cloud script.js  # runs on Grafana Cloud with dashboards
```

## Other Load Testing Tools

| Tool | Language | Strengths |
|------|----------|-----------|
| **k6** | JavaScript | Modern, CLI-first, Grafana integration |
| **JMeter** | Java/GUI | Enterprise, very flexible, heavy |
| **Gatling** | Scala/Java | Good reports, CI/CD friendly |
| **Locust** | Python | Simple, distributed, code-first |
| **NBomber** | C#/F# | .NET native, good for .NET teams |
| **Artillery** | JavaScript | YAML config, easy to start |

### NBomber (.NET native)

```csharp
var scenario = Scenario.Create("load_test", async context =>
{
    var response = await httpClient.GetAsync("https://api.myapp.com/api/products");
    return response.IsSuccessStatusCode
        ? Response.Ok()
        : Response.Fail();
})
.WithLoadSimulations(
    Simulation.Inject(rate: 50, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromMinutes(2))
);

NBomberRunner.RegisterScenarios(scenario).Run();
```

## Load Testing in CI/CD

```yaml
# GitHub Actions example
- name: Run load tests
  run: |
    k6 run --out json=results.json load-test.js
    
- name: Check thresholds
  run: |
    # k6 exits with non-zero code if thresholds fail
    # This automatically fails the pipeline
```

> Set **realistic thresholds** — failing a deploy because p95 > 500ms prevents performance regressions.

## Best Practices

1. **Test against a staging environment** — never load test production (unless you have explicit approval)
2. **Use realistic data** — synthetic IDs and random strings don't trigger the same DB queries
3. **Include think time** — real users don't click every millisecond, use `sleep(1-3)`
4. **Warm up the system** — JIT compilation, connection pools, and caches need a few seconds to stabilize
5. **Monitor the server** — CPU, memory, DB connections, and queue depth during the test
6. **Test regularly** — not just before release. Performance regressions creep in over time
7. **Start small** — begin with load testing, then stress, then soak
8. **Correlate with APM** — use Application Insights / Grafana to identify bottlenecks during the test

---

[← Previous: Mocking and Best Practices](02-mocking-and-best-practices.md) | [Back to index](README.md)
