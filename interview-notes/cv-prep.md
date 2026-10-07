# CV prep: what to say for each line

Written on Week 8 Day 2 (Tuesday 2026-10-06), the morning of the job summit.
The CV itself is in `Downloads/MY CVs/Anton-Dahdal-CV.docx`.

One rule for the whole CV: only claim what you can explain one level deeper. If a recruiter asks "how did that work?", you need a real answer.

---

## 1. Fix these lines in the CV before you send it

- **Project line.** Today it only says REST, Spring Security and JPA, which undersells it. Add one bullet with the real work: a row lock so the last seat can't be sold twice, a 10-minute seat hold with an expiry job, an outbox with a poller (the ticket and the confirm are saved in one commit), Resilience4j retry and circuit breaker on the call to Event, a gateway, and Docker Compose.
- **"Designed as microservices: authentication, catalog, booking, and API gateway."** This claims more than the code shows (see section 8). Replace it with: "Modular Spring Boot backend with an API gateway. Modules talk over HTTP with Resilience4j retry, a circuit breaker and a transactional outbox, ready to split into Auth, Event and Booking services."
- **"Catalog".** In the code this is the Event module. Use "Event" so the CV matches GitHub.
- **Skills.** Add Docker. You used it.
- **Summary.** "Currently deepening Spring Boot for a mid-level role" sounds junior for someone with five years. Just say what you do.
- **Spring Batch.** Until you've seen an import starting with `org.springframework.batch` in the Amdocs batch code, write "Spring-based batch jobs". Section 4 explains why.
- **SQL tuning.** Write "Wrote Oracle SQL and worked with DBAs on performance tuning using AWR reports and execution plans." That's what you actually did.
- **Microservices** (under methodologies). Be ready to say honestly where you did this. Section 8 has the answer.

---

## 2. "Tell me about yourself" (about 60 seconds)

The shape: who you are, the domain, one win, the project, what you want.

> I'm a backend Java engineer at Amdocs, with five years on large telecom billing and payment systems. I work in Java, Spring and Oracle, on both new development and production defects. I worked on two projects: Telefónica Hispam, where we ran and extended the system, and a customer migration after another company acquired our product. In the migration I built the batch flow that handled locked customers without failing the whole run. On the side I built an event-booking backend in Spring Boot, with JWT security, a gateway and an outbox, to work with the modern Spring stack. I'm looking for a backend Java and Spring Boot role.

Say "Telefónica Hispam", not "Tef". Say "defects", not "DFS". Recruiters only hear words they already know.

---

## 3. The two Amdocs projects

1. **Telefónica Hispam**, the Latin America telecom operator. We ran the system in production, and I did new development and fixed defects.
2. **Customer migration.** We moved subscriber accounts from our billing system to the system of the company that acquired our product. I did development and fixed defects.

### The migration story (your best story)

> During the migration, each customer could be open, locked, or already migrated. I enhanced the batch job that read customers from a file. Customers who were already migrated were rejected with an error. Locked customers were not dropped: they were saved to a retry table. Then I built a second batch job that read from that table instead of the file, validated each customer again, and processed it once it was unlocked.

Why two jobs: the first one reads the file and the second reads the retry table, so one locked customer never fails the whole run.

This is the same idea as the outbox poller in the booking app: park the record, and let a separate job retry it later. If the interviewer likes the first story, bridge to the second.

---

## 4. The Amdocs batch framework in Spring Batch words

In the internal framework, a job class extends the framework's base job class and is started from a bash script. Its main methods are `init`, the mapper, `run` and `performFinalActivities`.

An `@Override` on `performFinalActivities` doesn't prove it's Spring Batch. It only means your class replaces a method from its parent class, and Spring Batch has no method with that name. The real check is the imports: if you see `org.springframework.batch`, it's Spring Batch. If not, it's Amdocs' own framework built on Spring.

Either way, an interviewer will use Spring Batch words, so know how they map:

