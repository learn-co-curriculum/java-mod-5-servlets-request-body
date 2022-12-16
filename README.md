# Servlet Request Body

## Learning Goals

- Use the Jackson JSON Parser API to convert between JSON and POJO
- Override the `doPost()` method in a servlet
- Extract the request body data from an `HttpServletRequest` object
- Use Postman to issue a POST request to a servlet

## Code Along

## Introduction

In previous lessons we sent request data through
the URL either in the query string or as path parameters when we made a GET request.
A POST request typically sends data through the request
body, thus the data is not part of the URL.

In this lesson, we will override a servlet's `doPost()` method
to extract a JSON-formatted string from the request body, convert the JSON to a POJO,
save the POJO, and generate a response.

## Testing the `doGet()` method in Postman

Let's return to the `ContinentServlet` example and use Postman to test the `doGet()`
method  before moving on to implement the `doPost()` method.

1. Make sure you're running the `Tomcat9` plugin in IntelliJ.
2. In Postman, select `GET` as the request method and enter
   the URL `http://localhost:8080/continents/asia`.
3. Confirm the response body as shown:

![get continents request in postman](https://curriculum-content.s3.amazonaws.com/6036/java-mod-5-request-body/get_continents_request.png)

## Extracting JSON from the HTTP request body

We will override the `doPost()` method to create and add a `Continent` to the hashmap.
Let's add the submerged continent of Zealandia.  The JSON representation is shown below:

```json
{
    "name": "zealandia",
    "area": 4900000,
    "population": 0
}
```

When we issue a POST request, the request body will contain this string, just like we
did in the Postman lesson.

Our servlet will need to extract the JSON-formatted string from the 
`HttpServletRequest` object passed as a parameter to the `doPost()` method.
We can use the following code to do this:

```java
// Read client data from request body
String requestData = req.getReader().lines().collect(Collectors.joining());
```

We then need to convert the JSON-formatted string to an instance of the `Continent`
class so we can add the object to the hashmap.  While there are many libraries
that convert between JSON and Java objects, we will use the **Jackson JSON Parser API**.

## Jackson - JSON Processor for Java

The **Jackson JSON Parser API** provides an easy way to 
convert between JSON and POJO Objects.  We need to update
`pom.xml` with the appropriate dependency:

```xml
 <!-- https://mvnrepository.com/artifact/com.fasterxml.jackson.core/jackson-databind -->
 <dependency>
   <groupId>com.fasterxml.jackson.core</groupId>
   <artifactId>jackson-databind</artifactId>
   <version>2.14.1</version>
 </dependency>
```

Don't forget to reload Maven after making changes to `pom.xml`:

![reload maven icon](https://curriculum-content.s3.amazonaws.com/6036/create-webapp-project/reload_maven.png)

The current version of `pom.xml` should appear as:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>org.example</groupId>
  <artifactId>servlets</artifactId>
  <packaging>war</packaging>
  <version>1.0-SNAPSHOT</version>

  <!-- Set compiler to Java11 -->
  <properties>
    <maven.compiler.source>11</maven.compiler.source>
    <maven.compiler.target>11</maven.compiler.target>
  </properties>

  <name>servlets Maven Webapp</name>
  <url>http://maven.apache.org</url>
  <dependencies>

    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>3.8.1</version>
      <scope>test</scope>
    </dependency>

    <!-- https://mvnrepository.com/artifact/jakarta.servlet/jakarta.servlet-api -->
    <dependency>
      <groupId>javax.servlet</groupId>
      <artifactId>javax.servlet-api</artifactId>
      <version>4.0.1</version>
      <scope>provided</scope>
    </dependency>

    <!-- https://mvnrepository.com/artifact/com.fasterxml.jackson.core/jackson-databind -->
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
      <version>2.14.1</version>
    </dependency>

  </dependencies>

  <build>
    <finalName>servlets</finalName>

    <!-- Tomcat9 Maven Plugin -->
    <plugins>
      <plugin>
        <groupId>org.opoo.maven</groupId>
        <artifactId>tomcat9-maven-plugin</artifactId>
        <version>3.0.1</version>
        <configuration>
          <path>/</path>
        </configuration>
      </plugin>
    </plugins>

  </build>
</project>
```

We can use Jackson's `ObjectMapper` class to convert JSON to a `Continent` class instance:

```java
// create a new Continent from JSON-formatted string contained in the request body
Continent continent = new ObjectMapper().readValue(requestData, Continent.class);
```

We can convert a `Continent` class instance to a JSON string:

```java
String json = new ObjectMapper().writeValueAsString(continent);
```

## Overriding the `doPost()` method to add a new `Continent` to the data store

Finally, we will put everything together to implement the `doPost()` method
to read a `Continent` object from the request body and add the new object
to the hashmap. 

```java
 @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        // Read client data from request body
        String requestData = req.getReader().lines().collect(Collectors.joining());

        // create a new Continent from JSON-formatted string contained in the request body
        Continent continent = new ObjectMapper().readValue(requestData, Continent.class);

        // add the Continent to the hashmap using the abbreviation as the key
        map.put(continent.getName(), continent);

        // send created status and continent formatted as JSON as server response
        resp.setStatus(HttpServletResponse.SC_CREATED);  //status=201
        resp.setContentType("application/json;charset=UTF-8");
        resp.getOutputStream().print(new ObjectMapper().writeValueAsString(continent));
    }
```


We will also update the last print statement in the `doGet()` method
to use Jackson's `ObjectMapper` to create the JSON
representation of the `Continent`, rather than relying on a `toString()`
implementation.

```java
resp.getOutputStream().print(new ObjectMapper().writeValueAsString(continent));
```


The complete version of `ContinentServlet` is shown below:


```java
import com.fasterxml.jackson.databind.ObjectMapper;

import javax.servlet.ServletConfig;
import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.HashMap;
import java.util.stream.Collectors;

@WebServlet(urlPatterns="/continents/*")
public class ContinentServlet extends HttpServlet {

    //key is continent name, value is land area (Km^2)
    private HashMap<String, Continent> map;

    // init() is automatically called after the Servlet is instantiated
    @Override
    public void init(ServletConfig config) throws ServletException {
        super.init(config);
        // data from https://en.wikipedia.org/wiki/Continent
        map = new HashMap<>();
        map.put("asia", new Continent("asia", 44614000, 4700000000L ));
        map.put("africa", new Continent("africa", 30365000, 1400000000L));
        map.put("north_america", new Continent("north america", 24230000, 600000000L ));
        map.put("south_america", new Continent("south america", 17814000, 430000000L ));
        map.put("antarctica", new Continent("antarctica", 14200000, 0L));
        map.put("europe", new Continent("europe", 10000000, 750000000L));
        map.put("oceania", new Continent("australia/oceania", 8510900, 44000000L));
    }

    //http://localhost:8080/continents/{continent_name}
    protected void doGet(
            HttpServletRequest req,
            HttpServletResponse resp)
            throws ServletException, IOException {

        // get the {continent_name} path parameter http://localhost:8080/continents/{continent_name}
        String continent_name = req.getRequestURI().substring("/continents/".length());

        // lookup the continent from the hashmap using the continent name as the key
        Continent continent = map.get(continent_name);

        // generate the response
        if (continent == null) {
            // send error status and message as server response
            resp.setStatus(HttpServletResponse.SC_INTERNAL_SERVER_ERROR);  //status = 500
            resp.setContentType("text/plain;charset=UTF-8");
            resp.getOutputStream().print(String.format("Continent %s not found.", continent_name));
        }
        else {
            // send success status and state formatted as JSON as server response
            resp.setStatus(HttpServletResponse.SC_OK);   //status = 200
            resp.setContentType("application/json;charset=UTF-8");
            resp.getOutputStream().print(new ObjectMapper().writeValueAsString(continent));
        }
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        // Read client data from request body
        String requestData = req.getReader().lines().collect(Collectors.joining());

        // create a new Continent from JSON-formatted string contained in the request body
        Continent continent = new ObjectMapper().readValue(requestData, Continent.class);

        // add the Continent to the hashmap using the abbreviation as the key
        map.put(continent.getName(), continent);

        // send created status and continent formatted as JSON as server response
        resp.setStatus(HttpServletResponse.SC_CREATED);  //status=201
        resp.setContentType("application/json;charset=UTF-8");
        resp.getOutputStream().print(new ObjectMapper().writeValueAsString(continent));
    }
}
```

## Testing the `doPost()` method in Postman

1. Stop and restart the `Tomcat9` plugin in IntelliJ after saving your changes to `ContinentServlet`.
2. In Postman, select `POST` as the request method and enter
   the URL `http://localhost:8080/continents/`.
3. Select `Body`, then from the dropdown select `raw`, then `JSON`.
4. Enter the `JSON` object as the request body:
   ```json
   {
   "name":"zealandia",
   "area":4900000,
   "population":0
   }
   ```
5. Press `Send`.
6. Confirm the response body contains the HTTP created status code 201, along with the JSON
   representation of the new continent that was added to the hashmap.

![post continents request in postman](https://curriculum-content.s3.amazonaws.com/6036/java-mod-5-request-body/post_continents_request.png)

NOTE: Jackson methods `readValue` and `writeValueAsString` require a null-args constructor and setter/getter methods for the POJO class.


Let's confirm the continent was added to the hashman (at least temporarily until you restart the server).

Do a get request with the new continent's name:

~[get request zealandia](https://curriculum-content.s3.amazonaws.com/6036/java-mod-5-request-body/get_request_zealandia.png)


## Code Check

Let's confirm the current version of the code.

The file `pom.xml`:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>org.example</groupId>
  <artifactId>servlets</artifactId>
  <packaging>war</packaging>
  <version>1.0-SNAPSHOT</version>

  <!-- Set compiler to Java11 -->
  <properties>
    <maven.compiler.source>11</maven.compiler.source>
    <maven.compiler.target>11</maven.compiler.target>
  </properties>

  <name>servlets Maven Webapp</name>
  <url>http://maven.apache.org</url>
  <dependencies>

    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>3.8.1</version>
      <scope>test</scope>
    </dependency>

    <!-- https://mvnrepository.com/artifact/jakarta.servlet/jakarta.servlet-api -->
    <dependency>
      <groupId>javax.servlet</groupId>
      <artifactId>javax.servlet-api</artifactId>
      <version>4.0.1</version>
      <scope>provided</scope>
    </dependency>

    <!-- https://mvnrepository.com/artifact/com.fasterxml.jackson.core/jackson-databind -->
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
      <version>2.14.1</version>
    </dependency>

  </dependencies>

  <build>
    <finalName>servlets</finalName>

    <!-- Tomcat9 Maven Plugin -->
    <plugins>
      <plugin>
        <groupId>org.opoo.maven</groupId>
        <artifactId>tomcat9-maven-plugin</artifactId>
        <version>3.0.1</version>
        <configuration>
          <path>/</path>
        </configuration>
      </plugin>
    </plugins>

  </build>
</project>
```

The class `Continent`:

```java
public class Continent {
    private String name;
    private Integer area;
    private Long population;

    public Continent() {}

    public Continent(String name, Integer area, Long population) {
        this.name = name;
        this.area = area;
        this.population = population;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getArea() {
        return area;
    }

    public void setArea(Integer area) {
        this.area = area;
    }

    public Long getPopulation() {
        return population;
    }

    public void setPopulation(Long population) {
        this.population = population;
    }

    @Override
    public String toString() {
        return "{" +
                "\"name\":" + "\"" + name + "\"" + "," +
                "\"area\":" +  area +  "," +
                "\"population\":" + population +
                "}";
    }
}

```

The class `ContinentServlet`:

```java
import com.fasterxml.jackson.databind.ObjectMapper;

import javax.servlet.ServletConfig;
import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.HashMap;
import java.util.stream.Collectors;

@WebServlet(urlPatterns="/continents/*")
public class ContinentServlet extends HttpServlet {

    //key is continent name, value is land area (Km^2)
    private HashMap<String, Continent> map;

    // init() is automatically called after the Servlet is instantiated
    @Override
    public void init(ServletConfig config) throws ServletException {
        super.init(config);
        // data from https://en.wikipedia.org/wiki/Continent
        map = new HashMap<>();
        map.put("asia", new Continent("asia", 44614000, 4700000000L ));
        map.put("africa", new Continent("africa", 30365000, 1400000000L));
        map.put("north_america", new Continent("north america", 24230000, 600000000L ));
        map.put("south_america", new Continent("south america", 17814000, 430000000L ));
        map.put("antarctica", new Continent("antarctica", 14200000, 0L));
        map.put("europe", new Continent("europe", 10000000, 750000000L));
        map.put("oceania", new Continent("australia/oceania", 8510900, 44000000L));
    }

    //http://localhost:8080/continents/{continent_name}
    protected void doGet(
            HttpServletRequest req,
            HttpServletResponse resp)
            throws ServletException, IOException {

        // get the {continent_name} path parameter http://localhost:8080/continents/{continent_name}
        String continent_name = req.getRequestURI().substring("/continents/".length());

        // lookup the continent from the hashmap using the continent name as the key
        Continent continent = map.get(continent_name);

        // generate the response
        if (continent == null) {
            // send error status and message as server response
            resp.setStatus(HttpServletResponse.SC_INTERNAL_SERVER_ERROR);  //status = 500
            resp.setContentType("text/plain;charset=UTF-8");
            resp.getOutputStream().print(String.format("Continent %s not found.", continent_name));
        }
        else {
            // send success status and state formatted as JSON as server response
            resp.setStatus(HttpServletResponse.SC_OK);   //status = 200
            resp.setContentType("application/json;charset=UTF-8");
            resp.getOutputStream().print(new ObjectMapper().writeValueAsString(continent));
        }
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        // Read client data from request body
        String requestData = req.getReader().lines().collect(Collectors.joining());

        // create a new Continent from JSON-formatted string contained in the request body
        Continent continent = new ObjectMapper().readValue(requestData, Continent.class);

        // add the Continent to the hashmap using the abbreviation as the key
        map.put(continent.getName(), continent);

        // send created status and continent formatted as JSON as server response
        resp.setStatus(HttpServletResponse.SC_CREATED);  //status=201
        resp.setContentType("application/json;charset=UTF-8");
        resp.getOutputStream().print(new ObjectMapper().writeValueAsString(continent));
    }
}

```

## Conclusion

A POST request typically passes client data through the request body.
A servlet can extract the request body as a single string using the code:

```java
// Read client data from request body
String requestData = req.getReader().lines().collect(Collectors.joining());
```

We can use the `ObjectMapper` class from the Jackson API to map a JSON-formatted string to a POJO:

```java
// create a new Continent from JSON-formatted string contained in the request body
Continent continent = new ObjectMapper().readValue(requestData, Continent.class);
```

The `ObjectMapper` class can also map a POJO to JSON:

```java
String json = new ObjectMapper().writeValueAsString(continent);
```

## Resources

[Jackson JSON Parser API](https://github.com/FasterXML/jackson)
