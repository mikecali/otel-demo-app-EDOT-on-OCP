# EDOT on OpenShift - Complete Deployment and Testing Guide

## Overview

This guide demonstrates how to deploy and test the Elastic Distribution of OpenTelemetry (EDOT) on OpenShift using the official OpenTelemetry Demo application. You'll get 11+ microservices instrumented across multiple languages with automatic distributed tracing.

**Prerequisites:** OpenShift 4.12+ cluster access with admin privileges

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Part 1: Install OpenTelemetry Operator](#part-1-install-opentelemetry-operator)
- [Part 2: Deploy OpenTelemetry Demo Application](#part-2-deploy-opentelemetry-demo-application)
- [Part 3: Configure Instrumentation](#part-3-configure-instrumentation)
- [Part 4: Generate Realistic Traffic](#part-4-generate-realistic-traffic)
- [Part 5: Verify Telemetry in Elastic Cloud](#part-5-verify-telemetry-in-elastic-cloud)
- [Part 6: Advanced Testing](#part-6-advanced-testing)
- [Troubleshooting](#troubleshooting)
- [Fix Common Issues](#fix-common-otel-app-issues)
- [Cleanup](#cleanup)
- [Appendix](#appendix)

---

## Prerequisites

### Required Access

- OpenShift cluster (v4.12+) with cluster-admin privileges
- `oc` CLI tool installed and configured

### Required Credentials

You'll need an Elastic Cloud deployment with:

- **Cloud ID** (format: `deployment-name:base64string`)
- **API Key** (format: `essu_...`)
- **APM Endpoint** (format: `https://xxx.apm.region.cloud.com:443`)
- **Authorization Token** (format: `Bearer xxxxx`)

### Tools

- Helm 3.x (for operator installation)
- `curl` (for traffic generation)
- Basic knowledge of Kubernetes/OpenShift

---

## Part 1: Install OpenTelemetry Operator

### Step 1.1: Create Operator Namespace

```bash
oc new-project opentelemetry-operator-system
```

### Step 1.2: Download Values Configuration

Create `values.yaml`:

```yaml
opentelemetry-operator:
  manager:
    extraArgs:
      - --enable-go-instrumentation
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
          - ALL
      seccompProfile:
        type: RuntimeDefault
  
  kubeRBACProxy:
    enabled: false
  
  # Allow OpenShift SCC to inject user IDs
  securityContext:
    runAsUser: null
    runAsGroup: null
    fsGroup: null
    runAsNonRoot: null
```

### Step 1.3: Create Elastic Secret

Replace these values with your Elastic Cloud credentials:

```bash
# Set your Elastic Cloud credentials
ELASTIC_ENDPOINT="https://YOUR_DEPLOYMENT_ID.apm.YOUR_REGION.aws.elastic-cloud.com:443"
ELASTIC_API_KEY="YOUR_API_KEY"

# Create the secret
oc create secret generic elastic-secret-otel \
  --namespace opentelemetry-operator-system \
  --from-literal=elastic_endpoint="$ELASTIC_ENDPOINT" \
  --from-literal=elastic_api_key="$ELASTIC_API_KEY"
```

### Step 1.4: Install Operator via Helm

```bash
# Add Helm repository
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

# Install the operator
helm install opentelemetry-kube-stack open-telemetry/opentelemetry-kube-stack \
  --namespace opentelemetry-operator-system \
  --values values.yaml \
  --version 0.3.9
```

### Step 1.5: Grant Daemon Collector Permissions

The daemon collector needs elevated permissions for host-level monitoring:

```bash
# Grant privileged SCC (if allowed in your environment)
oc adm policy add-scc-to-user privileged \
  -z opentelemetry-kube-stack-daemon-collector \
  -n opentelemetry-operator-system
```

#### Alternative: Custom SCC

If privileged SCC is not allowed, create a custom SCC (requires OpenShift Admin):

Save as `otel-daemon-scc.yaml`:

```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: otel-daemon-collector
allowHostDirVolumePlugin: true
allowHostIPC: false
allowHostNetwork: true
allowHostPID: true
allowHostPorts: true
allowPrivilegeEscalation: false
allowPrivilegedContainer: false
fsGroup:
  type: RunAsAny
runAsUser:
  type: RunAsAny
seLinuxContext:
  type: RunAsAny
supplementalGroups:
  type: RunAsAny
volumes:
  - configMap
  - emptyDir
  - hostPath
  - projected
```

Apply and bind:

```bash
oc apply -f otel-daemon-scc.yaml
oc adm policy add-scc-to-user otel-daemon-collector \
  -z opentelemetry-kube-stack-daemon-collector \
  -n opentelemetry-operator-system
```

### Step 1.6: Verify Operator Installation

```bash
# Check operator pod
oc get pods -n opentelemetry-operator-system

# Expected output:
# NAME                                                         READY   STATUS
# opentelemetry-operator-controller-manager-xxxxx              2/2     Running
# opentelemetry-kube-stack-daemon-collector-xxxxx              1/1     Running
# opentelemetry-kube-stack-deployment-collector-xxxxx          1/1     Running

# Check logs
oc logs -n opentelemetry-operator-system \
  -l app.kubernetes.io/name=opentelemetry-operator --tail=50
```

---

## Part 2: Deploy OpenTelemetry Demo Application

### Step 2.1: Create Demo Namespace

```bash
oc new-project otel-demo
```

### Step 2.3: Deploy Application

Use the adjusted apps for OpenShift:

```bash
oc apply -f otel-demo-app.yaml
```

This deploys:

- 11+ microservices
- Redis for cart service
- Load generator (automatic traffic)
- All necessary services and networking

### Step 2.4: Monitor Deployment

```bash
# Watch pods starting
watch oc get pods -n otel-demo
```

### Step 2.5: Expose Frontend

```bash
# Create route
oc expose service frontend-external -n otel-demo

# Get URL
oc get route frontend-external -n otel-demo
```

---

## Part 3: Configure Instrumentation

### Step 3.1: Create Instrumentation Resource

Create `instrumentation.yaml`:

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: otel-demo-instrumentation
  namespace: otel-demo
spec:
  exporter:
    endpoint: https://YOUR_DEPLOYMENT.apm.YOUR_REGION.aws.elastic-cloud.com:443
  env:
    - name: OTEL_EXPORTER_OTLP_HEADERS
      value: "Authorization=Bearer YOUR_TOKEN"
  propagators:
    - tracecontext
    - baggage
    - b3
  sampler:
    type: parentbased_traceidratio
    argument: "1.0"
  java:
    env:
      - name: OTEL_JAVAAGENT_DEBUG
        value: "false"
  nodejs:
    env:
      - name: OTEL_NODEJS_DEBUG
        value: "false"
  python:
    env:
      - name: OTEL_PYTHON_LOG_LEVEL
        value: "info"
  dotnet:
    env:
      - name: OTEL_DOTNET_AUTO_LOG_DIRECTORY
        value: "/tmp"
```

> **Important:** Replace `YOUR_DEPLOYMENT`, `YOUR_REGION`, and `YOUR_TOKEN` with your actual Elastic Cloud values.

Apply:

```bash
oc apply -f instrumentation.yaml
```

### Step 3.2: Verify Instrumentation

```bash
# Check instrumentation resource
oc get instrumentation -n otel-demo

# Check for auto-instrumentation annotations
oc get pods -n otel-demo -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.instrumentation\.opentelemetry\.io/inject-*}{"\n"}{end}' | grep true

# Expected output shows services with instrumentation annotations:
# adservice-xxxxx    instrumentation.opentelemetry.io/inject-java":"true
# cartservice-xxxxx  instrumentation.opentelemetry.io/inject-dotnet":"true
# paymentservice-xxxxx  instrumentation.opentelemetry.io/inject-nodejs":"true
```

### Step 3.3: Check Init Containers

Auto-instrumentation injects init containers:

```bash
# Check a Python service (recommendationservice)
oc describe pod -n otel-demo -l app=recommendationservice | grep -A5 "Init Containers"
# Expected: opentelemetry-auto-instrumentation-python

# Check a Java service (adservice)
oc describe pod -n otel-demo -l app=adservice | grep -A5 "Init Containers"
# Expected: opentelemetry-auto-instrumentation-java
```

### Step 3.4: Verify OTEL Environment Variables

```bash
# Check environment variables in an instrumented pod
oc exec -n otel-demo deployment/paymentservice -- env | grep OTEL

# Expected output:
# OTEL_EXPORTER_OTLP_ENDPOINT=https://...
# OTEL_EXPORTER_OTLP_HEADERS=Authorization=Bearer ...
# OTEL_SERVICE_NAME=paymentservice
# OTEL_RESOURCE_ATTRIBUTES=...
```

---

## Part 4: Generate Realistic Traffic

### Option A: Use Built-in Load Generator

The demo includes a load generator that automatically creates traffic:

```bash
# Check load generator status
oc logs -n otel-demo deployment/loadgenerator -f

# You should see Locust generating requests automatically
```

### Option B: Manual Realistic Traffic

Create `generate-realistic-traffic.sh`:

```bash
#!/bin/bash
# Realistic traffic generator

ROUTE_URL=$(oc get route frontend-external -n otel-demo -o jsonpath='{.spec.host}')

PRODUCTS=(
  "0PUK6V6EV0" "1YMWWN1N4O" "2ZYFJ3GM2N" "66VCHSJNUP" "6E92ZMYYFZ"
  "9SIQT8TOJO" "L9ECAV7KIM" "LS4PSXUNUM" "OLJCESPC7Z"
)

echo "Generating realistic traffic to http://$ROUTE_URL"
echo "Press Ctrl+C to stop"
echo ""

session=0
while true; do
  session=$((session + 1))
  
  # Determine user behavior (60% browse, 25% abandon cart, 15% purchase)
  rand=$((RANDOM % 100))
  
  if [ $rand -lt 60 ]; then
    # Browser - just views pages
    echo "[$session] 👀 Browser: viewing products..."
    curl -s "http://$ROUTE_URL" > /dev/null
    sleep 2
    
    product=${PRODUCTS[$RANDOM % ${#PRODUCTS[@]}]}
    curl -s "http://$ROUTE_URL/product/$product" > /dev/null
    echo "  ✓ Viewed $product"
    sleep $((RANDOM % 5 + 3))
    
  elif [ $rand -lt 85 ]; then
    # Cart abandoner - adds items but doesn't buy
    echo "[$session] 🛒 Cart Abandoner: adding items..."
    curl -s "http://$ROUTE_URL" > /dev/null
    
    product=${PRODUCTS[$RANDOM % ${#PRODUCTS[@]}]}
    curl -s "http://$ROUTE_URL/product/$product" > /dev/null
    curl -s -X POST "http://$ROUTE_URL/cart" -d "product_id=$product&quantity=1" > /dev/null
    echo "  ✓ Added $product to cart"
    sleep 3
    
    curl -s "http://$ROUTE_URL/cart" > /dev/null
    echo "  → Abandoned cart"
    sleep $((RANDOM % 5 + 2))
    
  else
    # Buyer - completes purchase (FULL TRACE!)
    echo "[$session] 💳 Buyer: completing purchase..."
    curl -s "http://$ROUTE_URL" > /dev/null
    
    product=${PRODUCTS[$RANDOM % ${#PRODUCTS[@]}]}
    curl -s "http://$ROUTE_URL/product/$product" > /dev/null
    curl -s -X POST "http://$ROUTE_URL/cart" -d "product_id=$product&quantity=1" > /dev/null
    echo "  ✓ Added $product"
    sleep 2
    
    curl -s "http://$ROUTE_URL/cart" > /dev/null
    echo "  ✓ Viewing cart"
    sleep 3
    
    # Checkout - generates complete distributed trace
    HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" -X POST "http://$ROUTE_URL/cart/checkout" \
      -d "email=user$session@example.com" \
      -d "street_address=123 Main St" \
      -d "zip_code=12345" \
      -d "city=TestCity" \
      -d "state=CA" \
      -d "country=US" \
      -d "credit_card_number=4111111111111111" \
      -d "credit_card_expiration_month=12" \
      -d "credit_card_expiration_year=2026" \
      -d "credit_card_cvv=123")
    
    if [ "$HTTP_CODE" = "200" ] || [ "$HTTP_CODE" = "303" ]; then
      echo "  ✅ PURCHASE COMPLETE! (Full distributed trace)"
    else
      echo "  ❌ Checkout failed (HTTP $HTTP_CODE)"
    fi
    sleep $((RANDOM % 10 + 5))
  fi
  
  echo ""
  sleep $((RANDOM % 10 + 5))
done
```

Run it:

```bash
chmod +x generate-realistic-traffic.sh
./generate-realistic-traffic.sh
```

### Traffic Patterns Generated

The script creates realistic e-commerce patterns:

- **Homepage Views** → frontend, adservice
- **Product Browsing** → frontend, productcatalogservice, recommendationservice
- **Add to Cart** → frontend, cartservice, redis
- **View Cart** → frontend, cartservice, productcatalogservice
- **Checkout** → frontend, checkoutservice, cartservice, paymentservice, shippingservice, emailservice, productcatalogservice, quoteservice

> 💡 The checkout flow generates the most complete distributed traces!

---

## Part 5: Verify Telemetry in Elastic Cloud

### Step 5.1: Access Elastic Cloud APM

1. Log into Elastic Cloud: https://cloud.elastic.co
2. Select your deployment
3. Click **Kibana** button
4. Navigate to **Observability → APM** (or **Applications**)

### Step 5.2: Wait for Data

- Initial data appears in **2-3 minutes** after traffic starts
- Full traces may take **3-5 minutes**
- The load generator runs continuously

### Step 5.3: Verify Services

You should see these services in APM:

#### Auto-Instrumented (5):
- `adservice` (Java)
- `cartservice` (.NET)
- `paymentservice` (Node.js)
- `recommendationservice` (Python)
- `loadgenerator` (Python)

#### SDK-Instrumented (6):
- `frontend` (Go)
- `checkoutservice` (Go)
- `productcatalogservice` (Go)
- `emailservice` (Ruby)
- `shippingservice` (Rust)
- `quoteservice` (PHP)

**Total:** 11 services with full telemetry

### Step 5.4: Explore Service Map

1. In APM, click **Service Map**
2. You should see a graph showing service dependencies
3. Hover over connections to see call rates

### Step 5.5: View Distributed Traces

1. Click on `checkoutservice` (generates the longest traces)
2. Click **Transactions**
3. Click on any transaction
4. You'll see a waterfall view showing:
   - Timeline of each service call
   - Duration of each span
   - Service-to-service propagation
   - Total request time

### Step 5.6: Check Metrics

For each service, you can view:

- **Latency** (response time percentiles: p50, p95, p99)
- **Throughput** (requests per minute)
- **Error rate** (percentage of failed requests)
- **Instance count**

---

## Part 6: Advanced Testing

### Test 1: Generate Errors

Create some intentional errors to test error tracking:

```bash
# Try invalid product
curl "http://$(oc get route frontend-external -n otel-demo -o jsonpath='{.spec.host}')/product/INVALID"

# Try invalid checkout
curl -X POST "http://$(oc get route frontend-external -n otel-demo -o jsonpath='{.spec.host}')/cart/checkout"
```

Check APM for error traces (red in the UI).

### Test 2: Load Testing

Generate high load to test performance monitoring:

```bash
# Parallel requests
for i in {1..10}; do
  (for j in {1..10}; do
    curl -s "http://$(oc get route frontend-external -n otel-demo -o jsonpath='{.spec.host}')" > /dev/null
  done) &
done
wait

echo "Generated 100 concurrent requests"
```

Watch latency metrics increase in APM.

### Test 3: Service Dependency Analysis

1. In APM, select **Dependencies** tab for a service
2. See upstream (services calling it) and downstream (services it calls)
3. Identify bottlenecks or problematic dependencies

### Test 4: Custom Queries

In Kibana Discover, try these queries:

```
# All APM data
event.dataset: "apm*"

# Specific service
service.name: "checkoutservice"

# Errors only
event.outcome: "failure"

# Slow transactions (>500ms)
transaction.duration.us > 500000

# Specific trace ID
trace.id: "YOUR_TRACE_ID"
```

---

## Troubleshooting

### Issue: No Services in APM

**Check 1:** Verify network connectivity

```bash
oc exec -n otel-demo deployment/paymentservice -- \
  nc -zv YOUR_DEPLOYMENT.apm.YOUR_REGION.aws.elastic-cloud.com 443
```

**Check 2:** Verify OTEL environment variables

```bash
oc exec -n otel-demo deployment/paymentservice -- env | grep OTEL
```

**Check 3:** Check for errors in logs

```bash
oc logs -n otel-demo deployment/paymentservice | grep -i "error\|fail"
```

### Issue: Some Services Missing

**Check pod status:**

```bash
oc get pods -n otel-demo
```

**Check instrumentation:**

```bash
oc describe pod -n otel-demo <pod-name> | grep -A10 "Init Containers"
```

**Check operator logs:**

```bash
oc logs -n opentelemetry-operator-system \
  -l app.kubernetes.io/name=opentelemetry-operator --tail=100
```

### Issue: Traces Not Connected

**Check propagators:**

```bash
oc get instrumentation otel-demo-instrumentation -n otel-demo -o yaml | grep propagators
```

Should include:
- `tracecontext`
- `baggage`
- `b3`

### Issue: High Error Rate

**Check service logs:**

```bash
# Check all services
for svc in frontend checkoutservice paymentservice; do
  echo "=== $svc ==="
  oc logs -n otel-demo deployment/$svc --tail=20
done
```

### Debug Commands

```bash
# Check all pod statuses
oc get pods -n otel-demo -o wide

# Check events
oc get events -n otel-demo --sort-by='.lastTimestamp' | tail -20

# Check resource usage
oc top pods -n otel-demo

# Check service endpoints
oc get endpoints -n otel-demo

# Test internal service connectivity
oc run -it --rm debug --image=curlimages/curl --restart=Never -n otel-demo -- \
  curl -v http://paymentservice:8080

# Check instrumentation resource
oc get instrumentation -n otel-demo -o yaml
```

---

## Fix Common OTEL App Issues

### Issue 1: CartService CrashLoopBackOff

**Error:** `VALKEY_ADDR environment variable is required`

**Fix:**

```bash
oc set env deployment/cartservice -n otel-demo \
  VALKEY_ADDR=redis-cart:6379 \
  REDIS_ADDR=redis-cart:6379

oc rollout restart deployment/cartservice -n otel-demo
```

### Issue 2: CheckoutService Can't Connect to Kafka

**Error:** `kafka: client has run out of available brokers`

**Fix:** Make Kafka optional:

```bash
oc set env deployment/checkoutservice -n otel-demo \
  KAFKA_SERVICE_ADDR=""

oc rollout restart deployment/checkoutservice -n otel-demo
```

### Issue 3: CurrencyService C++ Error

**Error:** `basic_string: construction from null is not valid`

**Fix:** Add required environment variables:

```bash
oc set env deployment/currencyservice -n otel-demo \
  CURRENCY_SERVICE_PORT=8080 \
  PORT=8080

oc rollout restart deployment/currencyservice -n otel-demo
```

If still failing, scale down (not critical for demo):

```bash
oc scale deployment currencyservice --replicas=0 -n otel-demo
```

### Issue 4: Kafka and FeatureFlagService Failing

**Fix:** Scale down non-essential services:

```bash
oc scale deployment kafka featureflagservice --replicas=0 -n otel-demo
```

### Quick Fix Script

Run all fixes at once:

Create `fix-otel-demo.sh`:

```bash
#!/bin/bash
# fix-otel-demo.sh

# Fix cartservice
oc set env deployment/cartservice -n otel-demo \
  VALKEY_ADDR=redis-cart:6379 \
  REDIS_ADDR=redis-cart:6379

# Fix checkoutservice
oc set env deployment/checkoutservice -n otel-demo \
  KAFKA_SERVICE_ADDR=""

# Fix currencyservice
oc set env deployment/currencyservice -n otel-demo \
  CURRENCY_SERVICE_PORT=8080 \
  PORT=8080

# Scale down problematic services
oc scale deployment kafka featureflagservice --replicas=0 -n otel-demo

# Restart fixed services
oc rollout restart deployment/cartservice -n otel-demo
oc rollout restart deployment/checkoutservice -n otel-demo
oc rollout restart deployment/currencyservice -n otel-demo

echo "Fixes applied. Waiting for rollout..."
sleep 30

oc get pods -n otel-demo
```

Make executable and run:

```bash
chmod +x fix-otel-demo.sh
./fix-otel-demo.sh
```

---

## Cleanup

### Remove Demo Application

```bash
oc delete project otel-demo
```

### Remove Operator (Optional)

```bash
helm uninstall opentelemetry-kube-stack -n opentelemetry-operator-system
oc delete project opentelemetry-operator-system
```

### Remove Custom SCC (if created)

```bash
oc delete scc otel-daemon-collector
```

---

## Appendix

### Appendix A: Quick Reference

#### Essential Commands

```bash
# Check operator status
oc get pods -n opentelemetry-operator-system

# Check demo app status
oc get pods -n otel-demo

# View service logs
oc logs -n otel-demo deployment/<service-name> -f

# Check instrumentation
oc get instrumentation -n otel-demo

# Get frontend URL
oc get route frontend-external -n otel-demo

# Restart a service
oc rollout restart deployment/<service-name> -n otel-demo

# Scale a service
oc scale deployment/<service-name> --replicas=<count> -n otel-demo
```

### Appendix B: Configuration Examples

#### Complete Instrumentation Resource

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: otel-demo-instrumentation
  namespace: otel-demo
spec:
  exporter:
    endpoint: https://xxx.apm.region.aws.elastic-cloud.com:443
  env:
    - name: OTEL_EXPORTER_OTLP_HEADERS
      value: "Authorization=Bearer YOUR_TOKEN"
    - name: OTEL_EXPORTER_OTLP_PROTOCOL
      value: "grpc"
  propagators:
    - tracecontext
    - baggage
    - b3
  sampler:
    type: parentbased_traceidratio
    argument: "1.0"
  java:
    env:
      - name: OTEL_JAVAAGENT_DEBUG
        value: "false"
      - name: OTEL_INSTRUMENTATION_COMMON_EXPERIMENTAL_CONTROLLER_TELEMETRY_ENABLED
        value: "true"
    resource:
      addK8sUIDAttributes: true
  nodejs:
    env:
      - name: OTEL_NODEJS_DEBUG
        value: "false"
      - name: OTEL_INSTRUMENTATION_HTTP_CAPTURE_HEADERS_SERVER_REQUEST
        value: ".*"
  python:
    env:
      - name: OTEL_PYTHON_LOG_LEVEL
        value: "info"
      - name: OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED
        value: "true"
  dotnet:
    env:
      - name: OTEL_DOTNET_AUTO_LOG_DIRECTORY
        value: "/tmp"
      - name: OTEL_DOTNET_AUTO_TRACES_CONSOLE_EXPORTER_ENABLED
        value: "false"
```
---

## License

This guide is provided as-is for educational and demonstration purposes.

## Contributing

Contributions are welcome! Please submit issues or pull requests.

## Support

For issues related to:
- **OpenTelemetry:** Visit [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- **Elastic APM:** Visit [Elastic Observability Documentation](https://www.elastic.co/guide/en/observability/)
- **OpenShift:** Visit [OpenShift Documentation](https://docs.openshift.com/)


