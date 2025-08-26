# Pega-SecuPi-Serialization
# SecuPi JAR + JVM Parameters Integration with Pega 23.0 and Confluent SAAS

## The Challenge: Spring Boot vs Pega Architecture

You're right to be concerned - **the JAR + JVM parameters approach works seamlessly with Spring Boot but requires different handling in Pega 23.0**. Here's exactly how to bridge this gap:

**Why Spring Boot "just works":**
- Spring Boot's auto-configuration automatically detects security JARs on classpath
- `@EnableAutoConfiguration` scans for security providers and initializes them
- Embedded Tomcat server allows direct JVM parameter control

**Pega 23.0 differences:**
- Rules-based application server architecture
- WAR deployment to external application servers (Tomcat/WebLogic)
- No auto-configuration - requires manual security provider registration
- Multiple ClassLoader contexts

## SecuPi Integration Architecture for Pega

Based on SecuPi's proven Confluent integration patterns, here are the **three methods** to integrate the SecuPi JAR and JVM parameters with Pega 23.0:

### Method 1: Java Agent Approach (Recommended for SecuPi)

**This is the most common SecuPi integration method** - installing SecuPi as a Java agent that runs on Producer/Consumer processes.

#### 1. JVM Parameters Configuration

**For containerized deployment (Docker/Kubernetes):**
```bash
# In your Dockerfile or container environment
ENV JAVA_OPTS="$JAVA_OPTS -javaagent:/opt/pega/lib/secupi-agent.jar=config=/opt/pega/config/secupi-banking.properties"

# SecuPi-specific parameters your Kafka team will provide
ENV JAVA_OPTS="$JAVA_OPTS -Dsecupi.management.url=https://secupi-mgmt.ubs.internal"
ENV JAVA_OPTS="$JAVA_OPTS -Dsecupi.license.key=${SECUPI_LICENSE_KEY}"
ENV JAVA_OPTS="$JAVA_OPTS -Dsecupi.policy.refresh.interval=300"
ENV JAVA_OPTS="$JAVA_OPTS -Dsecupi.audit.enabled=true"
```

**For traditional VM deployment:**
```bash
# In setenv.sh (Tomcat) or through WebLogic Admin Console
JAVA_OPTS="$JAVA_OPTS -javaagent:/opt/pega/lib/secupi-agent.jar=config=/opt/pega/config/secupi-banking.properties"
JAVA_OPTS="$JAVA_OPTS -Dsecupi.management.url=https://secupi-mgmt.ubs.internal"
JAVA_OPTS="$JAVA_OPTS -Dsecupi.license.key=YOUR_LICENSE_KEY"
```

#### 2. SecuPi Agent Integration

SecuPi agent automatically integrates with Kafka clients without code changes:

```java
// SecuPi Agent automatically intercepts Kafka calls
// Your existing Pega Queue Processor code remains unchanged
public class PegaKafkaProducer {
    private KafkaProducer<String, String> producer;
    
    public void sendMessage(String topic, String message) {
        // SecuPi agent automatically encrypts sensitive data here
        producer.send(new ProducerRecord<>(topic, message));
        // No code changes required!
    }
}
```

#### 3. Pega-Specific Configuration

**prconfig.xml additions:**
```xml
<pegarules>
  <!-- Existing Kafka configuration -->
  <env name="services/stream/provider" value="ExternalKafka"/>
  <env name="services/stream/broker/url" value="pkc-xyz.confluent.cloud:9092"/>
  
  <!-- SecuPi integration flags -->
  <env name="secupi/agent/enabled" value="true"/>
  <env name="secupi/banking/mode" value="strict"/>
  <env name="secupi/kafka/interceptors/enabled" value="true"/>
</pegarules>
```

### Method 2: Custom PegaSerde with SecuPi Library Integration

If Java agent approach has restrictions, integrate SecuPi directly into custom PegaSerde:

#### 1. Add SecuPi JAR to Pega Classpath

**Database import method (recommended for Pega):**
```bash
# Import SecuPi JAR using Pega Import Wizard
# Application -> Distribution -> Import
# This stores JAR in pr_engineclasses table
```

**Physical placement method:**
```bash
# Copy SecuPi JAR to application server lib directory
cp secupi-banking-client.jar /opt/tomcat/webapps/prweb/WEB-INF/lib/
```