| Internal framework | Spring Batch |
|---|---|
| The whole run | A **Job**, made of **Steps** |
| `init` reads the file or the DB | **ItemReader** (a file reader or a DB reader) |
| The mapper turns a line or row into a Java object | **LineMapper** or **RowMapper**, inside the reader |
| `run` calls the service and processes | **ItemProcessor** (validate and transform) plus **ItemWriter** (save the chunk) |
| Batches of 100 | The **chunk size**, also called the commit interval. One chunk is one transaction |
| A batch fails, everything rolls back, then each record is retried one by one and only the bad one fails | A **fault-tolerant step with skip**, plus a skip limit. Spring Batch does exactly this |
| Several threads | A **multi-threaded step** (TaskExecutor) or **partitioning** |
| `finalActivities` marks the file as completed or failed | A **JobExecutionListener** with `afterJob`. The status (COMPLETED or FAILED) is stored in the **JobRepository**, which is also how a failed job restarts from where it stopped |

### A typical job of this kind, in Spring Batch words (made-up example)

Picture a job that reads records and writes them to an output file. Its three methods map one to one:

- **`next()` is the ItemReader.** Each call reads one record and wraps it in a holder object. When there are no more records it returns `null` and sets a "has more" flag to false. Spring Batch's `read()` has the same rule: returning `null` means "end of input."
- **`run(list, context)` is where a chunk arrives.** The framework calls `next()` many times, groups the results, and passes the group to `run`. In Spring Batch that's the **ItemWriter**, which receives one chunk at a time. In this example, `run` doesn't write anything yet: it adds each record to a list field.
- **`performFinalActivities()` is the end-of-job hook.** When the input is finished, it writes **all** the collected records to the file at once. In Spring Batch that's `afterStep` / `afterJob`.

A made-up job in that style (invented names, for reading):

```java
public class InvoiceExportJob extends BaseBatchJob {

    private final InvoiceSource source;
    private final InvoiceFileWriter fileWriter;
    private final List<Invoice> collected = new ArrayList<>();
    private boolean hasMore = true;

    // ItemReader: one record per call, null = no more input
    @Override
    public Object next() {
        Invoice invoice = source.nextInvoice();
        if (invoice == null) {
            hasMore = false;
            return null;
        }
        return new InvoiceHolder(invoice);
    }

    // A chunk arrives here (like ItemWriter.write(chunk)); this version only collects
    @Override
    public void run(List<Object> chunk, JobContext ctx) {
        for (Object item : chunk) {
            collected.add(((InvoiceHolder) item).getInvoice());
        }
    }

    // End-of-job hook (like afterStep): write everything at once
    @Override
    public void performFinalActivities() {
        if (!hasMore && !collected.isEmpty()) {
            try {
                fileWriter.write(collected);
            } catch (IOException e) {
                log.error("export failed", e);   // trap 3 below: only logged
            }
        }
    }
}
```

Three things worth noticing (good to raise if asked "what would you improve?"):

1. **Everything is kept in memory until the end.** Every record from every chunk sits in the list until `performFinalActivities`. With millions of records that risks an OutOfMemoryError. Spring Batch would write each chunk to the file as it goes (`FlatFileItemWriter`), so memory stays flat.
2. **A crash means starting over.** Nothing is written until the very end, so a crash at 90% loses everything. Spring Batch writes per chunk and saves progress in the JobRepository, so a restart continues from the last committed chunk.
3. **The `IOException` is only logged.** If writing the file fails, the job still finishes as if it worked, with no file. The fix is to rethrow so the job ends as FAILED. It's the same trap as an empty catch that returns 201 in the booking app.

If the framework runs `run` on several threads, the shared list would also need to be thread-safe. Check how the job is configured before claiming either way.

**The safe sentence:**

> At Amdocs I built batch jobs on our internal batch framework, which is built on Spring. It uses the same chunk model as Spring Batch: read, process and write in chunks of 100, multi-threaded, with rollback and one-by-one reprocessing to isolate the bad record.

---

### Caching in the batch jobs

**A hand-written cache (made-up example of the pattern).** A framework without Spring Cache can still cache lookups by hand. A typical lookup method does four steps:

