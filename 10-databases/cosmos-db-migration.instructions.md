---
description: 'Migration to Azure Cosmos DB via Spring Data: from Cassandra and JPA'
applyTo: '**/*.java, **/pom.xml, **/build.gradle, **/application*.properties, **/application*.yml, **/application*.conf'
---


# Comprehensive Guide: Converting Spring Boot Cassandra Applications to use Azure Cosmos DB with Spring Data Cosmos (spring-data-cosmos)

## Applicability

This guide applies to:
- ✅ Spring Boot 2.x - 3.x applications (both reactive and non-reactive)
- ✅ Maven and Gradle-based projects
- ✅ Applications using Spring Data Cassandra, Cassandra DAOs, or DataStax drivers
- ✅ Projects with or without Lombok
- ✅ UUID-based or String-based entity identifiers
- ✅ Both synchronous and reactive (Spring WebFlux) applications

This guide does NOT cover:
- ❌ Non-Spring frameworks (Jakarta EE, Micronaut, Quarkus, plain Java)
- ❌ Complex Cassandra features (materialized views, UDTs, counters, custom types)
- ❌ Bulk data migration (code conversion only - data must be migrated separately)
- ❌ Cassandra-specific features like lightweight transactions (LWT) or batch operations across partitions

## Overview

This guide provides step-by-step instructions for converting reactive Spring Boot applications from Apache Cassandra to Azure Cosmos DB using Spring Data Cosmos. It covers all major issues encountered and their solutions, based on real-world conversion experience.

## Prerequisites

- Java 11 or higher (Java 17+ required for Spring Boot 3.x)
- Azure CLI installed and authenticated (`az login`) for local development
- Azure Cosmos DB account created in Azure Portal
- Maven 3.6+ or Gradle 6+ (depending on your project)
- For Gradle projects with Spring Boot 3.x: Ensure JAVA_HOME environment variable points to Java 17+
- Basic understanding of your application's data model and query patterns

## Database Setup for Azure Cosmos DB

**CRITICAL**: Before running your application, ensure the database exists in your Cosmos DB account.

### Option 1: Manual Database Creation (Recommended for first run)
1. Go to Azure Portal → Your Cosmos DB account
2. Navigate to "Data Explorer"
3. Click "New Database"
4. Enter the database name matching your application configuration (check `application.properties` or `application.yml` for the configured database name)
5. Choose throughput settings (Manual or Autoscale based on your needs)
   - Start with Manual 400 RU/s for development/testing
   - Use Autoscale for production workloads with variable traffic
6. Click OK

### Option 2: Automatic Creation
Spring Data Cosmos can auto-create the database on first connection, but this requires:
- Proper RBAC permissions (Cosmos DB Built-in Data Contributor role)
- May fail if permissions are insufficient

### Container (Collection) Creation
Containers will be auto-created by Spring Data Cosmos when the application starts, using the `@Container` annotation settings from your entities. No manual container creation is needed unless you want to configure specific throughput or indexing policies.

## Authentication with Azure Cosmos DB

### Using DefaultAzureCredential (Recommended)
The `DefaultAzureCredential` authentication method is the recommended approach for both development and production:

**How it works**:
1. Tries multiple credential sources in order:
   - Environment variables
   - Workload Identity (for AKS)
   - Managed Identity (for Azure VMs/App Service)
   - Azure CLI (`az login`)
   - Azure PowerShell
   - Azure Developer CLI

**Setup for local development**:
```bash
# Login via Azure CLI
az login

# The application will automatically use your CLI credentials
```

**Configuration** (no key required):
```java
@Bean
public CosmosClientBuilder getCosmosClientBuilder() {
    return new CosmosClientBuilder()
        .endpoint(uri)
        .credential(new DefaultAzureCredentialBuilder().build());
}
```

**Properties file** (application-cosmos.properties or application.properties):
```properties
azure.cosmos.uri=https://<your-cosmos-account-name>.documents.azure.com:443/
azure.cosmos.database=<your-database-name>
# No key property needed when using DefaultAzureCredential
azure.cosmos.populate-query-metrics=false
```

**Note**: Replace `<your-cosmos-account-name>` and `<your-database-name>` with your actual values.

### RBAC Permissions Required
When using DefaultAzureCredential, your Azure identity needs appropriate RBAC permissions:

**Common startup error**:
```
Request blocked by Auth: Request for Read DatabaseAccount is blocked because principal
[xxx] does not have required RBAC permissions to perform action
[Microsoft.DocumentDB/databaseAccounts/sqlDatabases/write] on any scope.
```

**Solution**: Assign the "Cosmos DB Built-in Data Contributor" role:
```bash
# Get your user's object ID
PRINCIPAL_ID=$(az ad signed-in-user show --query id -o tsv)

# Assign the role (replace <resource-group> with your actual resource group)
az cosmosdb sql role assignment create \
  --account-name your-cosmos-account \
  --resource-group <resource-group> \
  --scope "/" \
  --principal-id $PRINCIPAL_ID \
  --role-definition-name "Cosmos DB Built-in Data Contributor"
```

**Alternative**: If you're logged in with `az login`, your account should already have permissions if you're the owner/contributor of the Cosmos DB account.

### Key-Based Authentication (Local Emulator Only)
Only use key-based authentication for local emulator development:

```java
@Bean
public CosmosClientBuilder getCosmosClientBuilder() {
    // Only for local emulator
    if (key != null && !key.isEmpty()) {
        return new CosmosClientBuilder()
            .endpoint(uri)
            .key(key);
    }
    // Production: use DefaultAzureCredential
    return new CosmosClientBuilder()
        .endpoint(uri)
        .credential(new DefaultAzureCredentialBuilder().build());
}
```

## Critical Lessons Learned

### Java Version Requirements (Spring Boot 3.x)
**Problem**: Spring Boot 3.0+ requires Java 17 or higher. Using Java 11 causes build failures.
**Error**:
```
No matching variant of org.springframework.boot:spring-boot-gradle-plugin:3.0.5 was found.
Incompatible because this component declares a component compatible with Java 17
and the consumer needed a component compatible with Java 11
```

**Solution**:
```bash
# Check Java version
java -version

# Set JAVA_HOME to Java 17+
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64  # Linux
# or
export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk-17.jdk/Contents/Home  # macOS

# Verify
echo $JAVA_HOME
```

**For Gradle projects**, always run with correct JAVA_HOME:
```bash
export JAVA_HOME=/path/to/java-17
./gradlew clean build
./gradlew bootRun
```

### Gradle-Specific Issues

#### Issue 1: Old Configuration File Conflicts
**Problem**: When renaming or replacing Cassandra configuration files, the old file may still exist, causing compilation errors:
```
error: class CosmosConfiguration is public, should be declared in a file named CosmosConfiguration.java
```

**Solution**: Explicitly delete the old Cassandra configuration file(s):
```bash
# Find and remove old Cassandra config files
find src/main/java -name "*CassandraConfig*.java" -o -name "*CassandraConfiguration*.java"
# Review the output, then delete if appropriate
rm src/main/java/<path-to-old-config>/CassandraConfig.java
```

#### Issue 2: Repository findAllById Returns Iterable
**Problem**: CosmosRepository's `findAllById()` returns `Iterable<Entity>`, not `List<Entity>`. Calling `.stream()` directly fails:
```
error: cannot find symbol
  symbol:   method stream()
  location: interface Iterable<YourEntity>
```

**Solution**: Handle Iterable properly:
```java
// WRONG - Iterable doesn't have stream() method
var entities = repository.findAllById(ids).stream()...

// CORRECT - Option 1: Use forEach to populate a collection
Iterable<YourEntity> entitiesIterable = repository.findAllById(ids);
Map<String, YourEntity> entityMap = new HashMap<>();
entitiesIterable.forEach(entity -> entityMap.put(entity.getId(), entity));

// CORRECT - Option 2: Convert to List first
List<YourEntity> entities = new ArrayList<>();
repository.findAllById(ids).forEach(entities::add);

// CORRECT - Option 3: Use StreamSupport (Java 8+)
List<YourEntity> entities = StreamSupport.stream(
    repository.findAllById(ids).spliterator(), false)
    .collect(Collectors.toList());
```

### package-info.java javax.annotation Issues
**Problem**: `package-info.java` using `javax.annotation.ParametersAreNonnullByDefault` causes compilation errors in Java 11+:
```
error: cannot find symbol
import javax.annotation.ParametersAreNonnullByDefault;
```

**Solution**: Remove or simplify the package-info.java file:
```java
// Simple version - just package declaration
package com.your.package;
```

### Entity Constructor Issues
**Problem**: Using Lombok `@NoArgsConstructor` with manual constructors causes duplicate constructor compilation errors.
**Solution**: Choose one approach:
- Option 1: Remove `@NoArgsConstructor` and keep manual constructors
- Option 2: Remove manual constructors and rely on Lombok annotations
- **Best Practice**: For Cosmos entities with initialization logic (like setting partition keys), remove `@NoArgsConstructor` and use manual constructors only.

### Business Object Constructor Removal
**Problem**: Removing `@AllArgsConstructor` or custom constructors from entity classes breaks existing code that uses those constructors.
**Impact**: Mapping utilities, data seeders, and test files will fail to compile.
**Solution**:
- After removing or modifying constructors, search ALL files for constructor calls to those entities
- Replace with default constructor + setter pattern:
  ```java
  // Before - using all-args constructor
  MyEntity entity = new MyEntity(id, field1, field2, field3);

  // After - using default constructor + setters
  MyEntity entity = new MyEntity();
  entity.setId(id);
  entity.setField1(field1);
  entity.setField2(field2);
  entity.setField3(field3);
  ```