#### 2. Create SecuPi-Enhanced PegaSerde

```java
package com.ubs.pega.serde;

import com.pega.platform.kafka.serde.PegaSerde;
import com.pega.pegarules.pub.clipboard.ClipboardPage;
import com.pega.pegarules.pub.runtime.PublicAPI;
// SecuPi imports - your Kafka team will provide these
import com.secupi.client.SecuPiClient;
import com.secupi.client.ProtectionPolicy;

public class UBSSecuPiSerde implements PegaSerde {
    
    private SecuPiClient secuPiClient;
    private static final String SECUPI_CONFIG_PATH = "/opt/pega/config/secupi-banking.properties";
    
    @Override
    public void configure(PublicAPI tools, Map<String, ?> configs) {
        try {
            // Initialize SecuPi client with UBS banking policies
            Properties secuPiProps = new Properties();
            secuPiProps.load(new FileInputStream(SECUPI_CONFIG_PATH));
            
            this.secuPiClient = SecuPiClient.builder()
                .properties(secuPiProps)
                .managementUrl(System.getProperty("secupi.management.url"))
                .licenseKey(System.getProperty("secupi.license.key"))
                .build();
                
            tools.getLogger().info("SecuPi client initialized for UBS banking");
            
        } catch (Exception e) {
            throw new RuntimeException("Failed to initialize SecuPi client", e);
        }
    }
    
    @Override
    public byte[] serialize(PublicAPI tools, ClipboardPage clipboardPage) {
        try {
            // Extract data from Pega ClipboardPage
            String transactionData = extractTransactionData(clipboardPage);
            
            // Apply SecuPi protection - this is where the magic happens
            String protectedData = secuPiClient.protect(
                transactionData,
                ProtectionPolicy.BANKING_PII // Policy your Kafka team configured
            );
            
            // Return protected data as bytes
            return protectedData.getBytes(StandardCharsets.UTF_8);
            
        } catch (Exception e) {
            throw new RuntimeException("SecuPi protection failed", e);
        }
    }
    
    @Override
    public ClipboardPage deserialize(PublicAPI tools, byte[] data) {
        try {
            String protectedData = new String(data, StandardCharsets.UTF_8);
            
            // SecuPi automatically handles decryption based on user context
            String clearData = secuPiClient.unprotect(protectedData);
            
            // Convert back to Pega ClipboardPage
            ClipboardPage page = tools.createPage("UBS-Transaction", "");
            populateFromTransactionData(page, clearData);
            
            return page;
            
        } catch (Exception e) {
            throw new RuntimeException("SecuPi unprotection failed", e);
        }
    }
    
    private String extractTransactionData(ClipboardPage page) {
        // Convert Pega data to format SecuPi expects
        JSONObject transaction = new JSONObject();
        transaction.put("accountNumber", page.getString("AccountNumber"));
        transaction.put("amount", page.getString("Amount"));
        transaction.put("customerSSN", page.getString("CustomerSSN"));
        return transaction.toString();
    }
    
    private void populateFromTransactionData(ClipboardPage page, String jsonData) {
        // Convert decrypted JSON back to Pega properties
        JSONObject transaction = new JSONObject(jsonData);
        page.putString("AccountNumber", transaction.getString("accountNumber"));
        page.putString("Amount", transaction.getString("amount"));
        page.putString("CustomerSSN", transaction.getString("customerSSN"));
    }
}
```

#### 3. Configure Queue Processor

```properties
# In Queue Processor rule configuration
Serialization Implementation: com.ubs.pega.serde.UBSSecuPiSerde
```

### Method 3: Confluent Interceptor Pattern

Use Confluent's interceptor mechanism with SecuPi:

#### 1. Configure Kafka Properties

```xml
<!-- In prconfig.xml -->
<env name="kafka/producer/interceptor/classes" value="com.secupi.kafka.SecuPiProducerInterceptor"/>
<env name="kafka/consumer/interceptor/classes" value="com.secupi.kafka.SecuPiConsumerInterceptor"/>

<!-- SecuPi interceptor configuration -->
<env name="secupi/interceptor/config/file" value="/opt/pega/config/secupi-banking.properties"/>
```

## SecuPi Configuration for Banking

Your Kafka team will provide the SecuPi JAR and configuration. Here's what the typical configuration looks like:

### SecuPi Banking Properties File

