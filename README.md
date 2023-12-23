# Spring Boot Performance

## Background
I have heard various claims being made about the performance considerations of technical choices within the Spring Boot ecosystem. These claims are related to the performance of the Servlet stack versus the reactive WebFlux stack. The performance of the standard Java Virtual Machine (JVM) versus GraalVM native images. The performance of the new Project Loom virtual threads supported by JDK 21 and Spring Boot 3.2.The performance of a cold versus a warmed up JVM.

I wanted to test these claims for myself and gain some insights into which technical choices are most likely to lead to the best performance.

## Method of testing
I created two Spring Boot applications with Java 21 and Spring Boot 3.2, one using the Servlet stack and the other using the Webflux stack. Both applications depend on a PostgreSQL database with a Person table containing the fields id, name and description. The applications expose a REST API to Create, Read, Update and Delete (CRUD) person records in the database.

I decided to perform all of my tests on the applications running inside Docker containers as GraalVM native images run inside Docker containers. Using docker containers I could also constrain the CPU cores and memory used by each container to prevent other processes running on the test environment from starving the applications of CPU cores or memory and thus harming the performance. I decided to reserve 1 CPU core and 200Mb RAM for the application containers and limit the container to a maximum of 2 CPU cores and 1Gb RAM.

I am using Docker Compose to run the PostgreSQL database container and the Spring Boot application container. Starting the Spring Boot application container is dependent on the PostgreSQL container being up and healthy. If PostgreSQL is not ready to receive connections, the Spring Boot application may fail to start up.

Project Loom virtual threads can be enabled/disabled using an environment variable in the Docker Compose file.

I also created a JMeter test project to test the performance of the REST APIs. Since both Spring Boot projects expose the same REST API, the single JMeter project can be used to test both. The JMeter thread group was configured with 10 threads and a loop count of 1000. This would result in each of the CRUD operations being run 10 x 1000 = 10 000 times.

For testing a cold JVM, the JMeter test is run immediately after the Docker container is started. For testing a warmed up JVM, the JMeter test is run like a cold JVM test, the results are then discarded and a second JMeter test is performed a few seconds later.

The test machine is a laptop with 24 CPU cores and 32Gb of RAM. JMeter will use 10 CPU cores for the 10 threads used to test and the Spring Boot application will use 2. This should leave sufficient CPU cores for PostgreSQL and other operating system threads.

## Expected results
Based of the claims I have heard and read about, I expected that:
- The reactive WebFlux stack will perform better than the Servlet stack.
- GraalVM native images will start up faster and use less memory than JVM images.
- JVM images will have a better throughput than GraalVM native images, but because GraalVM uses less memory resources and start up faster, GraalVM may still have better overall performance if you run multiple GraalVM containers.
- JVM images will perform faster when warmed up, but GraalVM will not gain performence when warmed up.
- Enabling Project Loom virtual threads will yield better performance, at least on the Servlet stack. It is unclear whether virtual threads would have any benefit on the WebFlux stack or when using GraalVM native images.

Thus when running a single Docker container, I expected to get the best performance when combining WebFlux, a warmed up JVM image and enabling virtual threads. I expected the worst results when combining Servlets, cold GraalVM native images and disabling virtual threads.

## Running the tests
If you would like to run the tests yourself. Please follow the instructions [here](INSTRUCTIONS.md).

## Actual results
The following table presents the actual results of the 12 test scenarios:

