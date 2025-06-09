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
        double totalRain = 0, totalRunoff = 0, totalExcess = 0, totalUptake = 0, totalGW = 0;

        try (
            BufferedReader br = new BufferedReader(new FileReader("rainfall.csv"));
            PrintWriter pw = new PrintWriter(new FileWriter("soil_water_balance_output.csv"))
        ) {
            String line = br.readLine(); // skip header
            pw.println("Day,Date,Rainfall in mm,Runoff + Excess Runoff in mm,Crop Water Uptake in mm,Soil Moisture in mm,Percolation to Groundwater in mm,Mass Balance Error");

            int day = 1;

            while ((line = br.readLine()) != null) {
                String[] parts = line.split(",");
                if (parts.length != 2) continue;

                String date = parts[0].trim();
                double rain = Double.parseDouble(parts[1].trim());

                double alpha = getRunoffCoefficient(rain);
                double runoff = alpha * rain;
                double infiltration = rain - runoff;

                double tempSm = sm + infiltration;

                double uptake = Math.min(tempSm, DAILY_CROP_WATER);
                tempSm -= uptake;

                double excess = Math.max(0, tempSm - C);
                double newSm = Math.min(tempSm, C);

                double gw = gamma * newSm;
                double totalDayRunoff = runoff + excess;

                // Mass balance check
                double lhs = rain;
                double rhs = (newSm - sm) + runoff + excess + uptake + gw;
                double error = lhs - rhs;

                // Write to CSV
                pw.printf(Locale.US, "%d,%s,%.2f,%.2f,%.2f,%.2f,%.2f,%.4f\n",
                        day, date, rain, totalDayRunoff, uptake, newSm, gw, error);

                // Track totals
                totalRain += rain;
                totalRunoff += runoff;
                totalExcess += excess;
                totalUptake += uptake;
                totalGW += gw;

                // Update for next day
                sm = newSm;
                day++;
            }

            double finalBalance = sm + totalRunoff + totalExcess + totalUptake + totalGW;
            double overallError = totalRain - finalBalance;

            System.out.println("\n=== Mass Balance Summary ===");
            System.out.printf("Total Rainfall       : %.2f mm\n", totalRain);
            System.out.printf("Final Soil Moisture  : %.2f mm\n", sm);
            System.out.printf("Total Runoff         : %.2f mm\n", totalRunoff);
            System.out.printf("Total Excess Runoff  : %.2f mm\n", totalExcess);
            System.out.printf("Total Uptake         : %.2f mm\n", totalUptake);
            System.out.printf("Total Groundwater    : %.2f mm\n", totalGW);
            System.out.printf("Final Mass Balance Error (LHS - RHS): %.4f mm\n", overallError);

            if (Math.abs(overallError) < 0.01)
                System.out.println("✔ Mass balance validation PASSED.");
            else
                System.out.println("✖ Mass balance validation FAILED.");

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