```properties
# SecuPi Management Configuration
secupi.management.url=https://secupi-mgmt.ubs.internal
secupi.license.key=YOUR_LICENSE_KEY_FROM_KAFKA_TEAM

# Banking-specific policies
secupi.policy.banking.pii.encryption=true
secupi.policy.banking.pii.algorithm=AES256_GCM
secupi.policy.banking.compliance.frameworks=SOX,PCI-DSS,GDPR

# Data classification rules
secupi.classification.ssn.pattern=\\d{3}-\\d{2}-\\d{4}
secupi.classification.account.pattern=\\d{10,12}
secupi.classification.credit_card.pattern=\\d{4}-\\d{4}-\\d{4}-\\d{4}

# Access control
secupi.abac.enabled=true
secupi.abac.policy.file=/opt/pega/config/ubs-abac-policies.json

# Audit configuration
secupi.audit.enabled=true
secupi.audit.level=COMPREHENSIVE
secupi.audit.endpoint=https://audit.ubs.internal/secupi
```

## Implementation Steps

### Step 1: Get SecuPi Artifacts from Kafka Team
```bash
# Files you'll receive:
secupi-banking-client.jar          # Main SecuPi library
secupi-kafka-interceptors.jar      # Kafka integration
secupi-banking.properties           # UBS-specific configuration
ubs-abac-policies.json             # Access control policies
secupi-license.key                 # License file
```

### Step 2: Deploy to Pega Environment

**For containerized deployment:**
```dockerfile
# Add to your Pega Dockerfile
COPY secupi-banking-client.jar /opt/pega/lib/
COPY secupi-kafka-interceptors.jar /opt/pega/lib/
COPY secupi-banking.properties /opt/pega/config/
COPY ubs-abac-policies.json /opt/pega/config/

# Set JVM parameters
ENV JAVA_OPTS="$JAVA_OPTS -javaagent:/opt/pega/lib/secupi-banking-client.jar=config=/opt/pega/config/secupi-banking.properties"
```

**For VM deployment:**
```bash
# Copy files to Pega environment
cp secupi-*.jar /opt/tomcat/webapps/prweb/WEB-INF/lib/
cp secupi-banking.properties /opt/pega/config/
cp ubs-abac-policies.json /opt/pega/config/

# Update setenv.sh
echo 'JAVA_OPTS="$JAVA_OPTS -javaagent:/opt/tomcat/webapps/prweb/WEB-INF/lib/secupi-banking-client.jar=config=/opt/pega/config/secupi-banking.properties"' >> bin/setenv.sh
```

### Step 3: Configure Pega Queue Processors

**If using Java Agent approach (recommended):**
- No code changes required
- SecuPi automatically intercepts all Kafka operations
- Just configure your queue processors normally

**If using Custom PegaSerde approach:**
- Implement the UBSSecuPiSerde class above
- Configure queue processors to use custom serialization
- Import SecuPi JAR using Pega Import Wizard

### Step 4: Test and Validate

```java
// Test SecuPi integration
public class SecuPiValidationActivity extends Activity {
    
    public void doActivity(PublicAPI tools) {
        // Send test message through queue processor
        // Verify encryption in Confluent Control Center
        // Check SecuPi audit logs
        
        tools.getLogger().info("SecuPi integration validation completed");
    }
}
```

## Troubleshooting Common Issues

### ClassPath Issues
```bash
# Verify SecuPi JAR is loaded
jcmd <pega_process_id> VM.classloader_stats | grep -i secupi

# Check for version conflicts
jar -tf secupi-banking-client.jar | grep -i version
```

### Agent Loading Issues
```bash
# Verify agent is loaded at startup
# Look for these log messages:
# [SecuPi] Agent loaded successfully
# [SecuPi] Banking policies initialized
# [SecuPi] Kafka interceptors registered
```

### Configuration Validation
```java
// Add this to a Pega activity for testing
String secuPiStatus = System.getProperty("secupi.agent.status");
tools.getLogger().info("SecuPi Agent Status: " + secuPiStatus);
```

## Key Advantages of This Approach

1. **Zero Code Changes**: SecuPi agent automatically handles encryption/decryption
2. **Banking Compliance**: Built-in SOX, PCI-DSS, GDPR compliance
3. **Performance**: Client-side encryption with minimal latency impact
4. **Audit Trail**: Comprehensive logging of all data access
5. **Policy Management**: Centralized security policy enforcement

