# Tool: Tsunami Security Scanner

> Extensible network security scanner for detecting high-severity vulnerabilities with high confidence.

## Overview

| Attribute | Value |
|-----------|-------|
| **Repository** | [google/tsunami-security-scanner](https://github.com/google/tsunami-security-scanner) |
| **Language** | Java |
| **Category** | Vulnerability Scanner |
| **Maintainer** | Google |

## Philosophy

**Core Insight**: Most vulnerability scanners optimize for **coverage** (find everything). Tsunami optimizes for **precision** (only report real, severe issues). In large environments, false positives waste more time than missed low-severity bugs.

**Design Principles**:
1. **High confidence** — Only report vulnerabilities we're certain exist
2. **High severity** — Focus on RCE, auth bypass, critical issues
3. **Scalable** — Handle thousands of hosts efficiently
4. **Extensible** — Easy to add new vulnerability detectors

**The Trade-off**: Miss some vulns vs. waste time on false positives → Tsunami chooses precision.

## Theoretical Foundations

### Two-Phase Scanning

```
Phase 1: Fingerprinting
  - Port scanning
  - Service detection
  - Version identification
  → Output: List of (host, port, service, version)

Phase 2: Vulnerability Detection
  - For each (service, version)
  - Run relevant detectors
  - Verify exploitation possible
  → Output: Confirmed vulnerabilities only
```

### Detector Philosophy

Traditional scanner:
```
IF version_banner MATCHES vulnerable_version THEN report
PROBLEM: Banner can be wrong, backports exist
```

Tsunami detector:
```
IF can_actually_exploit THEN report
APPROACH: Send payload, verify vulnerable behavior
```

### Confidence Levels

| Level | Meaning | Example |
|-------|---------|---------|
| **Confirmed** | Exploitation verified | Got shell, read file |
| **Firm** | Strong evidence | Error message indicating vuln |
| **Tentative** | Version match only | Banner says vulnerable |

Tsunami aims for **Confirmed** whenever possible.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      TSUNAMI ARCHITECTURE                                │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                        SCANNING WORKFLOW                         │   │
│  │                                                                  │   │
│  │  Target(s) ──► Port Scanner ──► Service Fingerprinter ──►       │   │
│  │                                                                  │   │
│  │               ┌──────────────────────────────────────────┐      │   │
│  │               │         VULNERABILITY DETECTION           │      │   │
│  │               │                                          │      │   │
│  │               │  ┌──────────┐  ┌──────────┐  ┌────────┐ │      │   │
│  │               │  │Detector 1│  │Detector 2│  │  ...   │ │      │   │
│  │               │  │ (Unauthd │  │ (Jenkins │  │        │ │      │   │
│  │               │  │  RCE)    │  │  RCE)    │  │        │ │      │   │
│  │               │  └──────────┘  └──────────┘  └────────┘ │      │   │
│  │               │                                          │      │   │
│  │               └──────────────────────────────────────────┘      │   │
│  │                              │                                   │   │
│  │                              ▼                                   │   │
│  │                      ┌──────────────┐                           │   │
│  │                      │   Results    │                           │   │
│  │                      │  (JSON/Proto)│                           │   │
│  │                      └──────────────┘                           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      PLUGIN SYSTEM                               │   │
│  │                                                                  │   │
│  │  Port Scanners:     NMAP, Masscan (via adapters)                │   │
│  │  Fingerprinters:    Banner grabbing, HTTP headers, etc.         │   │
│  │  Detectors:         Vuln-specific plugins (Java classes)        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

## Detector Design

### Detector Interface
```java
@PluginInfo(
    type = PluginType.VULN_DETECTION,
    name = "ExampleRceDetector",
    version = "0.1",
    description = "Detects Example RCE vulnerability",
    author = "Security Team"
)
public final class ExampleRceDetector implements VulnDetector {

  @Override
  public DetectionReportList detect(
      TargetInfo targetInfo, 
      ImmutableList<NetworkService> matchedServices) {
    
    return matchedServices.stream()
        .filter(this::isServiceVulnerable)
        .map(service -> buildDetectionReport(targetInfo, service))
        .collect(toDetectionReportList());
  }
  
  private boolean isServiceVulnerable(NetworkService service) {
    // Actually test for vulnerability
    // Don't just check version!
  }
}
```

### Detection Report
```java
DetectionReport.newBuilder()
    .setTargetInfo(targetInfo)
    .setNetworkService(service)
    .setDetectionStatus(DetectionStatus.VULNERABILITY_VERIFIED)
    .setVulnerability(
        Vulnerability.newBuilder()
            .setMainId(VulnerabilityId.newBuilder()
                .setPublisher("CVE")
                .setValue("CVE-2021-XXXXX"))
            .setSeverity(Severity.CRITICAL)
            .setTitle("Remote Code Execution in Example Service")
            .setDescription("...")
            .setRecommendation("Upgrade to version X.Y.Z"))
    .build();
```

## Usage

### Installation
```bash
# Clone
git clone https://github.com/google/tsunami-security-scanner

# Build
cd tsunami-security-scanner
./gradlew shadowJar

# Get plugins
git clone https://github.com/google/tsunami-security-scanner-plugins
cd tsunami-security-scanner-plugins/google
./build_all.sh
```

### Basic Scan
```bash
java -cp "tsunami.jar:plugins/*" \
  -Dtsunami.config.location=tsunami.yaml \
  com.google.tsunami.main.cli.TsunamiCli \
  --ip-v4-target=192.168.1.100 \
  --scan-results-local-output-format=JSON \
  --scan-results-local-output-filename=results.json
```

### Configuration (tsunami.yaml)
```yaml
port_scanner:
  name: "nmap"
  nmap:
    port_ranges:
      - "1-65535"

fingerprinter:
  http_fingerprinter:
    enabled: true

plugins:
  vuln_detectors:
    - name: "JenkinsExposedUiDetector"
      enabled: true
    - name: "WordpressExposedInstallPageDetector"
      enabled: true
```

### Docker
```bash
docker run --rm -it \
  -v $(pwd)/results:/results \
  tsunami-scanner \
  --ip-v4-target=192.168.1.100 \
  --scan-results-local-output-format=JSON \
  --scan-results-local-output-filename=/results/scan.json
```

## Available Detectors (Examples)

| Detector | CVE | Severity |
|----------|-----|----------|
| Log4Shell | CVE-2021-44228 | Critical |
| Spring4Shell | CVE-2022-22965 | Critical |
| Jenkins Unauthenticated RCE | CVE-2019-1003000 | Critical |
| Exposed Kubernetes Dashboard | N/A | High |
| WordPress Exposed Setup | N/A | High |
| Exposed Jupyter Notebooks | N/A | High |
| Apache Struts RCE | CVE-2017-5638 | Critical |

## Writing a Detector

### Detector Structure
```java
@PluginInfo(
    type = PluginType.VULN_DETECTION,
    name = "MyNewDetector",
    version = "1.0",
    description = "Detects CVE-XXXX-YYYY",
    author = "Your Name"
)
public final class MyNewDetector implements VulnDetector {

  private static final GoogleLogger logger = GoogleLogger.forEnclosingClass();
  
  private final HttpClient httpClient;

  @Inject
  MyNewDetector(HttpClient httpClient) {
    this.httpClient = checkNotNull(httpClient);
  }

  @Override
  public DetectionReportList detect(
      TargetInfo targetInfo,
      ImmutableList<NetworkService> matchedServices) {
    
    logger.atInfo().log("Starting detection for %s", targetInfo);
    
    return matchedServices.stream()
        .filter(NetworkServiceUtils::isWebService)
        .filter(this::isVulnerable)
        .map(service -> buildReport(targetInfo, service))
        .collect(toDetectionReportList());
  }

  private boolean isVulnerable(NetworkService service) {
    String targetUrl = NetworkServiceUtils.buildWebApplicationRootUrl(service);
    
    try {
      HttpResponse response = httpClient.send(
          HttpRequest.get(targetUrl + "/vulnerable-endpoint")
              .withEmptyHeaders()
              .build());
      
      // Check for vulnerable behavior, not just version
      return response.bodyString()
          .map(body -> body.contains("vulnerable_indicator"))
          .orElse(false);
          
    } catch (IOException e) {
      logger.atWarning().withCause(e).log("Request failed");
      return false;
    }
  }

  private DetectionReport buildReport(
      TargetInfo targetInfo, NetworkService service) {
    return DetectionReport.newBuilder()
        .setTargetInfo(targetInfo)
        .setNetworkService(service)
        .setDetectionStatus(DetectionStatus.VULNERABILITY_VERIFIED)
        .setVulnerability(
            Vulnerability.newBuilder()
                .setMainId(VulnerabilityId.newBuilder()
                    .setPublisher("CVE")
                    .setValue("CVE-XXXX-YYYY"))
                .setSeverity(Severity.CRITICAL)
                .setTitle("Vulnerability Title")
                .setDescription("What the vulnerability allows")
                .setRecommendation("How to fix it"))
        .build();
  }
}
```

## When to Use

- Scanning large networks for critical vulnerabilities
- CI/CD security gates (scan before deploy)
- Compliance scanning (known-vuln check)
- Incident response (check for exploitation)
- When precision matters more than coverage

## Limitations

- **Coverage** — Only detects what plugins exist for
- **Stealth** — Not designed for stealthy scanning
- **Authenticated scanning** — Limited support
- **Custom apps** — No generic web app scanning

## Related Tools

- **Nmap** — Used by Tsunami for port scanning
- **Nuclei** — Template-based scanner (broader coverage, less precision)
- **OpenVAS** — Full-featured vulnerability scanner
- **Nessus** — Commercial vulnerability scanner

## References

- [Tsunami Documentation](https://github.com/google/tsunami-security-scanner/blob/master/docs/howto.md)
- [Writing Detectors](https://github.com/google/tsunami-security-scanner/blob/master/docs/howto.md#build-your-own-plugins)
- [Plugin Repository](https://github.com/google/tsunami-security-scanner-plugins)
- [Google Security Blog Announcement](https://security.googleblog.com/2020/06/tsunami-security-scanning-tool.html)