1. Build a **key** from the method name and its parameters, for example the lookup name plus the business entity plus the reason code.
2. Ask a shared cache for that key. The cache lives in a **singleton** (`getInstance()`), so the whole process shares one cache.
3. **Miss** (null): run the query with plain **JDBC** (a `PreparedStatement`), then put the result in the cache with a **TTL** (minutes to live).
4. **Hit**: return the cached object. No database call.

A made-up version (invented names, for reading):

```java
public TaxRateData getTaxRate(String region, String productCode) {
    String key = "TaxRateLookup.getTaxRate," + region + "," + productCode;   // 1. key

    CacheEntry cached = LookupCache.getInstance().get(key);                  // 2. shared singleton cache
    if (cached != null) {
        return (TaxRateData) cached.getValue();                              // 4. hit: no DB call
    }

    TaxRateData data = loadFromDb(region, productCode);                      // 3. miss: plain JDBC
    LookupCache.getInstance().put(key, data, TTL_MINUTES);                   //    store with a TTL
    return data;
}

private TaxRateData loadFromDb(String region, String productCode) {
    String sql = "SELECT rate, valid_from FROM tax_rates WHERE region = ? AND product_code = ?";
    try (PreparedStatement ps = connection.prepareStatement(sql)) {
        ps.setString(1, region);
        ps.setString(2, productCode);
        try (ResultSet rs = ps.executeQuery()) {
            return rs.next() ? new TaxRateData(rs.getBigDecimal("rate"), rs.getDate("valid_from")) : null;
        }
    }
}
```

The same lookup with Spring is one annotation: `@Cacheable("taxRates")` on `getTaxRate`, and Spring builds the key from the parameters.

This is the **cache-aside** pattern with a TTL, the same one from the Week 7 cache board. Spring's `@Cacheable` does exactly these four steps for you through a proxy.

> Reference data can be cached with cache-aside: the key is built from the lookup parameters, a miss reads Oracle through JDBC and stores the result with a TTL, and a hit skips the database. It's what Spring's `@Cacheable` automates.

**The Spring equivalent, in plain Spring (not Boot).** The difference from Boot: nothing is automatic. You turn caching on yourself and declare the `CacheManager` bean yourself.

The lookup service:

```java
public class PricePlanService {
    private final PricePlanDao dao;

    public PricePlanService(PricePlanDao dao) { this.dao = dao; }

    @Cacheable("pricePlans")              // first call per id hits Oracle, then memory
    public PricePlan find(long planId) {
        return dao.findById(planId);
    }
}
```

Turning it on with Java config:

```java
@Configuration
@EnableCaching                            // Boot does this for you; plain Spring does not
public class BatchConfig {

    @Bean
    public CacheManager cacheManager() {  // Boot auto-creates one; here you must declare it
        return new ConcurrentMapCacheManager("pricePlans");
    }

    @Bean
    public PricePlanService pricePlanService(PricePlanDao dao) {
        return new PricePlanService(dao);
    }
}
```

The same thing in XML, which is what older projects like Amdocs often used:

```xml
<cache:annotation-driven/>

<bean id="cacheManager"
      class="org.springframework.cache.concurrent.ConcurrentMapCacheManager">
    <constructor-arg value="pricePlans"/>
</bean>

<bean id="pricePlanService" class="com.example.PricePlanService">
    <constructor-arg ref="pricePlanDao"/>
</bean>
```

The batch processor (or `run` in the internal framework) gets `PricePlanService` injected and calls `pricePlanService.find(planId)` for each customer. A million customers on ten price plans means ten Oracle reads.

**The trap:** `@Cacheable` works through a proxy, like `@Transactional`. If a method inside `PricePlanService` calls `find()` on itself (`this.find(...)`), the call never goes through the proxy and the cache is skipped. The call has to come from another bean.

**Stale data:** `ConcurrentMapCacheManager` never expires anything. For data that changes, use a cache with a TTL (Ehcache or Caffeine) or evict with `@CacheEvict`.

---

## 5. Oracle SQL tuning

**What actually happened:** when there was a performance issue, we took the report, found the slow query from the batch job, and tuned it together with the DBA.

That report is the **AWR report**: Oracle's snapshot of the slowest and heaviest SQL over a period of time.

### What to look for in an execution plan

