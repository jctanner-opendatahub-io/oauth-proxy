# Issue #1: OAuth Proxy Cannot Handle Certificate Hostname Mismatches

## Problem Description

The OpenShift OAuth Proxy fails when connecting to upstream HTTPS services that have certificate hostname mismatches, specifically with wildcard certificates that don't cover multi-level subdomains.

## Error Details

```
2025/07/23 16:02:05 reverseproxy.go:667: http: proxy error: tls: failed to verify certificate: x509: certificate is valid for *.apps-crc.testing, not odh-gateway.odh.apps-crc.testing
```

**Analysis:**
- **Certificate covers**: `*.apps-crc.testing` (single-level wildcard)
- **Trying to connect to**: `odh-gateway.odh.apps-crc.testing`
- **Problem**: Single-level wildcards (`*.apps-crc.testing`) don't match multi-level subdomains (`odh-gateway.odh.apps-crc.testing`)

## Root Cause

### Current TLS Configuration Limitations

**File**: `oauthproxy.go`, lines 124-131
```go
if len(opts.UpstreamCAs) > 0 {
    pool, err := util.GetCertPool(opts.UpstreamCAs, false)
    if err != nil {
        return nil, err
    }
    transport.TLSClientConfig = oscrypto.SecureTLSConfig(&tls.Config{RootCAs: pool})
}
```

**What's Missing:**
- ❌ No `ServerName` override capability
- ❌ No per-upstream TLS configuration
- ❌ No hostname verification bypass options
- ❌ No SNI (Server Name Indication) configuration

### Current Configuration Options

The OAuth proxy only supports:
- ✅ `--upstream-ca`: Custom CA certificates
- ⚠️ `--ssl-insecure-skip-verify`: Global certificate verification skip (likely ineffective for upstream connections)

### Why `--ssl-insecure-skip-verify` Doesn't Help

**Code Analysis** (`options.go` lines 321-326):
```go
if o.SSLInsecureSkipVerify {
    insecureTransport := &http.Transport{
        TLSClientConfig: oscrypto.SecureTLSConfig(&tls.Config{InsecureSkipVerify: true}),
    }
    http.DefaultClient = &http.Client{Transport: insecureTransport}  // Only affects DefaultClient
}
```

**Upstream Connection Code** (`oauthproxy.go` lines 115-125):
```go
transport := http.DefaultTransport.(*http.Transport).Clone()  // Uses DefaultTransport, not DefaultClient
```

**Result**: The flag modifies `http.DefaultClient` but upstream connections use a clone of `http.DefaultTransport` - these are separate transports in Go.

## Impact

- **OpenShift CRC environments**: Common issue due to `*.apps-crc.testing` wildcard certificates
- **Multi-level subdomain services**: Any service with hostnames like `service.namespace.domain.com`
- **Development environments**: Self-signed or improperly configured certificates
- **Enterprise environments**: Complex certificate hierarchies

## Current Workarounds

### 1. Global Certificate Verification Skip ⚠️ **NOT RECOMMENDED & LIKELY INEFFECTIVE**
```bash
./oauth-proxy --upstream=https://odh-gateway.odh.apps-crc.testing --ssl-insecure-skip-verify
```
**Problems:**
- **Likely doesn't fix upstream certificate errors** (only affects `http.DefaultClient`, not upstream proxy transport)
- Affects OAuth server connections (security risk)
- May not work for the specific hostname mismatch issue
- **Needs testing to confirm behavior**

### 2. Fix Certificate/Hostname Matching ✅ **RECOMMENDED**
```bash
# Option A: Use hostname that matches certificate
--upstream=https://odh-gateway-svc.apps-crc.testing

# Option B: Get proper certificate covering the actual hostname
# Certificate should include: odh-gateway.odh.apps-crc.testing
```

### 3. Custom CA Certificate (if self-signed)
```bash
--upstream-ca=/path/to/custom-ca.crt
```

## Proposed Solutions

### Short-term: Configuration Enhancement

Add per-upstream TLS configuration options:

```bash
# Proposed new flags (not implemented)
--upstream-tls-server-name=<hostname>     # Override ServerName for certificate verification
--upstream-tls-skip-verify=<true/false>   # Per-upstream skip verification
--upstream-tls-ca=<path>                  # Per-upstream CA (instead of global)
```

### Medium-term: URL-based Configuration

Support TLS parameters in upstream URLs:
```bash
# Proposed syntax (not implemented)
--upstream="https://service:8443?tls-server-name=matching.hostname.com&tls-skip-verify=false"
```

### Long-term: Complete TLS Configuration Overhaul

**File modifications needed**: `oauthproxy.go`, `options.go`

```go
// Proposed enhancement (not implemented)
type UpstreamTLSConfig struct {
    ServerName         string
    InsecureSkipVerify bool
    CAFile             string
    CertFile           string
    KeyFile            string
}

// In NewReverseProxy function
transport.TLSClientConfig = &tls.Config{
    RootCAs:            pool,
    ServerName:         upstreamConfig.ServerName,    // New: hostname override
    InsecureSkipVerify: upstreamConfig.InsecureSkipVerify, // New: per-upstream skip
}
```

## Code Locations for Enhancement

### Files to Modify:
1. **`oauthproxy.go`**:
   - `NewReverseProxy()` function (lines 112-137)
   - `NewWebSocketOrRestReverseProxy()` function (lines 186-191)

2. **`options.go`**:
   - Add new TLS configuration options (around line 94)
   - Add validation logic (around line 321)

3. **`main.go`**:
   - Add new command-line flags (around line 103)

### Key Functions:
- `NewReverseProxy(target *url.URL, opts *Options)` - HTTP proxy TLS config
- `NewWebSocketOrRestReverseProxy()` - WebSocket proxy TLS config
- `Options.Validate()` - Configuration validation

## OpenShift CRC Specific Notes

**Context**: OpenShift CodeReady Containers provides wildcard certificates for `*.apps-crc.testing`

**Common scenarios**:
- Services: `service-name.project-name.apps-crc.testing` (fails with current cert)
- Routes: `route-name.apps-crc.testing` (works with current cert)

**CRC Workaround**: Configure routes to use single-level subdomains instead of multi-level.

## Security Considerations

1. **Per-upstream configuration**: Allows granular security control
2. **ServerName override**: Enables certificate reuse while maintaining verification
3. **Avoid global skip**: Current `--ssl-insecure-skip-verify` is too broad
4. **CA certificate validation**: Should remain default behavior

## Testing Requirements

When implementing solutions, test scenarios should include:
- Single-level wildcard certificates (`*.domain.com`)
- Multi-level wildcard certificates (`*.*.domain.com`)
- Self-signed certificates
- Certificate chain validation
- SNI scenarios
- Mixed HTTP/HTTPS upstreams

## Priority

**High** - This affects common deployment scenarios, especially in development environments using OpenShift CRC and enterprise environments with complex certificate hierarchies.

## References

- Related error in `vendor/net/http/transport.go` (Go standard library)
- TLS configuration in `github.com/openshift/library-go/pkg/crypto`
- Certificate validation standards (RFC 6125) 