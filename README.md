# Envoy

To reproduce an issue where envoy fails to pick up the correct certificate from the trust store, we need to setup a
simple envoy configuration with a cluster and a listener.

There are multiple scenarios, which demonstrates the issue:

* [Trust Expired Cert **Last**](#trust-expired-cert-last) Envoy fails unexpectedly (curl succeeds)
* [Trust Expired Cert **First**](#trust-expired-cert-first) Envoy works as expected (curl fails)
* [Trust Expired Cert Only](#trust-expired-cert-only) Fails as expected.
* [Trust Valid Cert Only](#trust-valid-cert-only) Works as expected.

## Envoy v1.40.2 Results

This branch runs the reproducer against `envoyproxy/envoy:v1.40.2`, which includes
[envoyproxy/envoy#47511](https://github.com/envoyproxy/envoy/pull/47511) ("tls: always treat trusted_ca as a set of certs").
The original results below were recorded with `v1.33.0`.

| Scenario              | v1.33.0 `/certs` | v1.33.0 health      | v1.40.2 `/certs` | v1.40.2 health      |
|-----------------------|------------------|---------------------|------------------|---------------------|
| Expired cert **last** | valid only       | `/failed_active_hc` | both             | `healthy`           |
| Expired cert **first**| expired only     | `healthy`           | both             | `/failed_active_hc` |
| Valid cert only       | valid            | `healthy`           | valid            | `healthy`           |
| Expired cert only     | expired          | `/failed_active_hc` | expired          | `/failed_active_hc` |

Observations for v1.40.2:

* `/certs` now lists all certificates of the trust bundle (in file order). **Fixed.**
* Validation is still order dependent, but the failing order flipped: now the expired cert **first** fails with
  `verify cert failed: X509_verify_cert: certificate verification error at depth 1: certificate has expired`.
* No warning is logged when loading a trust bundle with two CA certs sharing subject and public key.
* `ssl.certificate.validation_context_sds.expiration_unix_time_seconds` reports the earliest expiration in the bundle
  (`1664582400`, i.e. the expired cert) in both orders, so it does not indicate which order actually fails.

Note: curl 8.18 / OpenSSL 3.5 rejects the intermediates with `invalid CA certificate (79)` since they lack
`basicConstraints=CA:TRUE`, so the curl comparisons below no longer reproduce with recent curl versions.

## Setup

Start two containers with docker compose:

```bash
mkdir -p ca
docker compose start 
```

There is a `client` container serving traffic on port 8000 employing the `certs/valid.crt` certificate.

Check the certificate chain with curl:

```bash
curl https://localhost:8000/ --cacert certs/valid.crt
```

Which should print: `Hello World!` (without failing on certificate validation)

## Trust Expired Cert **Last**

Now put the expired 

```bash
rm -f logs/health_check_events.log

cat certs/valid.crt > ca/ca.crt
cat certs/expired.crt >> ca/ca.crt
docker compose restart 
```

Check the certificate chain with curl:

```bash
curl https://localhost:8000/ --cacert ca/ca.crt
```

Which should print: `Hello World!` (without failing on certificate validation)
So curl is preferring the valid certificate.

Check that envoy has picked up both certificates in the trust store (repeat call if response is empty):

```bash
curl http://localhost:8001/certs
```

Envoy has only one certificate in the trust store, and seems to prefer the valid one.

Check health check logs:

```bash
cat logs/health_check_events.log
```

Check health check on cluster:

```bash
curl --silent --fail http://localhost:8001/clusters | grep health_flags -B 1
```

This prints out:

```txt
8000-cluster::172.18.0.3:8000::hostname::server
8000-cluster::172.18.0.3:8000::health_flags::/failed_active_hc
```

Inspect trace logs

```bash
docker compose logs | tail | grep -E 'default_validator|ssl_handshaker'
```

which show: 


```txt
[debug][connection] [source/common/tls/cert_validator/default_validator.cc:339] verify cert failed: X509_verify_cert: certificate verification error at depth 1: certificate has expired
[trace][connection] [source/common/tls/ssl_handshaker.cc:149] [Tags: "ConnectionId":"59"] ssl error occurred while read: SSL
```

## Trust Expired Cert **First**

Define the trust store from the certificates (docker compose watches the `./ca` folder and restarts the server container):

```bash
rm -f logs/health_check_events.log

cat certs/expired.crt > ca/ca.crt
cat certs/valid.crt >> ca/ca.crt
docker compose restart 
```

Check the certificate chain with curl:

```bash
curl https://localhost:8000/ --cacert ca/ca.crt
```

Which prints: `curl: (60) SSL certificate problem: certificate has expired`
So curl is preferring the expired certificate.

Check which certificate envoy has picked up (repeat call if response is empty):

```bash
curl http://localhost:8001/certs
```

The `ca_cert` contains only the expired certificate and shows `"days_until_expiration": "0"`.

Envoy has only one certificate in the trust store, and seems to prefer the expired one.

Check health check logs:

```bash
cat logs/health_check_events.log
```

Check health on cluster:

```bash
curl --silent --fail http://localhost:8001/clusters | grep health_flags -B 1
```

This prints out:

```txt
8000-cluster::127.0.0.1:8000::hostname::localhost
8000-cluster::127.0.0.1:8000::health_flags::healthy
```

## Trust Valid Cert Only

To verify the valid certificate works as expected when the expired certificate is not in the trust store run:

```bash
rm -f logs/health_check_events.log
cat certs/valid.crt > ca/ca.crt
docker compose restart 
```

Check health check on cluster (repeat until healthy):

```bash
curl --silent --fail http://localhost:8001/clusters | grep health_flags -B 1
```

Should print out:

```txt
8000-cluster::127.0.0.1:8000::hostname::localhost
8000-cluster::127.0.0.1:8000::health_flags::healthy
```

## Trust Expired Cert Only

And to cross-check that the expired certificate is not trusted, run:

```bash
rm -f logs/health_check_events.log
cat certs/expired.crt > ca/ca.crt
docker compose restart 
```

Check health check on cluster:

```bash
curl --silent --fail http://localhost:8001/clusters | grep health_flags -B 1
```

This prints out:

```txt
8000-cluster::127.0.0.1:8000::hostname::localhost
8000-cluster::127.0.0.1:8000::health_flags::/failed_active_hc
```

### Tear Down

```bash
docker compose stop
```
