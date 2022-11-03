# CVE-2022-31692 Demo

## Overview
A simple Spring Boot application demonstrating configuration that is vulnerable to [CVE-2022-31692](https://tanzu.vmware.com/security/cve-2022-31692).

This vulnerability may attract attention due to its severity - it has a CVSS 3.x base score of 9.8 as it allows authentication bypass. 
The purpose of this project is to demonstrate the conditions described in the advisory, which lead to the vulnerability being applicable.
- The application expects that Spring Security applies security to forward and include dispatcher types.
- The application uses the `AuthorizationFilter` either manually or via the `authorizeHttpRequests()` method.
- The application configures the `FilterChainProxy` to apply to forward and/or include requests (e.g. `spring.security.filter.dispatcher-types = request, error, async, forward, include`).
- The application may forward or include the request to a higher privilege-secured endpoint.
- The application configures Spring Security to apply to every dispatcher type via `authorizeHttpRequests().shouldFilterAllDispatcherTypes(true)`

For reference, I'm pretty sure [this](https://github.com/spring-projects/spring-security/commit/1f481aafff14f324ffe2b43a973d3d5f54ae92d4) is the commit 
that addresses the vulnerability.

## Demonstration
The application has three URLs:
1. `/` The index page
2. `/admin` An admin page, which requires the user to provide Basic auth (creds "user"/"pass") and be assigned the ROLE_ADMIN role
3. `/forward` A server-side forward to the admin page

Access controls are specified via authorizeHttpRequests() in the SecurityConfig class.

	.authorizeHttpRequests((authz) -> authz
		.antMatchers("/").permitAll()
		.antMatchers("/forward").permitAll()
		.antMatchers("/admin").hasAuthority("ROLE_ADMIN")
		.shouldFilterAllDispatcherTypes(true)
	)

### Expected behaviours

1. User accesses `/` and is not authenticated (thanks to `permitAll()`)

2. User accesses `/admin` . They don't provide authentication, and the request is rejected (401 Not authorized).

3. User accesses `/admin` . They provide valid authentication, but the request is still rejected (403 Unauthorised) 
because they do not have the required role `.hasAuthority("ROLE_ADMIN")`.

4. User accesses `/forward`. Their requests passes through the security filter chain for GET /forward, which passes 
as valid (thanks to `permitAll()`). The controller processes the request, and returns `forward:/admin` to the Dispatcher. 
As instructed by the `spring.security.filter.dispatcher-types` and `.shouldFilterAllDispatcherTypes(true)` settings, 
this is a FORWARD type, so should be passed through the filter chain again. This second pass through the filter results 
in the request being rejected (again, thanks to `hasAuthority("ROLE_ADMIN")`).

### Actual behaviour
User accesses `/forward`, the request is passed through the filter chain once, and passes as valid. The forward is 
processed, but instead of being passed through the chain again, it is just passed as valid, and the admin page is
returned.
