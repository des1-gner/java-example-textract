# Amazon Textract OCR with Java on EC2 (Amazon Linux 2023)

This guide shows how to set up Amazon Textract for OCR (Optical Character Recognition) using Java on an EC2 instance running Amazon Linux 2023.

## Prerequisites

1. Launch an EC2 instance with Amazon Linux 2023
2. Create an IAM role with the following permissions and attach it to your EC2 instance

### Required IAM Policy
Create an IAM role with this policy (replace `<your-bucket-name>` with your actual S3 bucket name):

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "textract:DetectDocumentText",
                "textract:AnalyzeDocument"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": "arn:aws:s3:::<your-bucket-name>/*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": "arn:aws:s3:::<your-bucket-name>"
        }
    ]
}
```

## Installation Steps

### 1. Update system and install Java 11
```bash
sudo yum update -y
sudo yum install java-11-amazon-corretto-devel -y

# Verify installation
java -version
javac -version
```

### 2. Install Maven
```bash
sudo yum install maven -y

# Verify installation 
mvn -version
```

### 3. Create Maven project
```bash
cd ~
mvn archetype:generate -DgroupId=com.example -DartifactId=textract-ocr -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false
cd textract-ocr
```

### 4. Create pom.xml file
Create a `pom.xml` file with the following content:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>textract-ocr</artifactId>
  <packaging>jar</packaging>
  <version>1.0-SNAPSHOT</version>
  <name>textract-ocr</name>
  
  <properties>
    <maven.compiler.source>11</maven.compiler.source>
    <maven.compiler.target>11</maven.compiler.target>
  </properties>
  
  <dependencies>
    <dependency>
      <groupId>com.amazonaws</groupId>
      <artifactId>aws-java-sdk-textract</artifactId>
      <version>1.12.529</version>
    </dependency>
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
      <version>2.15.2</version>
    </dependency>
  </dependencies>
  
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-assembly-plugin</artifactId>
        <executions>
          <execution>
            <phase>package</phase>
            <goals>
              <goal>single</goal>
            </goals>
            <configuration>
              <archive>
                <manifest>
                  <mainClass>com.example.TextractOCR</mainClass>
                </manifest>
              </archive>
              <descriptorRefs>
                <descriptorRef>jar-with-dependencies</descriptorRef>
              </descriptorRefs>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

### 5. Create Java source file
Create the directory structure and Java file:

```bash
mkdir -p src/main/java/com/example
```

Create `src/main/java/com/example/TextractOCR.java` with the following content (replace `<your-bucket-name>`, `<your-region>`, and `<your-document-path>` with your actual values):

```java
package com.example;

import com.amazonaws.services.textract.AmazonTextract;
import com.amazonaws.services.textract.AmazonTextractClientBuilder;
import com.amazonaws.services.textract.model.*;
import com.amazonaws.auth.DefaultAWSCredentialsProviderChain;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.io.File;
import java.util.List;

public class TextractOCR {
    public static void main(String[] args) {
        try {
            // Create Textract client
            AmazonTextract textractClient = AmazonTextractClientBuilder.standard()
                    .withCredentials(new DefaultAWSCredentialsProviderChain())
                    .withRegion("<your-region>") // e.g., "us-east-1"
                    .build();
            
            // Set up document
            S3Object s3Object = new S3Object()
                    .withBucket("<your-bucket-name>")
                    .withName("<your-document-path>"); // e.g., "documents/resume.pdf"
            
            Document document = new Document().withS3Object(s3Object);
            
            // For OCR (text extraction)
            DetectDocumentTextRequest request = new DetectDocumentTextRequest()
                    .withDocument(document);
            
            System.out.println("Calling Amazon Textract...");
            DetectDocumentTextResult result = textractClient.detectDocumentText(request);
            System.out.println("Processing response...");
            
            // Process results
            List<Block> blocks = result.getBlocks();
            for (Block block : blocks) {
                if (block.getBlockType().equals("LINE")) {
                    System.out.println(block.getText());
                }
            }
            
            // Save full result to file
            ObjectMapper mapper = new ObjectMapper();
            mapper.writeValue(new File("output.json"), result);
            System.out.println("Results saved to output.json");
            
        } catch (Exception e) {
            System.err.println("Error: " + e.getMessage());
            e.printStackTrace();
        }
    }
}
```

### 6. Build and run
```bash
mvn clean package
java -jar target/textract-ocr-1.0-SNAPSHOT-jar-with-dependencies.jar
```

## Expected Output

The application will:
1. Extract text from your PDF document using Amazon Textract
2. Print each line of text to the console
3. Save the complete response to `output.json`

## Troubleshooting

- **Permission errors**: Ensure your EC2 instance has the correct IAM role attached
- **Region mismatch**: Make sure the region in your code matches your S3 bucket's region
- **File not found**: Verify the S3 bucket name and file path are correct
- **Build errors**: Ensure Java 11 and Maven are properly installed

## Notes

- This example uses `DetectDocumentText` for basic OCR
- For form data extraction, use `AnalyzeDocument` with `FeatureTypes.FORMS`
- For table extraction, use `AnalyzeDocument` with `FeatureTypes.TABLES`
- The application uses IAM roles for authentication (recommended for EC2)