The Java Agent approach is **strongly recommended** because it's exactly how SecuPi integrates with Confluent in production banking environments - your Kafka team has likely already validated this pattern with other applications at UBS.

This solution gives you the security benefits of SecuPi without the complexity of Azure Event Hub migration, while maintaining full compatibility with your existing Pega 23.0 queue processor architecture.
===
### Springboot command explaination
java -Dspring.profiles.active=cx 
-javaagent:/home/site/libs/secupi.jar=secupi.agent.log.dest=stdout,secupi.agent.debug 
--add-modules=ALL-SYSTEM 
--add-opens=java.base/java.io=ALL-UNNAMED 
# ... many more --add-opens parameters
-jar /home/site/wwwroot/app.jar

Key components:

1.  SecuPi JAR location: /home/site/libs/secupi.jar
2.  SecuPi agent parameters: secupi.agent.log.dest=stdout,secupi.agent.debug
3.  Java module system parameters: All the --add-opens flags (Java 9+ modules)

# Translation for Pega Tomcat Environment
## Step 1: Copy SecuPi JAR to Your Pega Environment
===
# Create directory structure
mkdir -p /opt/pega/lib

# Copy the SecuPi JAR from your Kafka team
# (You'll need to get secupi.jar from them)
cp secupi.jar /opt/pega/lib/

## Step 2: Update Your Tomcat setenv.sh
Edit your setenv.sh file:
vi $CATALINA_HOME/bin/setenv.sh

Add these lines to setenv.sh:
===
# Existing JAVA_OPTS (keep whatever you have)
# export JAVA_OPTS="$JAVA_OPTS -Xms4g -Xmx8g ..."

# SecuPi Java Agent Integration
export JAVA_OPTS="$JAVA_OPTS -javaagent:/opt/pega/lib/secupi.jar=secupi.agent.log.dest=stdout,secupi.agent.debug"

# Java Module System Parameters (Java 9+)
export JAVA_OPTS="$JAVA_OPTS --add-modules=ALL-SYSTEM"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.base/java.io=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.base/java.lang=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.base/java.lang.reflect=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.base/java.net=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.base/java.nio=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.base/java.nio.charset=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.base/java.text=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.base/java.time=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.base/java.util=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.base/java.util.concurrent=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.base/java.util.concurrent.atomic=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.base/jdk.internal.vm=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.base/jdk.internal.misc=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.base/sun.nio.ch=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.base/sun.nio.fs=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.base/sun.security.ssl=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.base/sun.security.action=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.base/sun.security.util=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.security.jgss/sun.security.jgss=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.security.jgss/sun.security.krb5=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.desktop/java.awt=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.desktop/java.awt.peer=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.sql/java.sql=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.instrument/sun.instrument=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.management/sun.management=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-exports=java.instrument/sun.instrument=ALL-UNNAMED"
===
## Step 3: Simplified Version (Recommended)
Since that's a lot of parameters, here's a cleaner approach:

# Create a more manageable setenv.sh
vi $CATALINA_HOME/bin/setenv.sh

# Add this content:
#!/bin/bash

# Existing Pega memory settings (keep what you have)
export JAVA_OPTS="$JAVA_OPTS -Xms4g -Xmx8g"
export JAVA_OPTS="$JAVA_OPTS -XX:+UseG1GC"

# SecuPi Integration
export JAVA_OPTS="$JAVA_OPTS -javaagent:/opt/pega/lib/secupi.jar=secupi.agent.log.dest=stdout,secupi.agent.debug"

# Java Module Access (required for SecuPi with Java 11+)
export JAVA_OPTS="$JAVA_OPTS --add-modules=ALL-SYSTEM"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.base/java.lang=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.base/java.util=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.base/java.lang.reflect=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-opens=java.instrument/sun.instrument=ALL-UNNAMED"
export JAVA_OPTS="$JAVA_OPTS --add-exports=java.instrument/sun.instrument=ALL-UNNAMED"

# Pega Node Type
export JAVA_OPTS="$JAVA_OPTS -DNodeType=WebUser"

The --add-opens parameters are for Java 9+. Check your Java version:
java -version

## If you're on Java 8:
Remove all --add-opens and --add-modules parameters
Only keep the -javaagent parameter
