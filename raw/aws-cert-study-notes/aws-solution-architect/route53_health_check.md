# Route 53 Health Checks

Amazon Route 53 Health Checks are a critical feature used to monitor the availability and health of endpoints, supporting advanced DNS architectures and ensuring resilient application delivery. These checks are **configured separately from DNS records** and are utilized by those records to determine whether to route traffic to a healthy endpoint.

![[R53_health_check_infographic.png]]

## Core Concepts and Functionality

- **Health Checks are independent of DNS records:**  
  They are created and managed separately, then associated with Route 53 records to influence DNS response behavior.

- **Global Distribution of Health Checkers:**  
  Health checks are conducted by a fleet of globally distributed health checkers. Targets must allow incoming health checks from these sources to avoid false alarms caused by blocking these health checkers.
  ![[R53-health-checkers.png]]

- **Targets for Health Checks:**  
  Health checks can monitor any publicly accessible endpoint with an IP address, not limited to AWS resources.

- **Check Frequency:**  
  By default, health checks run every 30 seconds, with an option to increase the frequency to every 10 seconds at an additional cost.

## Types of Health Checks and Their Mechanisms

| **Health Check Type**               | **Description**                                                                                                                 | **Timing Requirements**                             | **Additional Features**                                                   |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------- | ------------------------------------------------------------------------- |
| **TCP Check**                       | Attempts to establish a TCP connection within 10 seconds.                                                                       | Must succeed within 10 seconds.                     | Basic connectivity check.                                                 |
| **HTTP Check**                      | Establishes TCP connection within 4 seconds, then expects HTTP status 200-399 within 2 seconds.                                 | TCP: 4 sec, HTTP response: 2 sec                    | More accurate for web apps than TCP check.                                |
| **HTTP/HTTPS with String Matching** | Same as HTTP Check but also verifies presence of a specified string within first 5,120 bytes of response body within 2 seconds. | TCP: 4 sec, HTTP status: 2 sec, String match: 2 sec | Most accurate check of application health by validating response content. |

### Health Check Types Based on Input

- **Endpoint Health Checks:**  
  Directly check a specified endpoint by IP or domain name.

- **CloudWatch Alarm Checks:**  
  Use CloudWatch alarms as health check inputs, allowing in-depth monitoring via metrics or in-app signals collected by the CloudWatch agent.

- **Calculated Health Checks:**  
  Aggregate the health status of multiple health checks to provide an overall application health status.

## Creating a Health Check (Console Overview)

- Accessed via **Route 53 Console > Health Checks** (not through hosted zones or records).
- Requires naming the health check and selecting the type (Endpoint, CloudWatch, or Calculated).
- For **Endpoint Health Checks**, users specify either an IP address or domain name.
- Protocol selection: TCP, HTTP, or HTTPS.
- Configurable parameters include:
  - **Port number**
  - **Host header** (for virtual hosts)
  - **Request path** (for HTTP/HTTPS)
  - **Request interval** (default 30 sec or fast mode 10 sec)
  - **Failure threshold** (number of consecutive failed checks needed to change status)
  - **String matching** for response validation
- Advanced options:
  - Latency graph to monitor response times.
  - Inversion of health check status (rarely used).
  - Disabling health checks temporarily (useful for maintenance).
  - Selection of health checker regions (recommended or custom).

## Health Check Outcomes and Integration

- Health status is updated based on consecutive pass/fail results.
- If more than **18% of health checkers report healthy**, the overall health check status is healthy; otherwise, it is unhealthy.
- Unhealthy endpoints are generally excluded from DNS query responses to maintain application availability.
- Optional integration with SNS topics allows notifications and external system responses when health checks fail.

## Architectural Role of Health Checks

- Health checks monitor application endpoints globally.
- They influence DNS routing decisions by determining which endpoints are healthy.
- Enable architects and engineers to design fault-tolerant and highly available systems.
- Serve as a foundational operational tool for managing Route 53-based infrastructure and application health.

## Key Takeaways

- **Health Checks are essential for resilient Route 53 DNS routing.**
- They support multiple protocols and detailed application-level validation through string matching.
- They integrate with CloudWatch and can aggregate multiple checks for complex applications.
- Configuration is flexible and must be done independently of DNS records.
- Understanding and implementing health checks is critical for AWS architects, developers, and engineers.
