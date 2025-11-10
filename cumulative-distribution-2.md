```java

import java.util.*;

public class CumulativeRateDistributionCalculator {
    
    public static class LocationRate {
        private final String cityName;
        private final double ratePerSecond;
        private final double latitude;
        private final double longitude;
        private final double radiusKm;
        private final int weight;
        private final double cumulativeProbability;
        
        public LocationRate(String cityName, double ratePerSecond, 
                          double latitude, double longitude, double radiusKm,
                          int weight, double cumulativeProbability) {
            this.cityName = cityName;
            this.ratePerSecond = ratePerSecond;
            this.latitude = latitude;
            this.longitude = longitude;
            this.radiusKm = radiusKm;
            this.weight = weight;
            this.cumulativeProbability = cumulativeProbability;
        }
        
        // Getters
        public String getCityName() { return cityName; }
        public double getRatePerSecond() { return ratePerSecond; }
        public double getLatitude() { return latitude; }
        public double getLongitude() { return longitude; }
        public double getRadiusKm() { return radiusKm; }
        public int getWeight() { return weight; }
        public double getCumulativeProbability() { return cumulativeProbability; }
    }
    
    public static class CumulativeDistribution {
        private final List<LocationRate> locationRates;
        private final double totalMsgRate;
        private final int durationSeconds;
        
        public CumulativeDistribution(List<LocationRate> locationRates, 
                                    double totalMsgRate, int durationSeconds) {
            this.locationRates = locationRates;
            this.totalMsgRate = totalMsgRate;
            this.durationSeconds = durationSeconds;
        }
        
        public List<LocationRate> getLocationRates() { return locationRates; }
        public double getTotalMsgRate() { return totalMsgRate; }
        public int getDurationSeconds() { return durationSeconds; }
        
        // Method to select a location based on cumulative distribution
        public LocationRate selectRandomLocation() {
            double randomValue = ThreadLocalRandom.current().nextDouble();
            for (LocationRate locationRate : locationRates) {
                if (randomValue <= locationRate.getCumulativeProbability()) {
                    return locationRate;
                }
            }
            return locationRates.get(locationRates.size() - 1); // Fallback
        }
        
        public void printDistribution() {
            System.out.println("=== CUMULATIVE RATE DISTRIBUTION ===");
            System.out.printf("Total Message Rate: %.0f msg/sec%n", totalMsgRate);
            System.out.printf("Duration: %d seconds%n", durationSeconds);
            System.out.printf("Expected Total: %,.0f messages%n", totalMsgRate * durationSeconds);
            System.out.println("--------------------------------------");
            
            double previousCumulative = 0.0;
            for (LocationRate locationRate : locationRates) {
                System.out.printf("%-20s: Weight: %3d | Rate: %6.1f msg/sec | Range: %.3f - %.3f%n",
                    locationRate.getCityName(),
                    locationRate.getWeight(),
                    locationRate.getRatePerSecond(),
                    previousCumulative,
                    locationRate.getCumulativeProbability());
                previousCumulative = locationRate.getCumulativeProbability();
            }
            System.out.println("======================================");
        }
    }
    
    public static CumulativeDistribution calculateRates(SimulationConfig config) {
        List<LocationRate> locationRates = new ArrayList<>();
        
        // Calculate total weight
        int totalWeight = config.getLocations().stream()
            .mapToInt(SimulationConfig.LocationConfig::getWeight)
            .sum();
        
        // Create cumulative distribution
        double cumulativeProbability = 0.0;
        for (SimulationConfig.LocationConfig location : config.getLocations()) {
            double proportion = (double) location.getWeight() / totalWeight;
            double locationRate = config.getTotalMsgRate() * proportion;
            cumulativeProbability += proportion;
            
            locationRates.add(new LocationRate(
                location.getCityName(),
                locationRate,
                location.getLatitude(),
                location.getLongitude(),
                location.getRadiusKm(),
                location.getWeight(),
                cumulativeProbability
            ));
        }
        
        // Ensure last cumulative probability is exactly 1.0
        if (!locationRates.isEmpty()) {
            LocationRate last = locationRates.get(locationRates.size() - 1);
            locationRates.set(locationRates.size() - 1,
                new LocationRate(
                    last.getCityName(),
                    last.getRatePerSecond(),
                    last.getLatitude(),
                    last.getLongitude(),
                    last.getRadiusKm(),
                    last.getWeight(),
                    1.0
                ));
        }
        
        return new CumulativeDistribution(locationRates, config.getTotalMsgRate(), config.getDurationSeconds());
    }
}
-------------------------


import io.gatling.javaapi.core.*;
import io.gatling.javaapi.http.*;
import java.time.Duration;
import java.util.*;

import static io.gatling.javaapi.core.CoreDsl.*;
import static io.gatling.javaapi.http.HttpDsl.*;

public class CumulativeDistributionSimulation extends Simulation {
    
    private final SimulationConfig config = createUserConfig();
    private final CumulativeRateDistributionCalculator.CumulativeDistribution distribution =
        CumulativeRateDistributionCalculator.calculateRates(config);
    private final VehicleDataGeneratorClient dataGenerator = new VehicleDataGeneratorClient();
    
    private HttpProtocolBuilder httpConf = http
        .baseUrl(System.getProperty("baseUrl", "http://your-api-endpoint.com"))
        .acceptHeader("application/json")
        .contentTypeHeader("application/json");
    
    private SimulationConfig createUserConfig() {
        double totalMsgRate = Double.parseDouble(System.getProperty("msgRate", "2500.0"));
        int durationSeconds = Integer.getInteger("duration", 60);
        
        List<SimulationConfig.LocationConfig> locations = Arrays.asList(
            new SimulationConfig.LocationConfig("London", 51.5074, -0.1278, 25.0, 350),
            new SimulationConfig.LocationConfig("Birmingham", 52.4862, -1.8904, 15.0, 200),
            new SimulationConfig.LocationConfig("Manchester", 53.4808, -2.2426, 12.0, 150),
            new SimulationConfig.LocationConfig("Glasgow", 55.8642, -4.2518, 12.0, 100),
            new SimulationConfig.LocationConfig("Leeds", 53.8008, -1.5491, 10.0, 80),
            new SimulationConfig.LocationConfig("Isle of Skye", 57.3081, -6.2307, 50.0, 20)
        );
        
        return new SimulationConfig(totalMsgRate, durationSeconds, locations);
    }
    
    // Option A: Single Scenario with Cumulative Distribution (Recommended)
    private ScenarioBuilder createCumulativeDistributionScenario() {
        return scenario("Cumulative Distribution Vehicles")
            .exec(session -> {
                try {
                    // Use cumulative distribution to select a random location
                    CumulativeRateDistributionCalculator.LocationRate selectedLocation = 
                        distribution.selectRandomLocation();
                    
                    // Generate coordinates within the selected location's radius
                    CoordinateGenerator.Coordinates coords = 
                        CoordinateGenerator.generateCoordinatesInRadius(
                            selectedLocation.getLatitude(),
                            selectedLocation.getLongitude(), 
                            selectedLocation.getRadiusKm()
                        );
                    
                    // Call your external data-generator
                    String vehicleData = dataGenerator.generateVehicleData(
                        coords.getLatitude(),
                        coords.getLongitude(),
                        selectedLocation.getCityName()
                    );
                    
                    return session
                        .set("vehicleMessage", vehicleData)
                        .set("locationName", selectedLocation.getCityName());
                    
                } catch (Exception e) {
                    System.err.printf("Error generating vehicle data: %s%n", e.getMessage());
                    return session;
                }
            })
            .exec(
                http("Send Vehicle Message")
                    .post("/api/vehicles")
                    .body(StringBody("${vehicleMessage}"))
                    .check(status().is(200))
            )
            .exec(session -> {
                // Track distribution in real-time
                if (session.userId() % 1000 == 0) {
                    String locationName = session.getString("locationName");
                    System.out.printf("Message %d: Location=%s%n", session.userId(), locationName);
                }
                return session;
            });
    }
    
    // Option B: Per-Location Scenarios (Alternative approach)
    private List<PopulationBuilder> createPerLocationPopulations() {
        List<PopulationBuilder> populations = new ArrayList<>();
        
        for (CumulativeRateDistributionCalculator.LocationRate locationRate : 
             distribution.getLocationRates()) {
            
            if (locationRate.getRatePerSecond() < 0.1) {
                System.out.printf("Skipping %s - rate too low: %.2f msg/sec%n", 
                    locationRate.getCityName(), locationRate.getRatePerSecond());
                continue;
            }
            
            ScenarioBuilder locationScenario = scenario(locationRate.getCityName() + " Vehicles")
                .exec(session -> {
                    CoordinateGenerator.Coordinates coords = 
                        CoordinateGenerator.generateCoordinatesInRadius(
                            locationRate.getLatitude(),
                            locationRate.getLongitude(), 
                            locationRate.getRadiusKm()
                        );
                    
                    String vehicleData = dataGenerator.generateVehicleData(
                        coords.getLatitude(),
                        coords.getLongitude(),
                        locationRate.getCityName()
                    );
                    
                    return session.set("vehicleMessage", vehicleData);
                })
                .exec(
                    http("Send " + locationRate.getCityName() + " Vehicle")
                        .post("/api/vehicles")
                        .body(StringBody("${vehicleMessage}"))
                        .check(status().is(200))
                );
            
            populations.add(
                locationScenario.injectOpen(
                    constantUsersPerSec(locationRate.getRatePerSecond())
                        .during(Duration.ofSeconds(config.getDurationSeconds()))
                ).protocols(httpConf)
            );
        }
        
        return populations;
    }
    
    {
        // Print the cumulative distribution
        distribution.printDistribution();
        
        System.out.printf("%n=== STARTING CUMULATIVE DISTRIBUTION SIMULATION ===%n");
        
        // Choose your preferred approach:
        
        // APPROACH 1: Single scenario with cumulative distribution (Recommended)
        // - More realistic: each message independently selects location
        // - Better distribution accuracy
        // - Single scenario, simpler monitoring
        
        setUp(
            createCumulativeDistributionScenario().injectOpen(
                constantUsersPerSec(config.getTotalMsgRate())
                    .during(Duration.ofSeconds(config.getDurationSeconds()))
            ).protocols(httpConf)
        ).maxDuration(Duration.ofSeconds(config.getDurationSeconds() + 10));
        
        /*
        // APPROACH 2: Per-location scenarios
        // - More control: each location has its own rate
        // - Better for monitoring individual location performance
        // - More complex setup
        
        List<PopulationBuilder> populations = createPerLocationPopulations();
        setUp(populations.toArray(new PopulationBuilder[0]))
            .maxDuration(Duration.ofSeconds(config.getDurationSeconds() + 10));
        */
    }
}

-----------------

public class DistributionVerificationTest {
    public static void main(String[] args) {
        // Create test configuration
        SimulationConfig config = new SimulationConfig(
            1000.0, // 1000 msg/sec
            1,      // 1 second (for easy percentage calculation)
            Arrays.asList(
                new SimulationConfig.LocationConfig("London", 51.5074, -0.1278, 25.0, 350),
                new SimulationConfig.LocationConfig("Birmingham", 52.4862, -1.8904, 15.0, 200),
                new SimulationConfig.LocationConfig("Manchester", 53.4808, -2.2426, 12.0, 150),
                new SimulationConfig.LocationConfig("Glasgow", 55.8642, -4.2518, 12.0, 100),
                new SimulationConfig.LocationConfig("Leeds", 53.8008, -1.5491, 10.0, 80),
                new SimulationConfig.LocationConfig("Isle of Skye", 57.3081, -6.2307, 50.0, 20)
            )
        );
        
        // Calculate distribution
        CumulativeRateDistributionCalculator.CumulativeDistribution distribution =
            CumulativeRateDistributionCalculator.calculateRates(config);
        
        distribution.printDistribution();
        
        // Test cumulative distribution selection
        System.out.println("\n=== CUMULATIVE DISTRIBUTION VERIFICATION ===");
        System.out.println("Testing random location selection...");
        
        Map<String, Integer> selectionCounts = new HashMap<>();
        int sampleSize = 100000;
        
        for (int i = 0; i < sampleSize; i++) {
            CumulativeRateDistributionCalculator.LocationRate selected = 
                distribution.selectRandomLocation();
            selectionCounts.merge(selected.getCityName(), 1, Integer::sum);
        }
        
        // Print results
        System.out.printf("%-20s %-12s %-12s %-12s%n", 
            "Location", "Expected %", "Actual %", "Difference");
        System.out.println("--------------------------------------------------------");
        
        for (CumulativeRateDistributionCalculator.LocationRate locationRate : 
             distribution.getLocationRates()) {
            
            double expectedPercentage = (locationRate.getWeight() * 100.0) / 
                distribution.getLocationRates().stream().mapToInt(lr -> lr.getWeight()).sum();
            
            int actualCount = selectionCounts.getOrDefault(locationRate.getCityName(), 0);
            double actualPercentage = (actualCount * 100.0) / sampleSize;
            double difference = actualPercentage - expectedPercentage;
            
            System.out.printf("%-20s %-12.2f %-12.2f %-12.2f%n",
                locationRate.getCityName(),
                expectedPercentage,
                actualPercentage,
                difference);
        }
    }
}

--------------

import java.util.concurrent.ThreadLocalRandom;

public class CoordinateGenerator {
    private static final ThreadLocalRandom RANDOM = ThreadLocalRandom.current();
    
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
    
    public static Coordinates generateCoordinatesInRadius(double centerLat, double centerLon, double radiusKm) {
        if (radiusKm <= 0) {
            // If no radius, return exact center coordinates
            return new Coordinates(centerLat, centerLon);
        }
        
        double radius = radiusKm * 1000; // Convert to meters
        double distance = Math.sqrt(RANDOM.nextDouble()) * radius;
        double bearing = RANDOM.nextDouble() * 2 * Math.PI;
        
        double latRad = Math.toRadians(centerLat);
        double lonRad = Math.toRadians(centerLon);
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
}
```
