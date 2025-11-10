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




```