1. **A full table scan on a big table.** The plan says `TABLE ACCESS FULL`. If the query only needs a few rows out of millions, an index is missing or isn't being used.
2. **Cost and row counts.** Find the step with the highest cost. If the estimated number of rows is far from the real number, the table statistics are probably stale.
3. **An index that exists but is skipped.** This happens when the column is wrapped in a function (for example `UPPER(name)`) or compared to a value of the wrong type.

Adding the index is the fix. The interviewer wants to hear what in the plan told you an index was needed.

### Example: adding an index

A batch job reads the customers of one billing cycle from a huge table:

```sql
SELECT * FROM customers WHERE billing_cycle = :cycle AND status = 'OPEN';
```

On production data the plan shows `TABLE ACCESS FULL` on `customers`, which has millions of rows. The fix is an index on the columns in the `WHERE`:

```sql
CREATE INDEX idx_customers_cycle_status ON customers (billing_cycle, status);
```

Now the plan shows `INDEX RANGE SCAN`, and Oracle reads only the matching rows.

The cost: every insert and update also has to update the index, so you don't index every column.

### Example: the PARALLEL hint

A night batch job has to scan a big table anyway, for example all the charges of one month:

```sql
SELECT /*+ PARALLEL(c, 4) */ c.customer_id, SUM(c.amount)
FROM charges c
WHERE c.charge_month = :month
GROUP BY c.customer_id;
```

The hint tells Oracle to split the scan across 4 parallel processes, so one big query finishes faster.

The cost: it uses much more CPU and disk, and it can slow down everyone else on the database. Use it for night batch jobs, not for queries that run all day.

### Say it

> When we had a performance issue, I pulled the AWR report to find the heaviest queries from the batch jobs and went through the execution plan with our DBA. Usually it showed a full table scan on a big production table, so we added an index on the filtered columns. For big night batch scans we sometimes added a PARALLEL hint so Oracle uses several processes.

A bridge to the booking app: a query that runs once per row instead of once per batch is the same N+1 problem you fixed there with `@EntityGraph`.

---

## 6. SOAP vs REST

Your first answer was "SOAP returns XML and is used in WebLogic; REST returns JSON." That's half right.

- **WebLogic is only the application server** your SOAP services ran on. SOAP isn't tied to WebLogic, and REST services can run on WebLogic too.
- **SOAP is a protocol.** It always sends XML inside an envelope. Its contract is a **WSDL** file, and clients are generated from it. Usually there's one POST endpoint, and the operation name is inside the XML. It comes with standards like **WS-Security**.
- **REST is a style** on top of plain HTTP: resources in the URL, the HTTP methods (GET, POST, PUT, DELETE), and status codes (201, 404, 409). It usually uses JSON, but it can use XML. It's lighter and easy for browsers and mobile apps.

### Don't compare WSDL with REST

They belong to different levels. Compare in pairs:

| | SOAP | REST (your booking app) |
|---|---|---|
| How the two sides talk | SOAP: a protocol with XML envelopes | REST: a style over HTTP, usually JSON |
| The contract file | **WSDL** (required) | **OpenAPI / Swagger** (optional) |
| Who is calling, and keeping it safe | **WS-Security**: a token, a signature and encryption inside the message | A **JWT** in the `Authorization` header, plus HTTPS |

**How to remember it:** SOAP is a formal letter. REST is a text message. The WSDL is the menu: what you can order and exactly what comes back. WS-Security is the ID card and the sealed envelope.

**WSDL is not a token, and it doesn't encrypt anything.** It's only the menu, written once and shared before any call. The token travels inside every SOAP message, in its header. (You mixed these two up twice on Day 2.)

**SOAP runs on HTTPS too.** HTTPS protects the connection, like an armored truck: the message is safe while it travels, but once it arrives or passes a gateway it's plain again. WS-Security protects the message itself, like a sealed and signed envelope inside the truck: it stays protected even after it's stored or forwarded, and the signature proves who sent it. Banks often use both.

**When you'd still choose SOAP:** integrations between companies that need a strict contract and message-level security, such as telecom billing, banks and payment systems. That's why Amdocs uses it.

### Say it

