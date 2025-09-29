# Technical Implementation

## Custom Linux Distribution for ARM64

### Build Automation & Practical Challenges
- Adapted an Automated Linux From Scratch (ALFS) workflow for **ARM64 (aarch64)**.
- Resolved 3–5 manual-intervention points in the ALFS process, including:
  - Cross-compilation toolchain setup for aarch64.
  - Kernel configuration adjustments for ARM-specific hardware.
  - Bootloader (U-Boot) customisation for target devices.
  - Architecture-specific package compilation fixes.
  - Filesystem layout and partition table configuration.

### System Architecture Decisions
- **Minimalist rootfs** based on BusyBox with only essential components.
- **Security-first kernel**: custom-compiled with unnecessary features disabled.
- **ARM64 optimisation** for both embedded and server-class ARM targets.

## .NET Application — Zero External Dependencies
- Full web platform implemented in pure .NET with **no NuGet packages**.
- Developed custom implementations for framework needs:
  - Manual JSON serialization/deserialization.
  - Custom database access layers (no Entity Framework).
  - Self-contained authentication and session management.
  - Native .NET logging and diagnostics.
- Practical trade-off: larger internal codebase but complete transparency and auditability.

## Security Benefits Realised
- Full code transparency and auditability for all platform components.
- Elimination of vulnerability exposure from third-party library updates.
- Reduced attack surface by avoiding complex dependency chains.

## AI-Powered Content Verification System

### Actual Implementation
- Integrated open-source AI models for content analysis (on-device / self-hosted).
- Correlation algorithms that compare incoming content against verified sources.
- Points-based reputation system tracking user contributions and their provenance.
- Dynamic scoring that updates as new verified information becomes available.

### Practical Scope (Built)
- Working content verification and scoring system.
- User reputation and contribution tracking.
- Real-time content quality assessment and points-based rewards.

### Planned Extension (Not Implemented)
- Web3 token conversion of reputation points — design and interfaces prepared, not deployed.
- Blockchain integration for decentralised governance — planned architecture documented.
- We approached the economic-security issues associated with consensus (commonly called the “51% problem”) from a mitigation and management perspective rather than claiming absolute prevention. We treated it as an economic/distribution challenge intrinsic to many distributed-ledger systems and designed tokenomics and governance proposals aimed at reducing centralisation risk.

## Ingress & Provenance — Custom NGINX + GeoIP

### What it provides
- Ingress gateway that performs:
  - GeoIP lookups (country, city, ISO code) via a GeoIP2 DB.
  - Parsing of `X-Forwarded-For` to capture the original client IP (when available), the last hop, and the full XFF chain.
  - Exposure of provenance and geo headers to backends and the verification stack.
  - Structured access logs for audit, attestation and analytics.

### Deployment & repo placement
- Add NGINX infra under `infra/nginx/`:
  - `infra/nginx/nginx.conf` — canonical example config (sanitised).
  - `infra/nginx/README.md` — deployment and GeoIP DB instructions (do **not** commit DB files).
  - `infra/nginx/systemd/` or `infra/nginx/docker/` — unit files / container packaging for deployment.
- Configure `set_real_ip_from` with trusted proxy ranges to avoid trusting forged XFF values.
- Use `real_ip_recursive on` if you explicitly trust the proxy chain in front of NGINX.

### Limitations & privacy
- Accurate last-node identification depends on network path and cooperating proxies. Tor, privacy proxies or forged headers may prevent reliable attribution beyond the immediate TCP peer.
- Log and retention policies must comply with applicable privacy law (e.g. GDPR). Consider anonymisation and strict access controls.

### Example

# load module (if built as dynamic)
load_module modules/ngx_http_geoip2_module.so;

http {
  # GeoIP2 DB
  geoip2 /usr/local/share/GeoIP/GeoLite2-City.mmdb {
    auto_reload 5m;
    $geoip2_country_name country names en;
    $geoip2_country_code country iso_code;
    $geoip2_city_name city names en;
  }

  # Extract first and last entries from X-Forwarded-For
  map $http_x_forwarded_for $xff_first {
    ""       $remote_addr;
    default  $remote_addr;
    "~^(?<first>[^, ]+)" $first;
  }
  map $http_x_forwarded_for $xff_last {
    ""       $remote_addr;
    default  $remote_addr;
    "~(?<last>[^, ]+)$" $last;
  }

  # Real IP settings — list your trusted proxy ranges here
  # (DO NOT set 0.0.0.0/0 on production)
  set_real_ip_from 10.0.0.0/8;
  set_real_ip_from 192.168.0.0/16;
  real_ip_header X-Forwarded-For;
  real_ip_recursive on;

  log_format verifiable '$remote_addr '
                        'original_client="$xff_first" '
                        'last_hop="$xff_last" '
                        'xff="$http_x_forwarded_for" '
                        'country="$geoip2_country_name" '
                        'country_code="$geoip2_country_code" '
                        'city="$geoip2_city_name" '
                        'time="$time_local" '
                        'request="$request" '
                        'upstream="$upstream_addr" '
                        'ua="$http_user_agent"';

  access_log /var/log/nginx/verified_access.log verifiable;

  server {
    listen 80;
    server_name example.com;

    location / {
      proxy_pass http://backend;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Geo-Country $geoip2_country_code;
      proxy_set_header X-Geo-City $geoip2_city_name;
    }
  }
}
