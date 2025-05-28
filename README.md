# IIT
A Java-based simulator for daily soil water balance during the Kharif season. Supports deep and shallow soils using rainfall data to compute runoff, crop uptake, soil moisture, and groundwater recharge. Outputs results in CSV format for agricultural analysis.
package soil;

import java.io.*;
import java.util.*;

public class SoilWaterBalanceSimulator {

    // Class to store daily result data
    static class DayData {
        int day;                  // Day number
        double rainfall;          // Rainfall on that day
        double runoff;            // Runoff + excess runoff
        double uptake;            // Water taken by crop
        double soilMoisture;      // Remaining soil moisture
        double groundwater;       // Percolation to groundwater

        // Convert object to CSV line
        public String toCSV() {
            return day + "," + rainfall + "," + runoff + "," + uptake + "," + soilMoisture + "," + groundwater;
        }
    }

    public static void main(String[] args) throws Exception {
        Scanner sc = new Scanner(System.in);

        // Take user input for soil type
        System.out.print("Enter soil type (deep/shallow): ");
        String soilType = sc.nextLine().trim().toLowerCase();

        // Initialize capacity (C) and groundwater coefficient (gamma) based on soil type
        double C = 0, gamma = 0;
        if (soilType.equals("deep")) {
            C = 100;       // Max soil moisture for deep soil
            gamma = 0.2;   // Groundwater coefficient for deep soil
        } else if (soilType.equals("shallow")) {
            C = 42;        // Max soil moisture for shallow soil
            gamma = 0.4;   // Groundwater coefficient for shallow soil
        } else {
            System.out.println("Invalid soil type.");
            return;
        }

        // Load daily rainfall values from CSV file
        List<Double> rainfallData = loadRainData("daily_rainfall_jalgaon_chalisgaon_talegaon_2022.csv");

        // List to store daily results
        List<DayData> results = new ArrayList<>();

        double prevSM = 0;  // Soil moisture from previous day

        // Process each day's rainfall data
        for (int i = 0; i < rainfallData.size(); i++) {
            double rain = rainfallData.get(i);            // Rainfall on current day
            double alpha = getAlpha(rain);                // Runoff coefficient
            double runoff = alpha * rain;                 // Primary runoff
            double infiltration = rain - runoff;          // Water that enters soil

            // Soil moisture before crop uptake
            double smBeforeUptake = Math.min(C, infiltration + prevSM);

            // Crop water uptake (max 4 mm/day)
            double uptake = Math.min(4, smBeforeUptake);

            // Soil moisture after crop uptake
            double smAfterUptake = smBeforeUptake - uptake;

            // Groundwater percolation = gamma * remaining soil moisture
            double gw = gamma * smAfterUptake;

            // Excess water if total exceeds soil capacity
            double excess = (infiltration + prevSM > C) ? (infiltration + prevSM - C) : 0;

            // Total runoff = primary runoff + excess
            double totalRunoff = runoff + excess;

            // Final soil moisture left
            double currentSM = smAfterUptake - gw;

            // Create and store daily data
            DayData day = new DayData();
            day.day = i + 1;
            day.rainfall = rain;
            day.runoff = totalRunoff;
            day.uptake = uptake;
            day.soilMoisture = currentSM;
            day.groundwater = gw;

            results.add(day);       // Add day's data to results
            prevSM = currentSM;     // Update previous soil moisture
        }

        // Save final results to CSV
        saveToCSV(results, "output_" + soilType + ".csv");
        System.out.println("Simulation complete. Output saved as output_" + soilType + ".csv");
    }

    // Method to determine runoff coefficient (alpha) based on rainfall amount
    static double getAlpha(double rain) {
        if (rain < 25) return 0.2;
        else if (rain < 50) return 0.3;
        else if (rain < 75) return 0.4;
        else if (rain < 100) return 0.5;
        else return 0.7;
    }

    // Method to load rainfall data from a CSV file
    static List<Double> loadRainData(String filename) throws Exception {
        List<Double> rainData = new ArrayList<>();
        BufferedReader br = new BufferedReader(new FileReader(filename));
        String line;
        while ((line = br.readLine()) != null) {
            try {
                rainData.add(Double.parseDouble(line.trim()));  // Convert line to number
            } catch (Exception ignored) {}
        }
        br.close();  // Close file
        return rainData;
    }

    // Method to save simulation results to a CSV file
    static void saveToCSV(List<DayData> data, String outputFile) throws Exception {
        PrintWriter pw = new PrintWriter(new FileWriter(outputFile));
        // CSV header
        pw.println("Day,Rainfall in mm,Runoff + excess in mm,Crop water uptake in mm,Soil moisture in mm,Percolation to groundwater in mm");
        for (DayData d : data) {
            pw.println(d.toCSV());  // Write each day's data
        }
        pw.close();  // Close writer
    }
}
