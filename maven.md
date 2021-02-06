maven

简单pom.xml

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>cn.dongyuit</groupId>
	<artifactId>sm</artifactId>
	<version>1.0.1</version>
	<name>sm</name>
	<description>sm</description>
	
	<!-- 配置属性 -->
	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<maven.compiler.encoding>UTF-8</maven.compiler.encoding>
		<java.version>1.8</java.version>
		<skipTests>true</skipTests>
	</properties>
	
	<!-- 相关依赖包 -->
	<dependencies>
		<dependency>
		    <groupId>org.bouncycastle</groupId>
		    <artifactId>bcprov-jdk15on</artifactId>
		    <version>1.64</version>
		</dependency>
		<dependency>
		    <groupId>junit</groupId>
		    <artifactId>junit</artifactId>
		    <version>4.13.1</version>
		    <scope>test</scope>
		</dependency>
	</dependencies>
	
	<build>  
	    <plugins>  
	        <plugin>  
	            <groupId>org.apache.maven.plugins</groupId>
	            <artifactId>maven-compiler-plugin</artifactId>  
	                <version>3.1</version>  
	                <configuration>  
	                    <source>1.8</source>  
	                    <target>1.8</target>  
	                </configuration>  
	            </plugin>  
	        </plugins>  
	</build>	
	
	<distributionManagement>
		<repository>
			<id>releases</id>
			<url>http://192.168.5.36:8081/nexus/content/repositories/releases</url>
		</repository>
	</distributionManagement>			
</project>
```

