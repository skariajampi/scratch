```java

import java.util.HashMap;
import java.util.Map;

public class GeoUtils {
    // Earth's radius in meters
    private static final double EARTH_RADIUS = 6371000.0;
    
    /**
     * Generate a new point from a reference point using distance and bearing
     * 
     * @param lat Reference latitude in degrees
     * @param lon Reference longitude in degrees
     * @param distanceKm Distance in kilometers
     * @param bearingDegrees Bearing in degrees (0 = North, 90 = East, etc.)
     * @return double array [latitude, longitude] of the new point
     */
    public static double[] generatePointFromReference(double lat, double lon, double distanceKm, double bearingDegrees) {
        // Convert to radians
        double latRad = Math.toRadians(lat);
        double lonRad = Math.toRadians(lon);
        double bearingRad = Math.toRadians(bearingDegrees);
        double distanceM = distanceKm * 1000;
        
        // Calculate new latitude
        double newLatRad = Math.asin(
            Math.sin(latRad) * Math.cos(distanceM / EARTH_RADIUS) +
            Math.cos(latRad) * Math.sin(distanceM / EARTH_RADIUS) * Math.cos(bearingRad)
        );
        
        // Calculate new longitude
        double newLonRad = lonRad + Math.atan2(
            Math.sin(bearingRad) * Math.sin(distanceM / EARTH_RADIUS) * Math.cos(latRad),
            Math.cos(distanceM / EARTH_RADIUS) - Math.sin(latRad) * Math.sin(newLatRad)
        );
        
        // Convert back to degrees
        double newLat = Math.toDegrees(newLatRad);
        double newLon = Math.toDegrees(newLonRad);
        
        return new double[]{newLat, newLon};
    }
    
    /**
     * Generate a boundary box around a point with given radius
     * 
     * @param lat Center latitude in degrees
     * @param lon Center longitude in degrees
     * @param radiusKm Radius in kilometers
     * @return Map containing top-left and bottom-right coordinates
     */
    public static Map<String, double[]> generateBoundaryBox(double lat, double lon, double radiusKm) {
        // Calculate the four cardinal points
        double[] north = generatePointFromReference(lat, lon, radiusKm, 0);   // North
        double[] east = generatePointFromReference(lat, lon, radiusKm, 90);   // East
        double[] south = generatePointFromReference(lat, lon, radiusKm, 180); // South
        double[] west = generatePointFromReference(lat, lon, radiusKm, 270);  // West
        
        // For boundary box:
        // Top-left: Northwest corner (max latitude, min longitude)
        // Bottom-right: Southeast corner (min latitude, max longitude)
        double topLeftLat = Math.max(north[0], west[0]);  // Should be north[0]
        double topLeftLon = Math.min(west[1], north[1]);  // Should be west[1]
        
        double bottomRightLat = Math.min(south[0], east[0]); // Should be south[0]
        double bottomRightLon = Math.max(east[1], south[1]); // Should be east[1]
        
        // Create and return the boundary box
        Map<String, double[]> boundaryBox = new HashMap<>();
        boundaryBox.put("topLeft", new double[]{topLeftLat, topLeftLon});
        boundaryBox.put("bottomRight", new double[]{bottomRightLat, bottomRightLon});
        boundaryBox.put("north", north);
        boundaryBox.put("south", south);
        boundaryBox.put("east", east);
        boundaryBox.put("west", west);
        boundaryBox.put("center", new double[]{lat, lon});
        
        return boundaryBox;
    }
    
    /**
     * Simplified version that returns only top-left and bottom-right
     */
    public static Map<String, double[]> getSimpleBoundaryBox(double lat, double lon, double radiusKm) {
        Map<String, double[]> fullBox = generateBoundaryBox(lat, lon, radiusKm);
        
        Map<String, double[]> simpleBox = new HashMap<>();
        simpleBox.put("topLeft", fullBox.get("topLeft"));
        simpleBox.put("bottomRight", fullBox.get("bottomRight"));
        
        return simpleBox;
    }
    
    /**
     * Alternative implementation using direct calculation (more efficient for boundary boxes)
     */
    public static Map<String, double[]> generateBoundaryBoxDirect(double lat, double lon, double radiusKm) {
        // Convert radius to radians
        double radiusRad = radiusKm / 6371.0; // Using approximate Earth radius in km
        
        // Calculate latitude boundaries
        double minLat = Math.toDegrees(Math.asin(Math.sin(Math.toRadians(lat)) * Math.cos(radiusRad) - 
                                                Math.cos(Math.toRadians(lat)) * Math.sin(radiusRad)));
        double maxLat = Math.toDegrees(Math.asin(Math.sin(Math.toRadians(lat)) * Math.cos(radiusRad) + 
                                                Math.cos(Math.toRadians(lat)) * Math.sin(radiusRad)));
        
        // Calculate longitude boundaries (accounting for latitude)
        double deltaLon = Math.asin(Math.sin(radiusRad) / Math.cos(Math.toRadians(lat)));
        double minLon = Math.toDegrees(Math.toRadians(lon) - deltaLon);
        double maxLon = Math.toDegrees(Math.toRadians(lon) + deltaLon);
        
        Map<String, double[]> boundaryBox = new HashMap<>();
        boundaryBox.put("topLeft", new double[]{maxLat, minLon});      // NW corner
        boundaryBox.put("bottomRight", new double[]{minLat, maxLon});  // SE corner
        
        return boundaryBox;
    }
    
    // Utility method to print boundary box
    public static void printBoundaryBox(Map<String, double[]> boundaryBox) {
        System.out.println("Geo Boundary Box:");
        System.out.printf("Top-Left: %.6f, %.6f%n", 
            boundaryBox.get("topLeft")[0], boundaryBox.get("topLeft")[1]);
        System.out.printf("Bottom-Right: %.6f, %.6f%n", 
            boundaryBox.get("bottomRight")[0], boundaryBox.get("bottomRight")[1]);
        
        if (boundaryBox.containsKey("north")) {
            System.out.printf("North: %.6f, %.6f%n", boundaryBox.get("north")[0], boundaryBox.get("north")[1]);
            System.out.printf("South: %.6f, %.6f%n", boundaryBox.get("south")[0], boundaryBox.get("south")[1]);
            System.out.printf("East: %.6f, %.6f%n", boundaryBox.get("east")[0], boundaryBox.get("east")[1]);
            System.out.printf("West: %.6f, %.6f%n", boundaryBox.get("west")[0], boundaryBox.get("west")[1]);
        }
    }
    
    // Example usage
    public static void main(String[] args) {
        double centerLat = 40.7128; // New York City
        double centerLon = -74.0060;
        double radiusKm = 10.0;
        
        System.out.println("Using point-by-point calculation:");
        Map<String, double[]> boundaryBox1 = generateBoundaryBox(centerLat, centerLon, radiusKm);
        printBoundaryBox(boundaryBox1);
        
        System.out.println("\nUsing direct calculation:");
        Map<String, double[]> boundaryBox2 = generateBoundaryBoxDirect(centerLat, centerLon, radiusKm);
        printBoundaryBox(boundaryBox2);
        
        System.out.println("\nSimple boundary box:");
        Map<String, double[]> simpleBox = getSimpleBoundaryBox(centerLat, centerLon, radiusKm);
        printBoundaryBox(simpleBox);
    }
}
---