| JVM | Spring Boot Stack | Project Loom Enabled | Warm JVM | Image build time (s) | Image size (MB) | Startup Time (seconds) | Memory Usage (MB) | Create throughput | Read throughput | Update throughput | Delete throughput | Total throughput |
|---|---|---|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| OpenJDK | Servlet | No | No | 17 | 299 | 10.192 | 176.7829 | 208.8 | 209.6 | 209.7 | 210.1 | 835.1 |
| OpenJDK | Servlet | No | Yes | 17 | 299 | 12.291 | 196.5141 | 441.3 | 441.3 | 441.3 | 441.4 | 1764.6 |
| OpenJDK | WebFlux | No | No | 15 | 284 | 6.36 | 143.9233 | 166.7 | 167 | 167.1 | 167.1 | 666.6 |
| OpenJDK | WebFlux | No | Yes | 15 | 284 | 8.686 | 143.3799 | 362.4 | 362.5 | 362.5 | 362.5 | 1449.5 |
| GraalVM | Servlet | No | No | 150 | 214 | 0.739 | 90.3086 | 356.2 | 356.5 | 356.5 | 356.5 | 1424.7 |
| GraalVM | Servlet | No | Yes | 150 | 214 | 0.607 | 79.2985 | 308.8 | 308.8 | 308.8 | 308.9 | 1234.9 |
| GraalVM | WebFlux | No | No | 114 | 159 | 0.28 | 52.4287 | 365.8 | 365.9 | 365.9 | 366 | 1463.1 |
| GraalVM | WebFlux | No | Yes | 114 | 159 | 0.166 | 20.9715 | 298.8 | 298.8 | 298.8 | 298.9 | 1194.9 |
| OpenJDK | Servlet | Yes | No | 17 | 299 | 9.303 | 182.7296 | 186.7 | 188 | 188.6 | 189.3 | 746.7 |
| OpenJDK | WebFlux | Yes | No | 15 | 284 | 6.857 | 156.0851 | 174.2 | 174.6 | 174.8 | 174.8 | 696.9 |
| GraalVM | Servlet | Yes | No | 150 | 214 | 0.428 | 80.8714 | 352.5 | 352.6 | 352.6 | 352.6 | 1409.7 |
| GraalVM | WebFlux | Yes | No | 114 | 159 | 0.139 | 40.8944 | 358.7 | 358.8 | 358.8 | 358.9 | 1434.6 |

## Data analysis

### Servlet vs WebFlux
I was surprised to find that the Servlet stack was faster than WebFlux when running on JVM and only slightly slower when running on a cold GraalVM.

It could be that in the example WebFlux application has a code issue that prevented optimal performance, but even this would be a significant finding. It would mean that Webflux doesn't automatically yield better performance and that developers need to take care to get the best performance out of Webflux.

It could also be that the current JMeter test project with 10 threads is insufficient to show the advantages of WebFlux. JMeter is currently making blocking HTTP calls, so perhaps the REST client needs to make non blocking HTTP/2 calls to show the advantages of WebFlux. 

It is unlikely that this result was caused by the CPU and memory limits applied to the Docker containers. When running similar tests (data not shown) without these limits, the performance of Servlet over WebFlux only widened.

I have seen similar studies online where WebFlux did outperform the Servlet stack by a wide margin. The code of these sudies did not appear to be very different, but they did use older versions of Java and Spring Boot.

It is unclear if the performance of the Servlet stack improved with recent Java and Spring Boot versions. Alternatively the performance of the Servlet vs WebFlux stacks could be highly dependent on the problem domain.

> ***Opinion***  
> I am of the opinion that Servlet code is easier to read and
> write, but I realize that this is highly subjective.
> I would be willing to sacrifice some code readability and 
> developer productivity in exchange for runtime performance
> in cases where performance is the primary concern.
> But it is hard to recommend WebFlux code if the performance
> gains are not clear.
>
> I would suggest comparing the Servlet and WebFlux code of this
> repository to form your own opinion about code readibility and
> developer productivity.

### JVM vs GraalVM
It was again surprising that GraalVM yielded better throughput than a cold JVM, but a warmed up JVM does indeed seem to outperrform GraalVM. It is also interesting to see that GraalVM seems to have a bigger performance improvement than the difference between Servlet and Webflux.

I would say that the results were as expected.

### Cold vs Warmed Up JVM
It is interesting to see that on a standard JVM, warming up the JVM more than doubles the performance.

It was also interesting to see that on GraalVM, warming up the JVM actually deteririorates the performance. The reason for this is unclear.

It seems that JVM imagees would benefit from being up and running for a long time, but that GraalVM images do not benefit from running for extended periods.

I would say that the results were as expected.

### Virtual Threads
It was surprising that enabling virtual threads yielded slightly lower performance all four scenarios. I limited the CPU cores of the docker containers to 2 and know that by default Spring Boot uses thread pools of 10 connections for database connections etc. I would have expected some thread switching to occur and that virtual threads would improve the performance.

The fact the the performance was lower when enabling virtual threads does mean the Spring Boot config setting is doing something.

It could be that virtual threads do benefit performance in some scenarios and not others.

## Summary of analysis
Based on the data analysis above, it seems that the best performance was achieved by using the Servlet stack on a standard JVM that has been warmed up and virtual threads being disabled. The worst performance was achieved using the WebFlux stack on a cold standard JVM.

## Conclusion
Performance measurement is a complex subject and good advice would be to test the performance of your own code instead of relying on the findings of others.