### Data Seeder Constructor Calls
**Problem**: Data seeding or initialization code uses entity constructors that may not exist after entity conversion to Cosmos annotations.
**Solution**: Update all entity instantiations in data seeding components to use setters:
```java
// Before - constructor-based initialization
MyEntity entity1 = new MyEntity("entity-1", "value1", "value2");

// After - setter-based initialization
MyEntity entity1 = new MyEntity();
entity1.setId("entity-1");
entity1.setField1("value1");
entity1.setField2("value2");
```

**Common files to check**: DataSeeder, DatabaseInitializer, TestDataLoader, or any `@Component` implementing `CommandLineRunner`
```java
OwnerEntity owner1 = new OwnerEntity();
owner1.setId("owner-1");
```

### Test File Updates Required
**Problem**: Test files reference old Cassandra DAOs and use UUID constructors.
**Critical Files to Update**:
1. Remove `MockReactiveResultSet.java` (Cassandra-specific)
2. Update `*ReactiveServicesTest.java` - replace DAO references with Cosmos repositories
3. Update `*ReactiveControllerTest.java` - replace DAO references with Cosmos repositories
4. Replace all `UUID.fromString()` with String IDs
5. Replace constructor calls: `new Owner(UUID.fromString(...))` with setter pattern

### Application Startup and DefaultAzureCredential Behavior
**Important**: DefaultAzureCredential tries multiple authentication methods in sequence, which is normal and expected.

**Expected startup log pattern**:
```
INFO c.azure.identity.ChainedTokenCredential : Azure Identity => Attempted credential EnvironmentCredential is unavailable.
INFO c.azure.identity.ChainedTokenCredential : Azure Identity => Attempted credential WorkloadIdentityCredential is unavailable.
INFO c.azure.identity.ChainedTokenCredential : Azure Identity => Attempted credential ManagedIdentityCredential is unavailable.
INFO c.azure.identity.ChainedTokenCredential : Azure Identity => Attempted credential SharedTokenCacheCredential is unavailable.
INFO c.azure.identity.ChainedTokenCredential : Azure Identity => Attempted credential IntelliJCredential is unavailable.
INFO c.azure.identity.ChainedTokenCredential : Azure Identity => Attempted credential AzureCliCredential is unavailable.
INFO c.azure.identity.ChainedTokenCredential : Azure Identity => Attempted credential AzurePowerShellCredential is unavailable.
INFO c.azure.identity.ChainedTokenCredential : Azure Identity => Attempted credential AzureDeveloperCliCredential returns a token
```

**Key points**:
- The "unavailable" messages are **NORMAL** - it's trying each credential source in order
- Once it finds one that works (e.g., AzureCliCredential or AzureDeveloperCliCredential), it will use that
- **Do not interrupt the startup process** - it takes 10-15 seconds to cycle through credential sources
- Application typically takes 30-60 seconds total to fully start and connect to Cosmos DB

**Success indicators**:
```
INFO c.a.c.i.RxDocumentClientImpl : Initializing DocumentClient [1] with serviceEndpoint [https://your-account.documents.azure.com:443/]
INFO c.a.c.i.GlobalEndpointManager : db account retrieved {...}
INFO c.a.c.implementation.SessionContainer : Registering a new collection resourceId [...]
INFO o.s.b.w.embedded.tomcat.TomcatWebServer : Tomcat started on port(s): 8944 (http)
INFO com.your.app.Application : Started Application in X.XXX seconds
```

**Troubleshooting startup failures**:

1. **If all credentials are "unavailable"**:
   ```bash
   # Re-authenticate with Azure CLI
   az login

   # Verify login
   az account show
   ```

2. **If you see permission errors**:
   ```
   Request blocked by Auth: principal [xxx] does not have required RBAC permissions
   ```
   - Ensure database exists in Cosmos DB account (see Database Setup section)
   - Verify RBAC permissions (see Authentication section)
   - Check that you're logged into the correct Azure subscription

3. **Port already in use**:
   ```bash
   # Find and kill the process
   lsof -ti:8944 | xargs kill -9

   # Or change the port in application.properties
   server.port=8945
   ```

### Application Startup Patience
**Problem**: Application takes 30-60 seconds to fully start (compilation + Spring Boot + Cosmos DB connection).
**Solution**:
- For Gradle: `./gradlew bootRun` (runs in foreground by default)
- For Maven: `mvn spring-boot:run`
- Use background execution if needed: `nohup ./gradlew bootRun > app.log 2>&1 &`
- **CRITICAL**: Do not interrupt the startup process, especially during credential authentication (10-15 seconds)
- Monitor logs: `tail -f app.log` or check for "Started Application" message
- Wait for Tomcat to start and show the port number before testing endpoints

### Port Configuration
**Problem**: Application may not run on default port 8080.
**Solution**:
- Check actual port: `ss -tlnp | grep java`
- Test connectivity: `curl http://localhost:<port>/petclinic/api/owners`
- Common ports: 8080, 9966, 9967

## Systematic Compilation Error Resolution

During this conversion, we encountered over 100 compilation errors. Here's the systematic approach that resolved them:

### Step 1: Identify Residual Cassandra Files
**Problem**: Old Cassandra-specific files cause compilation errors after dependencies are removed.
**Solution**: Delete all Cassandra-specific files systematically:

```bash
# Identify and delete old DAOs
find . -name "*Dao.java" -o -name "*DAO.java"
# Delete: OwnerReactiveDao, PetReactiveDao, VetReactiveDao, VisitReactiveDao

# Identify and delete Cassandra mappers
find . -name "*Mapper.java" -o -name "*EntityToOwnerMapper.java"
# Delete: EntityToOwnerMapper, EntityToPetMapper, EntityToVetMapper, EntityToVisitMapper

# Identify and delete old configuration
find . -name "*CassandraConfig.java" -o -name "CassandraConfiguration.java"
# Delete: CassandraConfiguration.java

# Identify test utilities for Cassandra
find . -name "MockReactiveResultSet.java"
# Delete: MockReactiveResultSet.java (Cassandra-specific test utility)
```

### Step 2: Run Incremental Compilation Checks
**Approach**: After each major change, compile to identify remaining issues:

```bash
# After deleting old files
mvn compile 2>&1 | grep -E "(ERROR|error)" | wc -l
# Expected: Number decreases with each fix

# After updating entity constructors
mvn compile 2>&1 | grep "constructor"
# Identify constructor-related compilation errors

# After fixing business object constructors
mvn compile 2>&1 | grep -E "(new Owner|new Pet|new Vet|new Visit)"
# Identify remaining constructor calls that need fixing
```

### Step 3: Fix Constructor-Related Errors Systematically
**Pattern**: Search for all constructor calls in specific file types:

```bash
# Find all constructor calls in MappingUtils
grep -n "new Owner\|new Pet\|new Vet\|new Visit" src/main/java/**/MappingUtils.java

# Find all constructor calls in DataSeeder
grep -n "new OwnerEntity\|new PetEntity\|new VetEntity\|new VisitEntity" src/main/java/**/DataSeeder.java

# Find all constructor calls in test files
grep -rn "new Owner\|new Pet\|new Vet\|new Visit" src/test/java/
```

### Step 4: Update Tests Last
**Rationale**: Fix application code before test code to see all issues clearly:

1. First: Update test repository mocks (DAO → Cosmos Repository)
2. Second: Fix UUID → String conversions in test data
3. Third: Update constructor calls in test setup
4. Finally: Run tests to verify: `mvn test`

### Step 5: Verify Zero Compilation Errors
**Final Check**:
```bash
# Clean and full compile
mvn clean compile

# Should see: BUILD SUCCESS
# Should NOT see any ERROR messages

# Verify test compilation
mvn test-compile

# Run tests
mvn test
```

**Success Indicators**:
- `mvn compile`: BUILD SUCCESS
- `mvn test`: All tests pass (even if some are skipped)
- No ERROR messages in output
- No "cannot find symbol" errors
- No "constructor cannot be applied" errors

## Conversion Steps

### 1. Update Maven Dependencies

#### Remove Cassandra Dependencies
```xml
<!-- REMOVE these Cassandra dependencies -->
<dependency>
    <groupId>com.datastax.oss</groupId>
    <artifactId>java-driver-core</artifactId>
</dependency>
<dependency>
    <groupId>com.datastax.oss</groupId>
    <artifactId>java-driver-query-builder</artifactId>
</dependency>
```

#### Add Azure Cosmos Dependencies
```xml
<!-- Azure Spring Data Cosmos (Java 11 compatible) -->
<dependency>
    <groupId>com.azure</groupId>
    <artifactId>azure-spring-data-cosmos</artifactId>
    <version>3.46.0</version>
</dependency>

<!-- Azure Identity for DefaultAzureCredential authentication -->
<dependency>
    <groupId>com.azure</groupId>
    <artifactId>azure-identity</artifactId>
    <version>1.11.4</version>
</dependency>
```

#### Critical: Add Version Management for Compatibility
Spring Boot 2.3.x has version conflicts with Azure libraries. Add this to your `<dependencyManagement>` section:

```xml
<dependencyManagement>
    <dependencies>
        <!-- Override reactor-netty version to fix compatibility with azure-spring-data-cosmos -->
        <dependency>
            <groupId>io.projectreactor.netty</groupId>
            <artifactId>reactor-netty</artifactId>
            <version>1.0.40</version>
        </dependency>
        <dependency>
            <groupId>io.projectreactor.netty</groupId>
            <artifactId>reactor-netty-http</artifactId>
            <version>1.0.40</version>
        </dependency>
        <dependency>
            <groupId>io.projectreactor.netty</groupId>
            <artifactId>reactor-netty-core</artifactId>
            <version>1.0.40</version>
        </dependency>

        <!-- Override reactor-core version to support Sinks API required by azure-identity -->
        <dependency>
            <groupId>io.projectreactor</groupId>
            <artifactId>reactor-core</artifactId>
            <version>3.4.32</version>
        </dependency>

        <!-- Override Netty versions to fix compatibility with Azure Cosmos Client -->
        <dependency>
            <groupId>io.netty</groupId>
            <artifactId>netty-bom</artifactId>
            <version>4.1.101.Final</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>

        <!-- Override netty-tcnative to match Netty version -->
        <dependency>
            <groupId>io.netty</groupId>
            <artifactId>netty-tcnative-boringssl-static</artifactId>
            <version>2.0.62.Final</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### 2. Configuration Setup

#### Create Cosmos Configuration Class
Replace your Cassandra configuration with:

```java
@Configuration
@EnableCosmosRepositories  // Required for non-reactive repositories
@EnableReactiveCosmosRepositories  // CRITICAL: Required for reactive repositories
public class CosmosConfiguration extends AbstractCosmosConfiguration {

    @Value("${azure.cosmos.uri}")
    private String uri;

    @Value("${azure.cosmos.database}")
    private String database;

    @Bean
    public CosmosClientBuilder getCosmosClientBuilder() {
        return new CosmosClientBuilder()
            .endpoint(uri)
            .credential(new DefaultAzureCredential());
    }

    @Bean
    public CosmosAsyncClient cosmosAsyncClient(CosmosClientBuilder cosmosClientBuilder) {
        return cosmosClientBuilder.buildAsyncClient();
    }

    @Bean
    public CosmosClientBuilderFactory cosmosFactory(CosmosAsyncClient cosmosAsyncClient) {
        return new CosmosClientBuilderFactory(cosmosAsyncClient);
    }

    @Bean
    public ReactiveCosmosTemplate reactiveCosmosTemplate(CosmosClientBuilderFactory cosmosClientBuilderFactory) {
        return new ReactiveCosmosTemplate(cosmosClientBuilderFactory, database);
    }

    @Override
    protected String getDatabaseName() {
        return database;
    }
}
```

**Critical Notes:**
- **BOTH annotations required**: @EnableCosmosRepositories AND @EnableReactiveCosmosRepositories
- Missing @EnableReactiveCosmosRepositories will cause "No qualifying bean" errors for reactive repositories

#### Application Properties
Add cosmos profile configuration:

```properties
# application-cosmos.properties
azure.cosmos.uri=https://your-cosmos-account.documents.azure.com:443/
azure.cosmos.database=your-database-name
```

### 3. Entity Conversion

#### Convert from Cassandra to Cosmos Annotations

**Before (Cassandra):**
```java
@Table(value = "entity_table")
public class EntityName {
    @PartitionKey
    private UUID id;

    @ClusteringColumn
    private String fieldName;

    @Column("column_name")
    private String anotherField;
}
```

**After (Cosmos):**
```java
@Container(containerName = "entities")
public class EntityName {
    @Id
    private String id;  // Changed from UUID to String

    @PartitionKey
    private String fieldName;  // Choose appropriate partition key

    private String anotherField;

    // Generate String IDs
    public EntityName() {
        this.id = UUID.randomUUID().toString();
    }
}
```

#### Key Changes:
- Replace `@Table` with `@Container(containerName = "...")`
- Change `@PartitionKey` to Cosmos partition key strategy
- Convert all IDs from `UUID` to `String`
- Remove `@Column` annotations (Cosmos uses field names)
- Remove `@ClusteringColumn` (not applicable in Cosmos)

### 4. Repository Conversion

#### Replace Cassandra Data Access Layer with Cosmos Repositories

**If your application uses DAOs or custom data access classes:**

**Before (Cassandra DAO pattern):**
```java
@Repository
public class EntityReactiveDao {
    // Custom Cassandra query methods
}
```

**After (Cosmos Repository):**
```java
@Repository
public interface EntityCosmosRepository extends ReactiveCosmosRepository<EntityName, String> {

    @Query("SELECT * FROM entities e WHERE e.fieldName = @fieldName")
    Flux<EntityName> findByFieldName(@Param("fieldName") String fieldName);

    @Query("SELECT * FROM entities e WHERE e.id = @id")
    Mono<EntityName> findEntityById(@Param("id") String id);
}
```

**If your application uses Spring Data Cassandra repositories:**

**Before:**
```java
@Repository
public interface EntityCassandraRepository extends ReactiveCassandraRepository<EntityName, UUID> {
    // Cassandra-specific methods
}
```

**After:**
```java
@Repository
public interface EntityCosmosRepository extends ReactiveCosmosRepository<EntityName, String> {
    // Convert existing methods to Cosmos queries
}
```

**If your application uses direct CqlSession or Cassandra driver:**
- Replace direct driver calls with repository pattern
- Convert CQL queries to Cosmos SQL syntax
- Implement repository interfaces as shown above

#### Key Points:
- **CRITICAL**: Use `ReactiveCosmosRepository<Entity, String>` for reactive programming (NOT CosmosRepository)
- Use `CosmosRepository<Entity, String>` for non-reactive applications
- **Repository Interface Change**: If converting from existing Cassandra repositories/DAOs, ensure all repository interfaces extend ReactiveCosmosRepository
- **Common Error**: "No qualifying bean of type ReactiveCosmosRepository" = missing @EnableReactiveCosmosRepositories
- **If using custom data access classes**: Convert to repository pattern for better integration
- **If already using Spring Data**: Change interface extension from ReactiveCassandraRepository to ReactiveCosmosRepository
- Implement custom queries with `@Query` annotation using SQL-like syntax (not CQL)
- All query parameters must use `@Param` annotation

### 5. Service Layer Updates

#### Update Service Classes for Reactive Programming (If Applicable)

**If your application has a service layer:**

**CRITICAL**: Service methods must return Flux/Mono, not Iterable/Optional

```java
@Service
public class EntityReactiveServices {
    private final EntityCosmosRepository repository;

    public EntityReactiveServices(EntityCosmosRepository repository) {
        this.repository = repository;
    }

    // CORRECT: Returns Flux<EntityName>
    public Flux<EntityName> findAll() {
        return repository.findAll();
    }

    // CORRECT: Returns Mono<EntityName>
    public Mono<EntityName> findById(String id) {
        return repository.findById(id);
    }

    // CORRECT: Returns Mono<EntityName>
    public Mono<EntityName> save(EntityName entity) {
        return repository.save(entity);
    }

    // Custom queries - MUST return Flux/Mono
    public Flux<EntityName> findByFieldName(String fieldName) {
        return repository.findByFieldName(fieldName);
    }

    // WRONG PATTERNS TO AVOID:
    // public Iterable<EntityName> findAll() - Will cause compilation errors
    // public Optional<EntityName> findById() - Will cause compilation errors
    // repository.findAll().collectList() - Unnecessary blocking
}
```

**If your application uses direct repository injection in controllers:**
- Consider adding a service layer for better separation of concerns
- Update controller dependencies to use new Cosmos repositories
- Ensure proper reactive type handling throughout the call chain

**Common Issues:**
- **Compilation Error**: "Cannot resolve method" when using Iterable return types
- **Runtime Error**: Attempting to call .collectList() or .block() unnecessarily
- **Performance**: Blocking reactive streams defeats the purpose of reactive programming

### 6. Controller Updates (If Applicable)

#### Update REST Controllers for String IDs

**If your application has REST controllers:**

**Before:**
```java
@GetMapping("/entities/{entityId}")
public Mono<EntityDto> getEntity(@PathVariable UUID entityId) {
    return entityService.findById(entityId);
}
```

**After:**
```java
@GetMapping("/entities/{entityId}")
public Mono<EntityDto> getEntity(@PathVariable String entityId) {
    return entityService.findById(entityId);
}
```

**If your application doesn't use controllers:**
- Apply the same UUID → String conversion principles to your data access layer
- Update any external APIs or interfaces that accept/return entity IDs

### 7. Data Mapping Utilities (If Applicable)

#### Update Mapping Between Domain Objects and Entities

**If your application uses mapping utilities or converters:**

```java
public class MappingUtils {

    // Convert domain object to entity
    public static EntityName toEntity(DomainObject domain) {
        EntityName entity = new EntityName();
        entity.setId(domain.getId()); // Now String instead of UUID
        entity.setFieldName(domain.getFieldName());
        entity.setAnotherField(domain.getAnotherField());
        // ... other fields
        return entity;
    }

    // Convert entity to domain object
    public static DomainObject toDomain(EntityName entity) {
        DomainObject domain = new DomainObject();
        domain.setId(entity.getId());
        domain.setFieldName(entity.getFieldName());
        domain.setAnotherField(entity.getAnotherField());
        // ... other fields
        return domain;
    }
}
```

**If your application doesn't use explicit mapping:**
- Ensure consistent ID type usage throughout your codebase
- Update any object construction or copying logic to handle String IDs

### 8. Test Updates

#### Update Test Classes

**Critical**: All test files must be updated to work with String IDs and Cosmos repositories:

```java
**If your application has unit tests:**

```java
@ExtendWith(MockitoExtension.class)
class EntityReactiveServicesTest {

    @Mock
    private EntityCosmosRepository entityRepository; // Updated to Cosmos repository

    @InjectMocks
    private EntityReactiveServices entityService;

    @Test
    void testFindById() {
        String entityId = "test-entity-id"; // Changed from UUID to String
        EntityName mockEntity = new EntityName();
        mockEntity.setId(entityId);

        when(entityRepository.findById(entityId)).thenReturn(Mono.just(mockEntity));

        StepVerifier.create(entityService.findById(entityId))
            .expectNext(mockEntity)
            .verifyComplete();
    }
}
```

**If your application has integration tests:**
- Update test data setup to use String IDs
- Replace Cassandra test containers with Cosmos DB emulator (if available)
- Update test queries to use Cosmos SQL syntax instead of CQL

