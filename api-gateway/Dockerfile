FROM maven:3.9.9-eclipse-temurin-21 AS builder

WORKDIR /app

COPY pom.xml .

RUN mvn dependency:go-offline -B #means cache unless change has been detected

COPY src ./src

RUN mvn clean package #create a new package from springboot app this will create single .jar file that has stuff in it

FROM eclipse-temurin:21-jdk AS runner

WORKDIR /app

COPY --from=builder ./app/target/api-gateway-0.0.1-SNAPSHOT.jar ./app.jar

EXPOSE 4004

#to run the jar file
ENTRYPOINT ["java", "-jar", "app.jar"]