# Questions for Clarification

## Priority 1 - Blocking Questions
*No blocking questions at this time - stack has been decided*

## Priority 2 - Important Clarifications

### Deployment Environment
1. ~~What is the target deployment environment (AWS, Azure, on-premise)?~~ **ANSWERED**: Standard cloud deployment (AWS/Azure/GCP) - see ASSUMPTIONS.md
2. ~~Are there any corporate proxy or firewall considerations for WebSocket connections?~~ **RESOLVED**: Using Server-Sent Events instead, no proxy issues - see DECISIONS.md
3. ~~What monitoring/logging infrastructure is available (e.g., ELK stack, Datadog)?~~ **ANSWERED**: Standard cloud provider tools - see ASSUMPTIONS.md

### Authentication & Security
1. Should we add optional authentication in future phases? **FUTURE**: Not required for Phase 1
2. ~~Are there any corporate security policies we need to comply with?~~ **ANSWERED**: Standard web app security only - see ASSUMPTIONS.md
3. ~~Should session links be time-limited or single-use for security?~~ **ANSWERED**: GUID-based with 10min timeout - see DECISIONS.md

### Branding & Design
1. ~~Are there corporate branding guidelines to follow?~~ **ANSWERED**: Not required for internal tool - see ASSUMPTIONS.md
2. ~~Do you have preference for the avatar icon set/style?~~ **ANSWERED**: Generic avatar icons acceptable - see ASSUMPTIONS.md  
3. ~~Should the app have a specific color scheme or use Material Design defaults?~~ **ANSWERED**: Standard Material Design colors - see ASSUMPTIONS.md

## Priority 3 - Nice to Have Clarifications

### Features
1. Should we add a chat feature for discussion during sessions?
2. Would you like session templates for common estimation patterns?
3. Should we support custom card sequences beyond Fibonacci?

### Analytics
1. Do you want anonymous usage analytics collected?
2. Should we track estimation accuracy over time in future phases?
3. Would session replay functionality be valuable?

### Internationalization
1. Will the app need to support multiple languages?
2. Are there regional differences in estimation practices to consider?

---

*Note: These questions are not blocking initial development. We can proceed with assumptions documented and refine based on answers.*