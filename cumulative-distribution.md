# cumulative distribution
## real populatin based weights:
```java


List<UKLocation> realPopulationLocations = Arrays.asList(
    new UKLocation("London", 51.5074, -0.1278, 9000),       // ~9 million people
    new UKLocation("Birmingham", 52.4862, -1.8904, 1150),   // ~1.15 million
    new UKLocation("Manchester", 53.4808, -2.2426, 550),    // ~550,000
    new UKLocation("Glasgow", 55.8642, -4.2518, 635),       // ~635,000
    new UKLocation("Leeds", 53.8008, -1.5491, 515),         // ~515,000
    new UKLocation("Liverpool", 53.4084, -2.9916, 486),     // ~486,000
    new UKLocation("Newcastle", 54.9783, -1.6178, 300),     // ~300,000
    new UKLocation("Sheffield", 53.3811, -1.4701, 556),     // ~556,000
    new UKLocation("Bristol", 51.4545, -2.5879, 472),       // ~472,000
    new UKLocation("Belfast", 54.5973, -5.9301, 345),       // ~345,000
    new UKLocation("Edinburgh", 55.9533, -3.1883, 507),     // ~507,000
    new UKLocation("Leicester", 52.6369, -1.1398, 368),     // ~368,000
    new UKLocation("Cardiff", 51.4816, -3.1791, 362),       // ~362,000
    new UKLocation("Coventry", 52.4068, -1.5197, 345),      // ~345,000
    new UKLocation("Nottingham", 52.9548, -1.1581, 331),    // ~331,000
    new UKLocation("Southampton", 50.9097, -1.4044, 255),   // ~255,000
    new UKLocation("Portsmouth", 50.8198, -1.0880, 249),    // ~249,000
    new UKLocation("Brighton", 50.8225, -0.1372, 290),      // ~290,000
    new UKLocation("Reading", 51.4543, -0.9781, 174),       // ~174,000
    new UKLocation("Aberdeen", 57.1497, -2.0943, 198)       // ~198,000
);
-----------------------
import java.util.*;

public class UKLocation {
    private final String name;
    private final double latitude;
    private final double longitude;
    private final int weight;
    private final double radiusKm;

    public UKLocation(String name, double latitude, double longitude, int weight, double radiusKm) {
        this.name = name;
        this.latitude = latitude;
        this.longitude = longitude;
        this.weight = weight;
        this.radiusKm = radiusKm;
    }

    public UKLocation(String name, double latitude, double longitude, int weight) {
        this(name, latitude, longitude, weight, 10.0);
    }

    // Getters
    public String getName() { return name; }
    public double getLatitude() { return latitude; }
    public double getLongitude() { return longitude; }
    public int getWeight() { return weight; }
    public double getRadiusKm() { return radiusKm; }
}
---------

import java.util.*;

public class LocationConfig {
    private static List<UKLocation> locations = Arrays.asList(
        new UKLocation("London", 51.5074, -0.1278, 350, 25.0),
        new UKLocation("Birmingham", 52.4862, -1.8904, 200, 15.0),
        new UKLocation("Manchester", 53.4808, -2.2426, 150, 12.0),
        new UKLocation("Glasgow", 55.8642, -4.2518, 100, 12.0),
        new UKLocation("Leeds", 53.8008, -1.5491, 80, 10.0),
        new UKLocation("Liverpool", 53.4084, -2.9916, 70, 10.0),
        new UKLocation("Isle of Skye", 57.3081, -6.2307, 50, 50.0)
    );

    public static List<UKLocation> getLocations() {
        return new ArrayList<>(locations);
    }
    
    public static void setCustomLocations(List<UKLocation> customLocations) {
        locations = new ArrayList<>(customLocations);
    }
}
--------------

import java.util.*;
import java.util.concurrent.ThreadLocalRandom;

public class LocationGenerator {
    private static final ThreadLocalRandom RANDOM = ThreadLocalRandom.current();
    private final List<LocationProbability> cumulativeDistribution;
    
    public LocationGenerator() {
        this.cumulativeDistribution = createCumulativeDistribution(LocationConfig.getLocations());
    }
    
    public LocationGenerator(List<UKLocation> customLocations) {
        this.cumulativeDistribution = createCumulativeDistribution(customLocations);
    }
    
    public static class Coordinates {
        private final double latitude;
        private final double longitude;
        
        public Coordinates(double latitude, double longitude) {
            this.latitude = latitude;
            this.longitude = longitude;
        }
        
        public double getLatitude() { return latitude; }
        public double getLongitude() { return longitude; }
    }
    
    private List<LocationProbability> createCumulativeDistribution(List<UKLocation> locations) {
        List<LocationProbability> distribution = new ArrayList<>();
        long totalWeight = locations.stream().mapToLong(UKLocation::getWeight).sum();
        double cumulative = 0.0;
        
        for (UKLocation location : locations) {
            double proportion = (double) location.getWeight() / totalWeight;
            cumulative += proportion;
            distribution.add(new LocationProbability(location, cumulative));
            
            System.out.printf("Location: %-20s Weight: %3d Probability: %5.2f%% Cumulative: %5.2f%%%n",
                location.getName(), location.getWeight(), proportion * 100, cumulative * 100);
        }
        
        return distribution;
    }
    
    public Coordinates generateWeightedLocation() {
        double randValue = RANDOM.nextDouble();
        UKLocation selectedLocation = selectLocation(randValue);
        return generateLocationInArea(selectedLocation);
    }
    
    private UKLocation selectLocation(double randValue) {
        for (LocationProbability lp : cumulativeDistribution) {
            if (randValue <= lp.cumulativeProbability) {
                return lp.location;
            }
        }
        return cumulativeDistribution.get(0).location;
    }
    
    private Coordinates generateLocationInArea(UKLocation location) {
        double radius = location.getRadiusKm() * 1000;
        double distance = Math.sqrt(RANDOM.nextDouble()) * radius;
        double bearing = RANDOM.nextDouble() * 2 * Math.PI;
        
        double latRad = Math.toRadians(location.getLatitude());
        double lonRad = Math.toRadians(location.getLongitude());
        double earthRadius = 6378137.0;
        
        double newLatRad = Math.asin(
            Math.sin(latRad) * Math.cos(distance / earthRadius) +
            Math.cos(latRad) * Math.sin(distance / earthRadius) * Math.cos(bearing)
        );
        
        double newLonRad = lonRad + Math.atan2(
            Math.sin(bearing) * Math.sin(distance / earthRadius) * Math.cos(latRad),
            Math.cos(distance / earthRadius) - Math.sin(latRad) * Math.sin(newLatRad)
        );
        
        return new Coordinates(Math.toDegrees(newLatRad), Math.toDegrees(newLonRad));
    }
    
    static class LocationProbability {
        final UKLocation location;
        final double cumulativeProbability;
        
        LocationProbability(UKLocation location, double cumulativeProbability) {
            this.location = location;
            this.cumulativeProbability = cumulativeProbability;
        }
    }
}

-------------

import java.time.Instant;
import java.util.UUID;
import java.util.concurrent.ThreadLocalRandom;

public class VehicleDataGenerator {
    private static final ThreadLocalRandom RANDOM = ThreadLocalRandom.current();
    private static final String LETTERS = "ABCDEFGHIJKLMNOPQRSTUVWXYZ";
    private static final String DIGITS = "0123456789";
    
    public static class VehicleData {
        private final String vehicleId;
        private final String registration;
        private final double latitude;
        private final double longitude;
        private final String timestamp;
        private final String imageUrl;
        
        public VehicleData(String vehicleId, String registration, double latitude, 
                          double longitude, String timestamp, String imageUrl) {
            this.vehicleId = vehicleId;
            this.registration = registration;
            this.latitude = latitude;
            this.longitude = longitude;
            this.timestamp = timestamp;
            this.imageUrl = imageUrl;
        }
        
        public String toJson() {
            return String.format(
                "{" +
                "\"vehicleId\": \"%s\"," +
                "\"registration\": \"%s\"," +
                "\"latitude\": %.6f," +
                "\"longitude\": %.6f," +
                "\"timestamp\": \"%s\"," +
                "\"imageUrl\": \"%s\"" +
                "}",
                vehicleId, registration, latitude, longitude, timestamp, imageUrl
            );
        }
    }
    
    public static String generateVehicleReg() {
        String part1 = randomString(LETTERS, 2);
        String part2 = randomString(DIGITS, 2);
        String part3 = randomString(LETTERS, 3);
        return part1 + part2 + " " + part3;
    }
    
    private static String randomString(String source, int length) {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < length; i++) {
            sb.append(source.charAt(RANDOM.nextInt(source.length())));
        }
        return sb.toString();
    }
    
    public static String generateVehicleId() {
        return UUID.randomUUID().toString();
    }
    
    public static String generateTimestamp() {
        return Instant.now().toString();
    }
}

---------------


import io.gatling.javaapi.core.*;
import io.gatling.javaapi.http.*;
import java.time.Duration;

import static io.gatling.javaapi.core.CoreDsl.*;
import static io.gatling.javaapi.http.HttpDsl.*;

public class RateAndDurationSimulation extends Simulation {

    // Configuration from system properties with defaults
    private static final int DURATION_SECONDS = Integer.getInteger("duration", 60);
    private static final double RATE_PER_SECOND = Double.parseDouble(System.getProperty("rate", "1000.0"));
    private static final String BASE_URL = System.getProperty("baseUrl", "http://your-api-endpoint.com");
    
    // Location generator - uses probabilistic approach
    private final LocationGenerator locationGenerator = new LocationGenerator();
    
    static {
        printSimulationConfig();
    }
    
    private static void printSimulationConfig() {
        System.out.println("=== GATLING SIMULATION CONFIGURATION ===");
        System.out.println("Duration: " + DURATION_SECONDS + " seconds");
        System.out.println("Rate: " + RATE_PER_SECOND + " messages/second");
        System.out.println("Base URL: " + BASE_URL);
        System.out.println("Expected total messages: " + (DURATION_SECONDS * RATE_PER_SECOND));
        System.out.println("Location distribution:");
        System.out.println("=========================================");
    }

    // HTTP configuration
    private HttpProtocolBuilder httpConf = http
        .baseUrl(BASE_URL)
        .acceptHeader("application/json")
        .contentTypeHeader("application/json")
        .userAgentHeader("Gatling-Vehicle-Simulator");

    // Scenario definition
    private ScenarioBuilder scn = scenario("Vehicle Message Simulation")
        .exec(session -> {
            // Generate random location based on weights for EACH message
            LocationGenerator.Coordinates coords = locationGenerator.generateWeightedLocation();
            
            // Create vehicle data
            String vehicleId = VehicleDataGenerator.generateVehicleId();
            VehicleDataGenerator.VehicleData vehicleData = new VehicleDataGenerator.VehicleData(
                vehicleId,
                VehicleDataGenerator.generateVehicleReg(),
                coords.getLatitude(),
                coords.getLongitude(),
                VehicleDataGenerator.generateTimestamp(),
                "https://example.com/vehicles/" + vehicleId + ".jpg"
            );
            
            return session.set("vehicleMessage", vehicleData.toJson());
        })
        .exec(
            http("Send Vehicle Message")
                .post("/api/vehicles")
                .body(StringBody("${vehicleMessage}"))
                .check(status().is(200))
                .check(bodyString().saveAs("responseBody"))
        )
        .exec(session -> {
            // Optional: Log progress (every 1000th message)
            long userId = session.userId();
            if (userId % 1000 == 0) {
                System.out.println("Progress: Sent " + userId + " messages");
            }
            return session;
        });

    {
        // Simulation setup with configurable duration and rate
        setUp(
            scn.injectOpen(
                constantUsersPerSec(RATE_PER_SECOND).during(Duration.ofSeconds(DURATION_SECONDS))
            )
        ).protocols(httpConf)
         .maxDuration(Duration.ofSeconds(DURATION_SECONDS + 10)); // Buffer for completion
    }
}


----------


import io.gatling.javaapi.core.*;
import io.gatling.javaapi.http.*;
import java.time.Duration;
import java.util.Arrays;
import java.util.List;

import static io.gatling.javaapi.core.CoreDsl.*;
import static io.gatling.javaapi.http.HttpDsl.*;

public class CustomLocationsSimulation extends Simulation {

    private static final int DURATION_SECONDS = Integer.getInteger("duration", 60);
    private static final double RATE_PER_SECOND = Double.parseDouble(System.getProperty("rate", "1000.0"));
    
    // Define your custom locations with weights
    private final List<UKLocation> customLocations = Arrays.asList(
        new UKLocation("London", 51.5074, -0.1278, 400),      // 40%
        new UKLocation("Birmingham", 52.4862, -1.8904, 300),  // 30%
        new UKLocation("Manchester", 53.4808, -2.2426, 200),  // 20%
        new UKLocation("Isle of Skye", 57.3081, -6.2307, 100) // 10%
    );
    
    private final LocationGenerator locationGenerator = new LocationGenerator(customLocations);
    
    private HttpProtocolBuilder httpConf = http
        .baseUrl(System.getProperty("baseUrl", "http://your-api-endpoint.com"))
        .acceptHeader("application/json")
        .contentTypeHeader("application/json");

    private ScenarioBuilder scn = scenario("Custom Locations Simulation")
        .exec(session -> {
            LocationGenerator.Coordinates coords = locationGenerator.generateWeightedLocation();
            String vehicleId = VehicleDataGenerator.generateVehicleId();
            
            String vehicleJson = String.format(
                "{\"vehicleId\": \"%s\", \"registration\": \"%s\", " +
                "\"latitude\": %.6f, \"longitude\": %.6f, " +
                "\"timestamp\": \"%s\", \"imageUrl\": \"%s\"}",
                vehicleId,
                VehicleDataGenerator.generateVehicleReg(),
                coords.getLatitude(),
                coords.getLongitude(),
                VehicleDataGenerator.generateTimestamp(),
                "https://example.com/vehicles/" + vehicleId + ".jpg"
            );
            
            return session.set("vehicleMessage", vehicleJson);
        })
        .exec(
            http("Send Vehicle Message")
                .post("/api/vehicles")
                .body(StringBody("${vehicleMessage}"))
                .check(status().is(200))
        );

    {
        setUp(
            scn.injectOpen(
                constantUsersPerSec(RATE_PER_SECOND).during(Duration.ofSeconds(DURATION_SECONDS))
            )
        ).protocols(httpConf);
    }
}

----------------------


```
---