> SOAP is a strict protocol that always sends XML. Its WSDL file is the contract that defines every operation and field, and WS-Security puts the token, signature and encryption inside the message. REST is a lighter style over plain HTTP, usually JSON, with Swagger as an optional contract and a JWT over HTTPS for security. That's why SOAP fits banks and billing systems, and REST fits web and mobile APIs.

The one-line version: **SOAP is strict XML with a WSDL contract and WS-Security. REST is light JSON over HTTP with a JWT.**

### A real example: Book as a SOAP service (for reading, not writing)

**The WSDL** is the menu, shared once. This is trimmed to the important parts:

```xml
<definitions name="BookingService" targetNamespace="http://booking.example.com/">
  <!-- types: the fields -->
  <types>
    <xsd:element name="bookSeatsRequest">
      <xsd:complexType><xsd:sequence>
        <xsd:element name="eventId" type="xsd:long"/>
        <xsd:element name="seats"   type="xsd:int"/>
      </xsd:sequence></xsd:complexType>
    </xsd:element>
    <xsd:element name="bookSeatsResponse">
      <xsd:complexType><xsd:sequence>
        <xsd:element name="bookingId" type="xsd:long"/>
        <xsd:element name="status"    type="xsd:string"/>
      </xsd:sequence></xsd:complexType>
    </xsd:element>
  </types>

  <!-- portType: the operations (like a Java interface) -->
  <portType name="BookingPort">
    <operation name="bookSeats">
      <input  message="tns:bookSeatsRequest"/>
      <output message="tns:bookSeatsResponse"/>
      <fault  name="SoldOutFault" message="tns:SoldOutFault"/>
    </operation>
  </portType>

  <!-- service: where it lives -->
  <service name="BookingService">
    <port name="BookingPort" binding="tns:BookingBinding">
      <soap:address location="https://booking.example.com/BookingService"/>
    </port>
  </service>
</definitions>
```

Read it in three parts: **types** are the fields, **portType** is the methods, and **service** is the URL.

**The SOAP request** is sent on every call. The token is in the header and the data is in the body:

```xml
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Header>
    <wsse:Security>                       <!-- WS-Security: who is calling -->
      <wsse:UsernameToken>
        <wsse:Username>partner-bank</wsse:Username>
        <wsse:Password>...</wsse:Password>
      </wsse:UsernameToken>
    </wsse:Security>
  </soap:Header>
  <soap:Body>                             <!-- the data -->
    <bookSeatsRequest>
      <eventId>5</eventId>
      <seats>2</seats>
    </bookSeatsRequest>
  </soap:Body>
</soap:Envelope>
```

**The SOAP response:**

```xml
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <bookSeatsResponse>
      <bookingId>123</bookingId>
      <status>CONFIRMED</status>
    </bookSeatsResponse>
  </soap:Body>
</soap:Envelope>
```

### The same example in REST with a JWT (your booking app)

**The OpenAPI / Swagger file** is the menu. It plays the same role as the WSDL, but in REST it's optional:

```yaml
paths:
  /api/bookings:
    post:
      summary: Book seats
      security:
        - bearerAuth: []          # needs a JWT
      requestBody:
        content:
          application/json:
            schema:
              properties:
                eventId: { type: integer }
                seats:   { type: integer }
      responses:
        "201": { description: Booked. Returns bookingId + status }
        "409": { description: Sold out }
        "401": { description: Missing or bad JWT }
```

Read it like this: the path plus the verb is the method, `requestBody` is the input fields, and `responses` are the outputs and errors. REST uses status codes where SOAP uses faults.

**The HTTP request** is sent on every call:

```http
POST /api/bookings HTTP/1.1
Host: booking.example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhbnRvbiIsInJvbGUiOiJVU0VSIiwiZXhwIjoxNzYwMDAwMDAwfQ.k3Jx...
Content-Type: application/json

{ "eventId": 5, "seats": 2 }
```

The token is in the HTTP `Authorization` header, and the data is the JSON body.

**The HTTP response:**

```http
HTTP/1.1 201 Created
Content-Type: application/json

{ "bookingId": 123, "status": "CONFIRMED" }
```