**If your application doesn't have tests:**
- Consider adding basic tests to verify the conversion works correctly
- Focus on testing ID conversion and basic CRUD operations
```

### 9. Common Issues and Solutions

#### Issue 1: NoClassDefFoundError with reactor.core.publisher.Sinks
**Problem**: Azure Identity library requires newer Reactor Core version
**Error**: `java.lang.NoClassDefFoundError: reactor/core/publisher/Sinks`
**Root Cause**: Spring Boot 2.3.x uses older reactor-core that doesn't have Sinks API
**Solution**: Add reactor-core version override in dependencyManagement (see Step 1)

#### Issue 2: NoSuchMethodError with Netty Epoll methods
**Problem**: Version mismatch between Spring Boot Netty and Azure Cosmos requirements
**Error**: `java.lang.NoSuchMethodError: 'boolean io.netty.channel.epoll.Epoll.isTcpFastOpenClientSideAvailable()'`
**Root Cause**: Spring Boot 2.3.x uses Netty 4.1.51.Final, Azure requires newer methods
**Solution**: Add netty-bom version override (see Step 1)

#### Issue 3: NoSuchMethodError with SSL Context
**Problem**: Netty TLS native library version mismatch
**Error**: `java.lang.NoSuchMethodError: 'boolean io.netty.internal.tcnative.SSLContext.setCurvesList(long, java.lang.String[])'`
**Root Cause**: netty-tcnative version incompatible with upgraded Netty
**Solution**: Add netty-tcnative-boringssl-static version override (see Step 1)

#### Issue 4: ReactiveCosmosRepository beans not created
**Problem**: Missing @EnableReactiveCosmosRepositories annotation
**Error**: `No qualifying bean of type 'ReactiveCosmosRepository' available`
**Root Cause**: Only @EnableCosmosRepositories doesn't create reactive repository beans
**Solution**: Add both @EnableCosmosRepositories and @EnableReactiveCosmosRepositories to configuration

#### Issue 5: Repository interface compilation errors
**Problem**: Using CosmosRepository instead of ReactiveCosmosRepository
**Error**: `Cannot resolve method 'findAll()' in 'CosmosRepository'`
**Root Cause**: CosmosRepository returns Iterable, not Flux
**Solution**: Change all repository interfaces to extend ReactiveCosmosRepository<Entity, String>

#### Issue 6: Service layer reactive type mismatches
**Problem**: Service methods returning Iterable/Optional instead of Flux/Mono
**Error**: `Required type: Flux<Entity> Provided: Iterable<Entity>`
**Root Cause**: Repository methods return reactive types, services must match
**Solution**: Update all service method signatures to return Flux/Mono

#### Issue 7: Authentication failures with DefaultAzureCredential
**Problem**: DefaultAzureCredential not finding credentials
**Error**: `All credentials in the chain are unavailable` or specific credential unavailable messages
**Root Cause**: No valid Azure credential source available

**Solutions**:
1. **For local development**: Ensure Azure CLI login
   ```bash
   az login
   # Verify login
   az account show
   ```

2. **For Azure-hosted applications**: Ensure Managed Identity is enabled and has proper RBAC permissions

3. **Check credential chain order**: DefaultAzureCredential tries in this order:
   - Environment variables → Workload Identity → Managed Identity → Azure CLI → PowerShell → Developer CLI

#### Issue 8: Database not found errors
**Problem**: Application fails to start with database not found errors
**Error**: `Database 'your-database-name' not found` or `Resource Not Found`
**Root Cause**: Database doesn't exist in Cosmos DB account

**Solution**: Create the database before first run (see Database Setup section):
```bash
# Via Azure CLI
az cosmosdb sql database create \
  --account-name your-cosmos-account \
  --name your-database-name \
  --resource-group your-resource-group

# Or via Azure Portal (recommended for first-time setup)
# Portal → Cosmos DB → Data Explorer → New Database
```

**Note**: Containers (collections) will be auto-created from entity `@Container` annotations, but the database itself may need to exist first depending on your RBAC permissions.

#### Issue 9: RBAC permission errors
**Problem**: Application fails with permission denied errors
**Error**:
```
Request blocked by Auth: principal [xxx] does not have required RBAC permissions
to perform action [Microsoft.DocumentDB/databaseAccounts/sqlDatabases/write]
```

**Root Cause**: Your Azure identity lacks required Cosmos DB permissions

**Solution**: Assign "Cosmos DB Built-in Data Contributor" role:
```bash
# Get resource group
RESOURCE_GROUP=$(az cosmosdb show --name your-cosmos-account --query resourceGroup -o tsv 2>/dev/null)

# If the above fails, list all Cosmos accounts to find it
az cosmosdb list --query "[?name=='your-cosmos-account'].{name:name, resourceGroup:resourceGroup}" -o table

# Assign role
az cosmosdb sql role assignment create \
  --account-name your-cosmos-account \
  --resource-group $RESOURCE_GROUP \
  --scope "/" \
  --principal-id $(az ad signed-in-user show --query id -o tsv) \
  --role-definition-name "Cosmos DB Built-in Data Contributor"
```

**Alternative**: Portal → Cosmos DB → Access Control (IAM) → Add role assignment → "Cosmos DB Built-in Data Contributor"

#### Issue 10: Partition key strategy differences
**Problem**: Cassandra clustering keys don't map directly to Cosmos partition keys
**Error**: Cross-partition queries or poor performance
**Root Cause**: Different data distribution strategies
**Solution**: Choose appropriate partition key based on query patterns, typically the most frequently queried field

#### Issue 10: UUID to String conversion issues
**Problem**: Test files and controllers still using UUID types
**Error**: `Cannot convert UUID to String` or type mismatch errors
**Root Cause**: Not all occurrences of UUID were converted to String
**Solution**: Systematically search and replace all UUID references with String

### 10. Data Seeding (If Applicable)

#### Implement Data Population

**If your application needs initial data:**

```java
@Component
public class DataSeeder implements CommandLineRunner {

    private final EntityCosmosRepository entityRepository;

    @Override
    public void run(String... args) throws Exception {
        if (entityRepository.count().block() == 0) {
            // Seed initial data
            EntityName entity = new EntityName();
            entity.setFieldName("Sample Value");
            entity.setAnotherField("Sample Data");

            entityRepository.save(entity).block();
        }
    }
}
```

**If your application has existing data migration needs:**
- Create migration scripts to export from Cassandra and import to Cosmos DB
- Consider data transformation needs (UUID to String conversion)
- Plan for any schema differences between Cassandra and Cosmos data models

**If your application doesn't need data seeding:**
- Skip this step and proceed to verification

### 11. Application Profiles

#### Update application.yml for Cosmos profile
```yaml
spring:
  profiles:
    active: cosmos

---
spring:
  profiles: cosmos

