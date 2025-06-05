# IIT
A Java-based simulator for daily soil water balance during the Kharif season. Supports deep and shallow soils using rainfall data to compute runoff, crop uptake, soil moisture, and groundwater recharge. Outputs results in CSV format for agricultural analysis.

package Soilwb;

import java.io.*;
import java.util.*;

public class SoilWaterBalanceSimulator {

    static final double DAILY_CROP_WATER = 4.0;

    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        System.out.print("Enter soil type (deep/shallow): ");
        String soilType = scanner.nextLine().trim().toLowerCase();

        double C, gamma;

        if (soilType.equals("deep")) {
            C = 100.0;
            gamma = 0.2;
        } else if (soilType.equals("shallow")) {
            C = 42.0;
            gamma = 0.4;
        } else {
            System.out.println("Invalid soil type.");
            return;
        }

        double sm = 0.0; // initial soil moisture

        try (BufferedReader br = new BufferedReader(new FileReader("rainfall.csv"));
             PrintWriter pw = new PrintWriter(new FileWriter("soil_water_balance_output.csv"))) {

            String line = br.readLine(); // skip header

            // Write CSV header
            pw.println("Day,Date,Rainfall in mm,Runoff + Excess Runoff in mm,Crop Water Uptake in mm,Soil Moisture in mm,Percolation to Groundwater in mm");

            int day = 1;

            while ((line = br.readLine()) != null) {
                String[] parts = line.split(",");
                String date = parts[0];
                double rain = Double.parseDouble(parts[1]);

                double alpha = getRunoffCoefficient(rain);
                double runoff = alpha * rain;
                double infiltration = rain - runoff;

                double uptake = Math.min(sm, DAILY_CROP_WATER);
                double newSm = Math.min(sm + infiltration - uptake, C);
                double excess = Math.max(0, sm + infiltration - uptake - C);
                double totalRunoff = runoff + excess;
                double gw = gamma * newSm;

                pw.printf(Locale.US, "%d,%s,%.2f,%.2f,%.2f,%.2f,%.2f\n",
                        day, date, rain, totalRunoff, uptake, newSm, gw);

                sm = newSm;
                day++;
            }

            System.out.println("CSV file 'soil_water_balance_output.csv' generated successfully.");

        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    static double getRunoffCoefficient(double rain) {
        if (rain < 25) return 0.2;
        else if (rain < 50) return 0.3;
        else if (rain < 75) return 0.4;
        else if (rain < 100) return 0.5;
        else return 0.7;
    }
}