If the event is sold out, the answer is `409 Conflict`. If the token is missing or expired, the answer is `401`, sent by the security filter chain before the request ever reaches the controller.

### The JWT itself

A JWT is three Base64 parts joined by dots: `header.payload.signature`.

| Part | Decoded | What it means |
|---|---|---|
| Header | `{"alg": "HS256"}` | How the token is signed |
| Payload (the claims) | `{"sub": "anton", "role": "USER", "exp": 1760000000}` | Who the user is, their role, and when the token expires |
| Signature | An HMAC of the header and payload, made with the secret | Proves Auth issued the token and nobody changed it |

Auth issues the token at login. On every request, `JwtAuthenticationFilter` checks the signature and the expiry using the same secret (the `JWT_SECRET` environment variable). There's no database call per request, which is why it's called stateless.

The payload is encoded, not encrypted, so anyone can read it. Never put a password in it. HTTPS keeps the token from being stolen on the way.

### Side by side

| | SOAP | REST with JWT |
|---|---|---|
| The menu | WSDL (XML), required | OpenAPI (YAML or JSON), optional |
| The call | An XML envelope, POSTed to one URL, with the operation inside the body | A URL plus an HTTP verb, with a JSON body |
| Who is calling | A WS-Security token in the **SOAP header** | A JWT in the **HTTP `Authorization` header** |
| Errors | A SOAP fault | Status codes (401, 404, 409) |
| Security | HTTPS, plus optional signing and encryption of the message | HTTPS, plus a signed JWT |

---

## 7. EJB

### What an EJB is

An **EJB (Enterprise JavaBean)** is a Java EE business component that runs inside an application server. At Amdocs that server was WebLogic. The server's EJB container creates the bean and gives it transactions, security, pooling and remote calls.

There are three kinds:

- **Stateless session bean** (`@Stateless`): business logic with no memory of the caller between calls. The server keeps a pool of them. This is the common one.
- **Stateful session bean** (`@Stateful`): keeps state for one client, like the old shopping-cart example. Rare.
- **Message-driven bean** (`@MessageDriven`): listens to a JMS queue and processes each message.

Other systems find a remote EJB by name through **JNDI**, the server's lookup directory, and call it over RMI.

### EJB vs Spring bean

They do the same jobs with different tools:

| EJB (Java EE) | Spring |
|---|---|
| A `@Stateless` session bean | A `@Service` (singleton) |
| A container-managed transaction (`@TransactionAttribute`) | `@Transactional` (through a proxy) |
| A `@MessageDriven` bean on a JMS queue | `@JmsListener` or `@KafkaListener` |
| A JNDI lookup, or `@EJB` | Constructor injection |
| Needs a full application server (WebLogic) | Runs anywhere: embedded Tomcat, `java -jar`, Docker |

The difference in one line: both are managed objects with transactions, but an EJB needs a heavy application server, while a Spring bean is a plain Java class that runs anywhere and is easy to unit-test. Spring was created as the lighter alternative to old EJB.

> An EJB is a Java EE component managed by the application server, which at Amdocs was WebLogic. Mostly these are stateless session beans for business logic, with container-managed transactions. A Spring bean does the same job, a managed object with `@Transactional`, but it's a plain class that doesn't need an application server. That's why new services are built with Spring Boot.

### What you did with EJB at Amdocs

You built APIs in your application that the **CRM** system calls, and you gave the CRM team a jar that Amdocs calls the **client kit**.

That's a classic **remote EJB with a client jar**:

- Your business logic is an EJB running in your application on WebLogic.
- CRM **calls** your EJB and **sends a request object** as the parameter. The object is serialized, sent over the network, and rebuilt on your side. Your bean returns another object the same way.
- The **client kit** holds the remote interface (your method signatures) and the data classes those methods take and return, and usually the JNDI name. It contains no business logic.
- CRM needs the client kit because their Java code has to compile against your interface and classes before it can call you.

So the client kit is **the contract**. It does the same job for EJB that the WSDL does for SOAP and Swagger does for REST. Every integration needs a contract; only the packaging changes.

Two things to avoid saying:

- Not "CRM sends us an EJB." The EJB is your service. What CRM sends is the data object.
- Not "web API." An EJB call has no HTTP and no URL. Say "remote EJB API."

