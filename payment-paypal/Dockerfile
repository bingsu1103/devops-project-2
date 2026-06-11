FROM eclipse-temurin:25-jre-alpine
COPY target/payment-paypal-*SNAPSHOT.jar app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]