azure:
  cosmos:
    uri: ${COSMOS_URI:https://your-account.documents.azure.com:443/}
    database: ${COSMOS_DATABASE:your-database}
```

## Verification Steps

1. **Compile Check**: `mvn compile` should succeed without errors
2. **Test Check**: `mvn test` should pass with updated test cases
3. **Runtime Check**: Application should start without version conflicts
4. **Connection Check**: Application should connect to Cosmos DB successfully
5. **Data Check**: CRUD operations should work through the API
6. **UI Check**: Frontend should display data from Cosmos DB

## Best Practices

1. **ID Strategy**: Always use String IDs instead of UUIDs for Cosmos DB
2. **Partition Key**: Choose partition keys based on query patterns and data distribution
3. **Query Design**: Use @Query annotation for custom queries instead of method naming conventions
4. **Reactive Programming**: Stick to Flux/Mono patterns throughout the service layer
5. **Version Management**: Always include dependency version overrides for Spring Boot 2.x projects
6. **Testing**: Update all test files to use String IDs and mock Cosmos repositories
7. **Authentication**: Use DefaultAzureCredential for production-ready authentication

## Troubleshooting Commands

```bash
# Check dependencies and version conflicts
mvn dependency:tree | grep -E "(reactor|netty|cosmos)"

# Verify specific problematic dependencies
mvn dependency:tree | grep "reactor-core"
mvn dependency:tree | grep "reactor-netty"
mvn dependency:tree | grep "netty-tcnative"

# Test connection
curl http://localhost:8080/api/entities

# Check Azure login status
az account show

# Clean and rebuild (often fixes dependency issues)
mvn clean compile

# Run with debug logging for dependency resolution
mvn dependency:resolve -X

# Check for compilation errors specifically
mvn compile 2>&1 | grep -E "(ERROR|error)"

# Run with debug for runtime issues
mvn spring-boot:run -Dspring-boot.run.jvmArguments="-Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=5005"

# Check application logs for version conflicts
grep -E "(NoSuchMethodError|NoClassDefFoundError|reactor|netty)" application.log
```

## Typical Error Sequence and Resolution

Based on real conversion experience, you'll likely encounter these errors in this order:

### **Phase 1: Compilation Errors**
1. **Missing dependencies** → Add azure-spring-data-cosmos and azure-identity
2. **Configuration class errors** → Create CosmosConfiguration (if not already present)
3. **Entity annotation errors** → Convert @Table to @Container, etc.
4. **Repository interface errors** → Change to ReactiveCosmosRepository (if using repository pattern)

### **Phase 2: Bean Creation Errors**
5. **"No qualifying bean of type ReactiveCosmosRepository"** → Add @EnableReactiveCosmosRepositories
6. **Service layer type mismatches** → Change Iterable to Flux, Optional to Mono (if using service layer)

### **Phase 3: Runtime Version Conflicts** (Most Complex)
7. **NoClassDefFoundError: reactor.core.publisher.Sinks** → Add reactor-core 3.4.32 override
8. **NoSuchMethodError: Epoll.isTcpFastOpenClientSideAvailable** → Add netty-bom 4.1.101.Final override
9. **NoSuchMethodError: SSLContext.setCurvesList** → Add netty-tcnative-boringssl-static 2.0.62.Final override

### **Phase 4: Authentication & Connection**
10. **ManagedIdentityCredential authentication unavailable** → Run `az login --use-device-code`
11. **Application starts successfully** → Connected to Cosmos DB!

**Critical**: Address these in order. Don't skip ahead - each phase must be resolved before the next appears.

## Performance Considerations

1. **Partition Strategy**: Design partition keys to distribute load evenly
2. **Query Optimization**: Use indexes and avoid cross-partition queries when possible
3. **Connection Pooling**: Cosmos client automatically manages connections
4. **Request Units**: Monitor RU consumption and adjust throughput as needed
5. **Bulk Operations**: Use batch operations for multiple document updates

This guide covers all major aspects of converting from Cassandra to Cosmos DB, including all version conflicts and authentication issues encountered in real-world scenarios.

---


# Convert Spring JPA project to Spring Data Cosmos

This generalized guide applies to any JPA to Spring Data Cosmos DB conversion project.

## High-level plan

1. Swap build dependencies (remove JPA, add Cosmos + Identity).
2. Add `cosmos` profile and properties.
3. Add Cosmos config with proper Azure identity authentication.
4. Transform entities (ids → `String`, add `@Container` and `@PartitionKey`, remove JPA mappings, adjust relationships).
5. Convert repositories (`JpaRepository` → `CosmosRepository`).
6. **Create service layer** for relationship management and template compatibility.
7. **CRITICAL**: Update ALL test files to work with String IDs and Cosmos repositories.
8. Seed data via `CommandLineRunner`.
9. **CRITICAL**: Test runtime functionality and fix template compatibility issues.

## Step-by-step

### Step 1 — Build dependencies

- **Maven** (`pom.xml`):
  - Remove dependency `spring-boot-starter-data-jpa`
  - Remove database-specific dependencies (H2, MySQL, PostgreSQL) unless needed elsewhere
  - Add `com.azure:azure-spring-data-cosmos:5.17.0` (or latest compatible version)
  - Add `com.azure:azure-identity:1.15.4` (required for DefaultAzureCredential)
- **Gradle**: Apply same dependency changes for Gradle syntax
- Remove testcontainers and JPA-specific test dependencies

### Step 2 — Properties and Configuration

- Create `src/main/resources/application-cosmos.properties`:
  ```properties
  azure.cosmos.uri=${COSMOS_URI:https://localhost:8081}
  azure.cosmos.database=${COSMOS_DATABASE:petclinic}
  azure.cosmos.populate-query-metrics=false
  azure.cosmos.enable-multiple-write-locations=false
  ```
- Update `src/main/resources/application.properties`:
  ```properties
  spring.profiles.active=cosmos
  ```

### Step 3 — Configuration class with Azure Identity

- Create `src/main/java/<rootpkg>/config/CosmosConfiguration.java`:
  ```java
  @Configuration
  @EnableCosmosRepositories(basePackages = "<rootpkg>")
  public class CosmosConfiguration extends AbstractCosmosConfiguration {

    @Value("${azure.cosmos.uri}")
    private String uri;

    @Value("${azure.cosmos.database}")
    private String dbName;

    @Bean
    public CosmosClientBuilder getCosmosClientBuilder() {
      return new CosmosClientBuilder().endpoint(uri).credential(new DefaultAzureCredentialBuilder().build());
    }

    @Override
    protected String getDatabaseName() {
      return dbName;
    }

    @Bean
    public CosmosConfig cosmosConfig() {
      return CosmosConfig.builder().enableQueryMetrics(false).build();
    }
  }

  ```
- **IMPORTANT**: Use `DefaultAzureCredentialBuilder().build()` instead of key-based authentication for production security

### Step 4 — Entity transformation

- Target all classes with JPA annotations (`@Entity`, `@MappedSuperclass`, `@Embeddable`)
- **Base entity changes**:
  - Change `id` field type from `Integer` to `String`
  - Add `@Id` and `@GeneratedValue` annotations
  - Add `@PartitionKey` field (typically `String partitionKey`)
  - Remove all `jakarta.persistence` imports
- **CRITICAL - Cosmos DB Serialization Requirements**:
  - **Remove ALL `@JsonIgnore` annotations** from fields that need to be persisted to Cosmos DB
  - **Authentication entities (User, Authority) MUST be fully serializable** - no `@JsonIgnore` on password, authorities, or other persisted fields
  - **Use `@JsonProperty` instead of `@JsonIgnore`** when you need to control JSON field names but still persist the data
  - **Common authentication serialization errors**: `Cannot pass null or empty values to constructor` usually means `@JsonIgnore` is blocking required field serialization
- **Entity-specific changes**:
  - Replace `@Entity` with `@Container(containerName = "<plural-entity-name>")`
  - Remove `@Table`, `@Column`, `@JoinColumn`, etc.
  - Remove relationship annotations (`@OneToMany`, `@ManyToOne`, `@ManyToMany`)
  - For relationships:
    - Embed collections for one-to-many (e.g., `List<Pet> pets` in Owner)
    - Use reference IDs for many-to-one (e.g., `String ownerId` in Pet)
    - **For complex relationships**: Store IDs but add transient properties for templates
  - Add constructor to set partition key: `setPartitionKey("entityType")`
- **CRITICAL - Authentication Entity Pattern**:
  - **For User entities with Spring Security**: Store authorities as `Set<String>` instead of `Set<Authority>` objects
  - **Example User entity transformation**:
    ```java
    @Container(containerName = "users")
    public class User {

      @Id
      private String id;

      @PartitionKey
      private String partitionKey = "user";

      private String login;
      private String password; // NO @JsonIgnore - must be serializable

      @JsonProperty("authorities") // Use @JsonProperty, not @JsonIgnore
      private Set<String> authorities = new HashSet<>(); // Store as strings

      // Add transient property for Spring Security compatibility if needed
      // @JsonIgnore - ONLY for transient properties not persisted to Cosmos
      private Set<Authority> authorityObjects = new HashSet<>();

      // Conversion methods between string authorities and Authority objects
      public void setAuthorityObjects(Set<Authority> authorities) {
        this.authorityObjects = authorities;
        this.authorities = authorities.stream().map(Authority::getName).collect(Collectors.toSet());
      }
    }

    ```
- **CRITICAL - Template Compatibility for Relationship Changes**:
  - **When converting relationships to ID references, preserve template access**
  - **Example**: If entity had `List<Specialty> specialties` → convert to:
    - Storage: `List<String> specialtyIds` (persisted to Cosmos)
    - Template: `@JsonIgnore private List<Specialty> specialties = new ArrayList<>()` (transient)
    - Add getters/setters for both properties
  - **Update entity method logic**: `getNrOfSpecialties()` should use the transient list
- **CRITICAL - Template Compatibility for Thymeleaf/JSP Applications**:
  - **Identify template property access**: Search for `${entity.relationshipProperty}` in `.html` files
  - **For each relationship property accessed in templates**:
    - **Storage**: Keep ID-based storage (e.g., `List<String> specialtyIds`)
    - **Template Access**: Add transient property with `@JsonIgnore` (e.g., `private List<Specialty> specialties = new ArrayList<>()`)
    - **Example**:

      ```java
      // Stored in Cosmos (persisted)
      private List<String> specialtyIds = new ArrayList<>();

      // For template access (transient)
      @JsonIgnore
      private List<Specialty> specialties = new ArrayList<>();

      // Getters/setters for both properties
      public List<String> getSpecialtyIds() {
        return specialtyIds;
      }

      public List<Specialty> getSpecialties() {
        return specialties;
      }

      ```

    - **Update count methods**: `getNrOfSpecialties()` should use transient list, not ID list
- **CRITICAL - Method Signature Conflicts**:
  - **When converting ID types from Integer to String, check for method signature conflicts**
  - **Common conflict**: `getPet(String name)` vs `getPet(String id)` - both have same signature
  - **Solution**: Rename methods to be specific:
    - `getPet(String id)` for ID-based lookup
    - `getPetByName(String name)` for name-based lookup
    - `getPetByName(String name, boolean ignoreNew)` for conditional name-based lookup
  - **Update ALL callers** of renamed methods in controllers and tests
- **Method updates for entities**:
  - Update `addVisit(Integer petId, Visit visit)` to `addVisit(String petId, Visit visit)`
  - Ensure all ID comparison logic uses `.equals()` instead of `==`

### Step 5 — Repository conversion

- Change all repository interfaces:
  - From: `extends JpaRepository<Entity, Integer>`
  - To: `extends CosmosRepository<Entity, String>`
- **Query method updates**:
  - Remove pagination parameters from custom queries
  - Change `Page<Entity> findByX(String param, Pageable pageable)` to `List<Entity> findByX(String param)`
  - Update `@Query` annotations to use Cosmos SQL syntax
  - **Replace custom method names**: `findPetTypes()` → `findAllOrderByName()`
  - **Update ALL references** to changed method names in controllers and formatters

### Step 6 — **Create service layer** for relationship management and template compatibility

- **CRITICAL**: Create service classes to bridge Cosmos document storage with existing template expectations
- **Purpose**: Handle relationship population and maintain template compatibility
- **Service pattern for each entity with relationships**:
  ```java
  @Service
  public class EntityService {

    private final EntityRepository entityRepository;
    private final RelatedRepository relatedRepository;

    public EntityService(EntityRepository entityRepository, RelatedRepository relatedRepository) {
      this.entityRepository = entityRepository;
      this.relatedRepository = relatedRepository;
    }

    public List<Entity> findAll() {
      List<Entity> entities = entityRepository.findAll();
      entities.forEach(this::populateRelationships);
      return entities;
    }

    public Optional<Entity> findById(String id) {
      Optional<Entity> entityOpt = entityRepository.findById(id);
      if (entityOpt.isPresent()) {
        Entity entity = entityOpt.get();
        populateRelationships(entity);
        return Optional.of(entity);
      }
      return Optional.empty();
    }

    private void populateRelationships(Entity entity) {
      if (entity.getRelatedIds() != null && !entity.getRelatedIds().isEmpty()) {
        List<Related> related = entity
          .getRelatedIds()
          .stream()
          .map(relatedRepository::findById)
          .filter(Optional::isPresent)
          .map(Optional::get)
          .collect(Collectors.toList());
        // Set transient property for template access
        entity.setRelated(related);
      }
    }
  }

  ```

### Step 6.5 — **Spring Security Integration** (CRITICAL for Authentication)

- **UserDetailsService Integration Pattern**:
  ```java
  @Service
  @Transactional
  public class DomainUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;
    private final AuthorityRepository authorityRepository;

    @Override
    public UserDetails loadUserByUsername(String login) {
      log.debug("Authenticating user: {}", login);

      return userRepository
        .findOneByLogin(login)
        .map(user -> createSpringSecurityUser(login, user))
        .orElseThrow(() -> new UsernameNotFoundException("User " + login + " was not found"));
    }

    private org.springframework.security.core.userdetails.User createSpringSecurityUser(String lowercaseLogin, User user) {
      if (!user.isActivated()) {
        throw new UserNotActivatedException("User " + lowercaseLogin + " was not activated");
      }

      // Convert string authorities back to GrantedAuthority objects
      List<GrantedAuthority> grantedAuthorities = user
        .getAuthorities()
        .stream()
        .map(SimpleGrantedAuthority::new)
        .collect(Collectors.toList());

      return new org.springframework.security.core.userdetails.User(user.getLogin(), user.getPassword(), grantedAuthorities);
    }
  }

  ```
- **Key Authentication Requirements**:
  - User entity must be fully serializable (no `@JsonIgnore` on password/authorities)
  - Store authorities as `Set<String>` for Cosmos DB compatibility
  - Convert between string authorities and `GrantedAuthority` objects in UserDetailsService
  - Add comprehensive debugging logs to trace authentication flow
  - Handle activated/deactivated user states appropriately

#### **Template Relationship Population Pattern**

Each service method that returns entities for template rendering MUST populate transient properties:

```java
private void populateRelationships(Entity entity) {
  // For each relationship used in templates
  if (entity.getRelatedIds() != null && !entity.getRelatedIds().isEmpty()) {
    List<Related> relatedObjects = entity
      .getRelatedIds()
      .stream()
      .map(relatedRepository::findById)
      .filter(Optional::isPresent)
      .map(Optional::get)
      .collect(Collectors.toList());
    entity.setRelated(relatedObjects); // Set transient property
  }
}

```

#### **Critical Service Usage in Controllers**

- **Replace ALL direct repository calls** with service calls in controllers
- **Never return entities from repositories directly** to templates without relationship population
- **Update controllers** to use service layer instead of repositories directly
- **Controller pattern change**:

  ```java
  // OLD: Direct repository usage
  @Autowired
  private EntityRepository entityRepository;

  // NEW: Service layer usage
  @Autowired
  private EntityService entityService;
  // Update method calls
  // OLD: entityRepository.findAll()
  // NEW: entityService.findAll()

  ```

### Step 7 — Data seeding

- Create `@Component` implementing `CommandLineRunner`:
  ```java
  @Component
  public class DataSeeder implements CommandLineRunner {

    @Override
    public void run(String... args) throws Exception {
      if (ownerRepository.count() > 0) {
        return; // Data already exists
      }
      // Seed comprehensive test data with String IDs
      // Use meaningful ID patterns: "owner-1", "pet-1", "pettype-1", etc.
    }
  }

  ```
- **CRITICAL - BigDecimal Reflection Issues with JDK 17+**:
  - **If using BigDecimal fields**, you may encounter reflection errors during seeding
  - **Error pattern**: `Unable to make field private final java.math.BigInteger java.math.BigDecimal.intVal accessible`
  - **Solutions**:
    1. Use `Double` or `String` instead of `BigDecimal` for monetary values
    2. Add JVM argument: `--add-opens java.base/java.math=ALL-UNNAMED`
    3. Wrap BigDecimal operations in try-catch and handle gracefully
  - **The application will start successfully even if seeding fails** - check logs for seeding errors

### Step 8 — Test file conversion (CRITICAL SECTION)

**This step is often overlooked but essential for successful conversion**

#### A. **COMPILATION CHECK STRATEGY**

- **After each major change, run `mvn test-compile` to catch issues early**
- **Fix compilation errors systematically before proceeding**
- **Don't rely on IDE - Maven compilation reveals all issues**

#### B. **Search and Update ALL test files systematically**

**Use search tools to find and update every occurrence:**

- Search for: `int.*TEST.*ID` → Replace with: `String.*TEST.*ID = "test-xyz-1"`
- Search for: `setId\(\d+\)` → Replace with: `setId("test-id-X")`
- Search for: `findById\(\d+\)` → Replace with: `findById("test-id-X")`
- Search for: `\.findPetTypes\(\)` → Replace with: `.findAllOrderByName()`
- Search for: `\.findByLastNameStartingWith\(.*,.*Pageable` → Remove pagination parameter

#### C. Update test annotations and imports

- Replace `@DataJpaTest` with `@SpringBootTest` or appropriate slice test
- Remove `@AutoConfigureTestDatabase` annotations
- Remove `@Transactional` from tests (unless single-partition operations)
- Remove imports from `org.springframework.orm` package

#### D. Fix entity ID usage in ALL test files

**Critical files that MUST be updated (search entire test directory):**

- `*ControllerTests.java` - Path variables, entity creation, mock setup
- `*ServiceTests.java` - Repository interactions, entity IDs
- `EntityUtils.java` - Utility methods for ID handling
- `*FormatterTests.java` - Repository method calls
- `*ValidatorTests.java` - Entity creation with String IDs
- Integration test classes - Test data setup

#### E. **Fix Controller and Service classes affected by repository changes**

- **Update controllers that call repository methods with changed signatures**
- **Update formatters/converters that use repository methods**
- **Common files to check**:
  - `PetTypeFormatter.java` - often calls `findPetTypes()` method
  - `*Controller.java` - may have pagination logic to remove
  - Service classes that use repository methods

#### F. Update repository mocking in tests

- Remove pagination from repository mocks:
  - `given(repository.findByX(param, pageable)).willReturn(pageResult)`
  - → `given(repository.findByX(param)).willReturn(listResult)`
- Update method names in mocks:
  - `given(petTypeRepository.findPetTypes()).willReturn(types)`
  - → `given(petTypeRepository.findAllOrderByName()).willReturn(types)`

#### G. Fix utility classes used by tests

- Update `EntityUtils.java` or similar:
  - Remove JPA-specific exception imports (`ObjectRetrievalFailureException`)
  - Change method signatures from `int id` to `String id`
  - Update ID comparison logic: `entity.getId() == entityId` → `entity.getId().equals(entityId)`
  - Replace JPA exceptions with standard exceptions (`IllegalArgumentException`)

#### H. Update assertions for String IDs

- Change ID assertions:
  - `assertThat(entity.getId()).isNotZero()` → `assertThat(entity.getId()).isNotEmpty()`
  - `assertThat(entity.getId()).isEqualTo(1)` → `assertThat(entity.getId()).isEqualTo("test-id-1")`
  - JSON path assertions: `jsonPath("$.id").value(1)` → `jsonPath("$.id").value("test-id-1")`

### Step 8 — Test file conversion (CRITICAL SECTION)

**This step is often overlooked but essential for successful conversion**

#### A. **COMPILATION CHECK STRATEGY**

- **After each major change, run `mvn test-compile` to catch issues early**
- **Fix compilation errors systematically before proceeding**
- **Don't rely on IDE - Maven compilation reveals all issues**

#### B. **Search and Update ALL test files systematically**

**Use search tools to find and update every occurrence:**

- Search for: `setId\(\d+\)` → Replace with: `setId("test-id-X")`
- Search for: `findById\(\d+\)` → Replace with: `findById("test-id-X")`
- Search for: `\.findPetTypes\(\)` → Replace with: `.findAllOrderByName()`
- Search for: `\.findByLastNameStartingWith\(.*,.*Pageable` → Remove pagination parameter

#### C. Update test annotations and imports

- Replace `@DataJpaTest` with `@SpringBootTest` or appropriate slice test
- Remove `@AutoConfigureTestDatabase` annotations
- Remove `@Transactional` from tests (unless single-partition operations)
- Remove imports from `org.springframework.orm` package

#### D. Fix entity ID usage in ALL test files

**Critical files that MUST be updated (search entire test directory):**

- `*ControllerTests.java` - Path variables, entity creation, mock setup
- `*ServiceTests.java` - Repository interactions, entity IDs
- `EntityUtils.java` - Utility methods for ID handling
- `*FormatterTests.java` - Repository method calls
- `*ValidatorTests.java` - Entity creation with String IDs
- Integration test classes - Test data setup

#### E. **Fix Controller and Service classes affected by repository changes**

- **Update controllers that call repository methods with changed signatures**
- **Update formatters/converters that use repository methods**
- **Common files to check**:
  - `PetTypeFormatter.java` - often calls `findPetTypes()` method
  - `*Controller.java` - may have pagination logic to remove
  - Service classes that use repository methods

#### F. Update repository mocking in tests

- Remove pagination from repository mocks:
  - `given(repository.findByX(param, pageable)).willReturn(pageResult)`
  - → `given(repository.findByX(param)).willReturn(listResult)`
- Update method names in mocks:
  - `given(petTypeRepository.findPetTypes()).willReturn(types)`
  - → `given(petTypeRepository.findAllOrderByName()).willReturn(types)`

#### G. Fix utility classes used by tests

- Update `EntityUtils.java` or similar:
  - Remove JPA-specific exception imports (`ObjectRetrievalFailureException`)
  - Change method signatures from `int id` to `String id`
  - Update ID comparison logic: `entity.getId() == entityId` → `entity.getId().equals(entityId)`
  - Replace JPA exceptions with standard exceptions (`IllegalArgumentException`)

#### H. Update assertions for String IDs

- Change ID assertions:
  - `assertThat(entity.getId()).isNotZero()` → `assertThat(entity.getId()).isNotEmpty()`
  - `assertThat(entity.getId()).isEqualTo(1)` → `assertThat(entity.getId()).isEqualTo("test-id-1")`
  - JSON path assertions: `jsonPath("$.id").value(1)` → `jsonPath("$.id").value("test-id-1")`

### Step 9 — **Runtime Testing and Template Compatibility**

#### **CRITICAL**: Test the running application after compilation success

- **Start the application**: `mvn spring-boot:run`
- **Navigate through all pages** in the web interface to identify runtime errors
- **Common runtime issues after conversion**:
  - Templates trying to access properties that no longer exist (e.g., `vet.specialties`)
  - Service layer not populating transient relationship properties
  - Controllers not using service layer for relationship loading

#### **Template compatibility fixes**:

- **If templates access relationship properties** (e.g., `entity.relatedObjects`):
  - Ensure transient properties exist on entities with proper getters/setters
  - Verify service layer populates these transient properties
  - Update `getNrOfXXX()` methods to use transient lists instead of ID lists
- **Check for SpEL (Spring Expression Language) errors** in logs:
  - `Property or field 'xxx' cannot be found` → Add missing transient property
  - `EL1008E` errors → Service layer not populating relationships

#### **Service layer verification**:

- **Ensure all controllers use service layer** instead of direct repository access
- **Verify service methods populate relationships** before returning entities
- **Test all CRUD operations** through the web interface

### Step 9.5 — **Template Runtime Validation** (CRITICAL)

#### **Systematic Template Testing Process**

After successful compilation and application startup:

1. **Navigate to EVERY page** in the application systematically
2. **Test each template that displays entity data**:
   - List pages (e.g., `/vets`, `/owners`)
   - Detail pages (e.g., `/owners/{id}`, `/vets/{id}`)
   - Forms and edit pages
3. **Look for specific template errors**:
   - `Property or field 'relationshipName' cannot be found on object of type 'EntityName'`
   - `EL1008E` Spring Expression Language errors
   - Empty or missing data where relationships should appear

#### **Template Error Resolution Checklist**

When encountering template errors:

- [ ] **Identify the missing property** from error message
- [ ] **Check if property exists as transient field** in entity
- [ ] **Verify service layer populates the property** before returning entity
- [ ] **Ensure controller uses service layer**, not direct repository access
- [ ] **Test the specific page again** after fixes

#### **Common Template Error Patterns**

- `Property or field 'specialties' cannot be found` → Add `@JsonIgnore private List<Specialty> specialties` to Vet entity
- `Property or field 'pets' cannot be found` → Add `@JsonIgnore private List<Pet> pets` to Owner entity
- Empty relationship data displayed → Service not populating transient properties

### Step 10 — **Systematic Error Resolution Process**

#### When compilation fails:

1. **Run `mvn compile` first** - fix main source issues before tests
2. **Run `mvn test-compile`** - systematically fix each test compilation error
3. **Focus on most frequent error patterns**:
   - `int cannot be converted to String` → Change test constants and entity setters
   - `method X cannot be applied to given types` → Remove pagination parameters
   - `cannot find symbol: method Y()` → Update to new repository method names
   - Method signature conflicts → Rename conflicting methods

### Step 10 — **Systematic Error Resolution Process**

#### When compilation fails:

1. **Run `mvn compile` first** - fix main source issues before tests
2. **Run `mvn test-compile`** - systematically fix each test compilation error
3. **Focus on most frequent error patterns**:
   - `int cannot be converted to String` → Change test constants and entity setters
   - `method X cannot be applied to given types` → Remove pagination parameters
   - `cannot find symbol: method Y()` → Update to new repository method names
   - Method signature conflicts → Rename conflicting methods
#### When runtime fails:

1. **Check application logs** for specific error messages
2. **Look for template/SpEL errors**:
   - `Property or field 'xxx' cannot be found` → Add transient property to entity
   - Missing relationship data → Service layer not populating relationships
3. **Verify service layer usage** in controllers
4. **Test navigation through all application pages**

#### Common error patterns and solutions:

- **`method findByLastNameStartingWith cannot be applied`** → Remove `Pageable` parameter
- **`cannot find symbol: method findPetTypes()`** → Change to `findAllOrderByName()`
- **`incompatible types: int cannot be converted to String`** → Update test ID constants
- **`method getPet(String) is already defined`** → Rename one method (e.g., `getPetByName`)
- **`cannot find symbol: method isNotZero()`** → Change to `isNotEmpty()` for String IDs
- **`Property or field 'specialties' cannot be found`** → Add transient property and populate in service
- **`ClassCastException: reactor.core.publisher.BlockingIterable cannot be cast to java.util.List`** → Fix repository `findAllWithEagerRelationships()` method to use StreamSupport
- **`Unable to make field...BigDecimal.intVal accessible`** → Replace BigDecimal with Double throughout application
- **Health check database failure** → Remove 'db' from health check readiness configuration

#### **Template-Specific Runtime Errors**

- **`Property or field 'XXX' cannot be found on object of type 'YYY'`**:

  - Root cause: Template accessing relationship property that was converted to ID storage
  - Solution: Add transient property to entity + populate in service layer
  - Prevention: Always check template usage before converting relationships

- **`EL1008E` Spring Expression Language errors**:

  - Root cause: Service layer not populating transient properties
  - Solution: Verify `populateRelationships()` methods are called and working
  - Prevention: Test all template navigation after service layer implementation

- **Empty/null relationship data in templates**:
  - Root cause: Controller bypassing service layer or service not populating relationships
  - Solution: Ensure all controller methods use service layer for entity retrieval
  - Prevention: Never return repository results directly to templates

### Step 11 — Validation checklist

After conversion, verify:

- [ ] **Main application compiles**: `mvn compile` succeeds
- [ ] **All test files compile**: `mvn test-compile` succeeds
- [ ] **No compilation errors**: Address every single compilation error
- [ ] **Application starts successfully**: `mvn spring-boot:run` without errors
- [ ] **All web pages load**: Navigate through all application pages without runtime errors
- [ ] **Service layer populates relationships**: Transient properties are correctly set
- [ ] **All template pages render without errors**: Navigate through entire application
- [ ] **Relationship data displays correctly**: Lists, counts, and related objects show properly
- [ ] **No SpEL template errors in logs**: Check application logs during navigation
- [ ] **Transient properties are @JsonIgnore annotated**: Prevents JSON serialization issues
- [ ] **Service layer used consistently**: No direct repository access in controllers for template rendering
- [ ] No remaining `jakarta.persistence` imports
- [ ] All entity IDs are `String` type consistently
- [ ] All repository interfaces extend `CosmosRepository<Entity, String>`
- [ ] Configuration uses `DefaultAzureCredential` for authentication
- [ ] Data seeding component exists and works
- [ ] Test files use String IDs consistently
- [ ] Repository mocks updated for Cosmos methods
- [ ] **No method signature conflicts** in entity classes
- [ ] **All renamed methods updated** in callers (controllers, tests, formatters)

### Common pitfalls to avoid

1. **Not checking compilation frequently** - Run `mvn test-compile` after each major change
2. **Method signature conflicts** - Method overloading issues when converting ID types
3. **Forgetting to update method callers** - When renaming methods, update ALL callers
4. **Missing repository method renames** - Custom repository methods must be updated everywhere called
5. **Using key-based authentication** - Use `DefaultAzureCredential` instead
6. **Mixing Integer and String IDs** - Be consistent with String IDs everywhere, especially in tests
7. **Not updating controller pagination logic** - Remove pagination from controllers when repositories change
8. **Leaving JPA-specific test annotations** - Replace with Cosmos-compatible alternatives
9. **Incomplete test file updates** - Search entire test directory, not just obvious files
10. **Skipping runtime testing** - Always test the running application, not just compilation
11. **Missing service layer** - Don't access repositories directly from controllers
12. **Forgetting transient properties** - Templates may need access to relationship data
13. **Not testing template navigation** - Compilation success doesn't mean templates work
14. **Missing transient properties for templates** - Templates need object access, not just IDs
15. **Service layer bypassing** - Controllers must use services, never direct repository access
16. **Incomplete relationship population** - Service methods must populate ALL transient properties used by templates
17. **Forgetting @JsonIgnore on transient properties** - Prevents serialization issues
18. **@JsonIgnore on persisted fields** - **CRITICAL**: Never use `@JsonIgnore` on fields that need to be stored in Cosmos DB
19. **Authentication serialization errors** - User/Authority entities must be fully serializable without `@JsonIgnore` blocking required fields
20. **BigDecimal reflection issues** - Use alternative data types or JVM arguments for JDK 17+ compatibility
21. **Repository reactive type casting** - Don't cast `findAll()` directly to `List`, use `StreamSupport.stream().collect(Collectors.toList())`
22. **Health check database references** - Remove database dependencies from Spring Boot health checks after JPA removal
23. **Collection type mismatches** - Update service methods to handle String vs object collections consistently

### Debugging compilation issues systematically

If compilation fails after conversion:

1. **Start with main compilation**: `mvn compile` - fix entity and controller issues first
2. **Then test compilation**: `mvn test-compile` - fix each error systematically
3. **Check for remaining `jakarta.persistence` imports** throughout codebase
4. **Verify all test constants use String IDs** - search for `int.*TEST.*ID`
5. **Ensure repository method signatures match** new Cosmos interface
6. **Check for mixed Integer/String ID usage** in entity relationships and tests
7. **Validate all mocking uses correct method names** (`findAllOrderByName()` not `findPetTypes()`)
8. **Look for method signature conflicts** - resolve by renaming conflicting methods
9. **Verify assertion methods work with String IDs** (`isNotEmpty()` not `isNotZero()`)

### Debugging runtime issues systematically

If runtime fails after successful compilation:

1. **Check application startup logs** for initialization errors
2. **Navigate through all pages** to identify template/controller issues
3. **Look for SpEL template errors** in logs:
   - `Property or field 'xxx' cannot be found` → Missing transient property
   - `EL1008E` → Service layer not populating relationships
4. **Verify service layer is being used** instead of direct repository access
5. **Check that transient properties are populated** in service methods
6. **Test all CRUD operations** through the web interface
7. **Verify data seeding worked correctly** and relationships are maintained
8. **Authentication-specific debugging**:
   - `Cannot pass null or empty values to constructor` → Check for `@JsonIgnore` on required fields
   - `BadCredentialsException` → Verify User entity serialization and password field accessibility
   - Check logs for "DomainUserDetailsService" debugging output to trace authentication flow

### **Pro Tips for Success**

- **Compile early and often** - Don't let errors accumulate
- **Use global search and replace** - Find all occurrences of patterns to update
- **Be systematic** - Fix one type of error across all files before moving to next
- **Test method renames carefully** - Ensure all callers are updated
- **Use meaningful String IDs** - "owner-1", "pet-1" instead of random strings
- **Check controller classes** - They often call repository methods that change signatures
- **Always test runtime** - Compilation success doesn't guarantee functional templates
- **Service layer is critical** - Bridge between document storage and template expectations

### **Authentication Troubleshooting Guide** (CRITICAL)

#### **Common Authentication Serialization Errors**:

1. **`Cannot pass null or empty values to constructor`**:

   - **Root Cause**: `@JsonIgnore` preventing required field serialization to Cosmos DB
   - **Solution**: Remove `@JsonIgnore` from all persisted fields (password, authorities, etc.)
   - **Verification**: Check User entity has no `@JsonIgnore` on stored fields

2. **`BadCredentialsException` during login**:

   - **Root Cause**: Password field not accessible during authentication
   - **Solution**: Ensure password field is serializable and accessible in UserDetailsService
   - **Verification**: Add debug logs in `loadUserByUsername` method

3. **Authorities not loading correctly**:

   - **Root Cause**: Authority objects stored as complex entities instead of strings
   - **Solution**: Store authorities as `Set<String>` and convert to `GrantedAuthority` in UserDetailsService
   - **Pattern**:

     ```java
     // In User entity - stored in Cosmos
     @JsonProperty("authorities")
     private Set<String> authorities = new HashSet<>();

     // In UserDetailsService - convert for Spring Security
     List<GrantedAuthority> grantedAuthorities = user
       .getAuthorities()
       .stream()
       .map(SimpleGrantedAuthority::new)
       .collect(Collectors.toList());

     ```

4. **User entity not found during authentication**:
   - **Root Cause**: Repository query methods not working with String IDs
   - **Solution**: Update repository `findOneByLogin` method to work with Cosmos DB
   - **Verification**: Test repository methods independently

#### **Authentication Debugging Checklist**:

- [ ] User entity fully serializable (no `@JsonIgnore` on persisted fields)
- [ ] Password field accessible and not null
- [ ] Authorities stored as `Set<String>`
- [ ] UserDetailsService converts string authorities to `GrantedAuthority`
- [ ] Repository methods work with String IDs
- [ ] Debug logging enabled in authentication service
- [ ] User activation status checked appropriately
- [ ] Test login with known credentials (admin/admin)

### **Common Runtime Issues and Solutions**

#### **Issue 1: Repository Reactive Type Casting Errors**

**Error**: `ClassCastException: reactor.core.publisher.BlockingIterable cannot be cast to java.util.List`

**Root Cause**: Cosmos repositories return reactive types (`Iterable`) but legacy JPA code expects `List`

**Solution**: Convert reactive types properly in repository methods:

```java
// WRONG - Direct casting fails
default List<Entity> customFindMethod() {
    return (List<Entity>) this.findAll(); // ClassCastException!
}

// CORRECT - Convert Iterable to List
default List<Entity> customFindMethod() {
    return StreamSupport.stream(this.findAll().spliterator(), false)
            .collect(Collectors.toList());
}
```

**Files to Check**:

- All repository interfaces with custom default methods
- Any method that returns `List<Entity>` from Cosmos repository calls
- Import `java.util.stream.StreamSupport` and `java.util.stream.Collectors`

#### **Issue 2: BigDecimal Reflection Issues in Java 17+**

**Error**: `Unable to make field private final java.math.BigInteger java.math.BigDecimal.intVal accessible`

**Root Cause**: Java 17+ module system restricts reflection access to BigDecimal internal fields during serialization

**Solutions**:

1. **Replace with Double for simple cases**:

   ```java
   // Before: BigDecimal fields
   private BigDecimal amount;

   // After: Double fields (if precision requirements allow)
   private Double amount;

   ```

2. **Use String for high precision requirements**:

   ```java
   // Store as String, convert as needed
   private String amount; // Store "1500.00"

   public BigDecimal getAmountAsBigDecimal() {
     return new BigDecimal(amount);
   }

   ```

3. **Add JVM argument** (if BigDecimal must be kept):
   ```
   --add-opens java.base/java.math=ALL-UNNAMED
   ```

#### **Issue 3: Health Check Database Dependencies**

**Error**: Application fails health checks looking for removed database components

**Root Cause**: Spring Boot health checks still reference JPA/database dependencies after removal

**Solution**: Update health check configuration:

```yaml
# In application.yml - Remove database from health checks
management:
  health:
    readiness:
      include: 'ping,diskSpace' # Remove 'db' if present
```

**Files to Check**:

- All `application*.yml` configuration files
- Remove any database-specific health indicators
- Check actuator endpoint configurations

#### **Issue 4: Collection Type Mismatches in Services**

**Error**: Type mismatch errors when converting entity relationships to String-based storage

**Root Cause**: Service methods expecting different collection types after entity conversion

**Solution**: Update service methods to handle new entity structure:

```java
// Before: Entity relationships
public Set<RelatedEntity> getRelatedEntities() {
    return entity.getRelatedEntities(); // Direct entity references
}

// After: String-based relationships with conversion
public Set<RelatedEntity> getRelatedEntities() {
    return entity.getRelatedEntityIds()
        .stream()
        .map(relatedRepository::findById)
        .filter(Optional::isPresent)
        .map(Optional::get)
        .collect(Collectors.toSet());
}

### **Enhanced Error Resolution Process**

#### **Common Error Patterns and Solutions**:

1. **Reactive Type Casting Errors**:
   - **Pattern**: `cannot be cast to java.util.List`
   - **Fix**: Use `StreamSupport.stream().collect(Collectors.toList())`
   - **Files**: Repository interfaces with custom default methods

2. **BigDecimal Serialization Errors**:
   - **Pattern**: `Unable to make field...BigDecimal.intVal accessible`
   - **Fix**: Replace with Double, String, or add JVM module opens
   - **Files**: Entity classes, DTOs, data initialization classes

3. **Health Check Database Errors**:
   - **Pattern**: Health check fails looking for database
   - **Fix**: Remove database references from health check configuration
   - **Files**: application.yml configuration files

4. **Collection Type Conversion Errors**:
   - **Pattern**: Type mismatch in entity relationship handling
   - **Fix**: Update service methods to handle String-based entity references
   - **Files**: Service classes, DTOs, entity relationship methods

#### **Enhanced Validation Checklist**:
- [ ] **Repository reactive casting handled**: No ClassCastException on collection returns
- [ ] **BigDecimal compatibility resolved**: Java 17+ serialization works
- [ ] **Health checks updated**: No database dependencies in health configuration
- [ ] **Service layer collection handling**: String-based entity references work correctly
- [ ] **Data seeding completes**: "Data seeding completed" message appears in logs
- [ ] **Application starts fully**: Both frontend and backend accessible
- [ ] **Authentication works**: Can sign in without serialization errors
- [ ] **CRUD operations functional**: All entity operations work through UI

## **Quick Reference: Common Post-Migration Fixes**

### **Top Runtime Issues to Check**

1. **Repository Collection Casting**:
   ```java
   // Fix any repository methods that return collections:
   default List<Entity> customFindMethod() {
       return StreamSupport.stream(this.findAll().spliterator(), false)
               .collect(Collectors.toList());
   }

2. **BigDecimal Compatibility (Java 17+)**:

   ```java
   // Replace BigDecimal fields with alternatives:
   private Double amount; // Or String for high precision

   ```

3. **Health Check Configuration**:
   ```yaml
   # Remove database dependencies from health checks:
   management:
     health:
       readiness:
         include: 'ping,diskSpace'
   ```

### **Authentication Conversion Patterns**

- **Remove `@JsonIgnore` from fields that need Cosmos DB persistence**
- **Store complex objects as simple types** (e.g., authorities as `Set<String>`)
- **Convert between simple and complex types** in service/repository layers

### **Template/UI Compatibility Patterns**

- **Add transient properties** with `@JsonIgnore` for UI access to related data
- **Use service layer** to populate transient relationships before rendering
- **Never return repository results directly** to templates without relationship population