> I developed remote EJB APIs in our application that the CRM system calls, and gave them the client kit, with the interface and data classes, so they stay in sync with our contract.

### Why EJB inside Amdocs and SOAP outside (your own answer)

- **Inside Amdocs** (CRM calling your billing application): both sides are Amdocs Java applications on WebLogic, so remote EJB fits well. It sends Java objects directly, it's typed, there's no XML to parse, and it's fast.
- **Outside** (banks and telecom operators): you don't control their language or their stack, so you use SOAP, with a WSDL contract anyone can read and WS-Security for sensitive data.

> EJB for internal Java-to-Java calls between Amdocs products, SOAP for external partners like banks.

Today, the same split is usually REST or gRPC inside a company and REST or SOAP with outside partners.

---

## 8. Is the booking app split into microservices?

**Not fully.** Checked in the code on Day 2. Docker Compose runs two apps:

1. **One Spring Boot app** (`EventBookingPlatformApplication`, port 8080) that contains the Auth, Event and Booking code and one shared H2 database.
2. **A separate gateway app** (port 8081) that routes requests to it.

What makes it more than a plain monolith is that Booking calls Event and Auth **over HTTP**, through `EventClient` and `AuthClient` with `WebClient`, with retry, a circuit breaker, timeouts and the outbox, as if they were separate services. The base URL is `http://localhost:8080`, so today it calls itself. The boundaries between the services are real, but the deployment is still one app.

The usual name for this is a **modular monolith with service boundaries**, ready to split.

> It's one Spring Boot app plus a separate gateway. But I built Booking to call Event and Auth over HTTP through clients, with retry, a circuit breaker and an outbox, as if they were separate services. Splitting it means changing a base URL and giving each service its own database, not rewriting the booking logic.

### What's still missing to make it real microservices

1. **Booking's tables are linked to Event's and Auth's.** `Booking` has `@ManyToOne Event` and `@ManyToOne User`, which means a foreign key from the bookings table into the events and users tables. Separate services can't share tables, so these must become plain `eventId` and `userId` columns. The data already comes over HTTP through `EventClient` and `AuthClient`. This is the biggest blocker.
2. **One shared database.** Today it's one in-memory H2 database. Each service needs its own, for example one Postgres database each.
3. **One deployable.** It needs to become three Spring Boot apps (Auth, Event and Booking), each its own container in Docker Compose. `event.service.base-url` would then point to the Event container instead of `localhost:8080`.
4. **The gateway only routes Book** (`POST /api/events/{eventId}/bookings`). It also needs routes for auth, events, venues and my-bookings.
5. **JWT after the split.** Today the one app both issues and checks the token. After the split, Auth issues it, and Event and Booking only check it, using the same secret or Auth's public key with RS256.

**Update W8 Day 3 (2026-10-07):** items 1 and 4 are done. `Booking` now stores `eventId`, `userId` and a copy of `eventTitle`, with no foreign keys into Event or User. The gateway routes Auth, Event, venues, Book and "My tickets", and keeps Event's internal `seat-reservations` and `holds/{holdId}/confirm` off. Items 2, 3 and 5 are left.

**Already done, and it's the hard part:** HTTP clients between the modules, retry, a circuit breaker and timeouts, the outbox and poller, the seat hold with expiry, a correlation id, health probes, Docker Compose, and the gateway.

> The service boundaries are done: Booking reaches Event and Auth only over HTTP, with resilience and an outbox. What's left is replacing the JPA links between Booking and Event/User with plain ids, one database per service, and deploying three apps behind the gateway.

This turns "it's not finished" into "I know exactly how to finish it."

---

## 9. Still to prepare

- **"Collaborated with DevOps on high-availability systems":** one concrete example, such as a deployment, an outage or a failover, and what your part was.
- **Root cause analysis:** one real production defect from start to end. What broke, how you found the cause (logs, SQL, reproducing it), and what you fixed.
- **Two questions to ask the recruiter.** For example: "What does the backend stack look like: Spring Boot, microservices, cloud?" and "What would I work on in the first three months?"
