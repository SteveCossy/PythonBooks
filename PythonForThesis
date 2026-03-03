Kia ora Steve!

This is a solid research plan. Moving from raw logs to aggregated metrics is exactly how you build the "rigor" mentioned in your thesis.

However, to visualize the metrics you listed in **8.3**, we have a small problem: **our current CSV only contains successful data packet transmissions.**

To analyze **Packet Delivery Ratio (PDR)** and **Control Packets (DIO, DAO, DIS)**, we need to extract more data from the log file than we are currently doing.

Here is the roadmap:
1.  **The Data Gap:** What is missing from the current CSV.
2.  **The Solution:** A Python analysis script using `pandas` and `seaborn` to visualize the metrics (assuming we have the data).

---

### 1. The Data Gap

To plot the metrics in **8.3**, your CSV needs to look slightly different, or you need two CSVs.

*   **Current CSV:** Good for **Latency (8.3.3)**.
*   **Missing for PDR (8.3.2):** We need to know how many packets were sent *but never received*. Currently, we only log the matches.
*   **Missing for Control Packets (8.3.1):** We need to count lines in the log file containing `Sending DAO`, `Sending DIO`, and `Sending DIS`.

**Recommendation:**
I recommend updating your parser (I can help with this in the next step) to produce a second "Summary CSV" that looks like this:
`RunID, Network, DODAGS, NodeCount, Seed, Total_Sent, Total_Received, Count_DIO, Count_DAO, Count_DIS`

---

### 2. The Python Analysis Script

Assuming you have your detailed packet data (`simulation_data.csv`) and a summary of runs (`simulation_summary.csv`), here is the Python script to generate publication-ready graphs for your thesis.

You will need to install the libraries:
`pip install pandas seaborn matplotlib`

```python
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt

# ================= CONFIGURATION =================
# We assume two input files:
# 1. The file we created previously (Latency data)
PACKET_FILE = 'simulation_data_with_time.csv' 

# 2. A hypothetical summary file (I will help you generate this next)
# Columns: [Network, DODAGS, Timestamp, Node_Count, Seed, PDR, DIO_Count, DAO_Count]
SUMMARY_FILE = 'simulation_summary_metrics.csv' 
# =================================================

def set_style():
    """Sets a professional style for thesis plots"""
    sns.set_theme(style="whitegrid")
    sns.set_context("paper", font_scale=1.4)
    plt.rcParams['figure.figsize'] = (10, 6)

def plot_latency(df):
    """
    8.3.3 Packet Delivery Rate / Latency
    Visualizes the time taken for packets to traverse the network.
    """
    print("Generating Latency Boxplot...")
    plt.figure()
    
    # We filter out extreme outliers if necessary, but boxplots handle them well.
    
    # Boxplot: Shows Median, Quartiles, and Outliers
    sns.boxplot(
        data=df, 
        x='Network', 
        y='Time', 
        hue='DODAGS', 
        palette="viridis",
        showfliers=False # Turn to True to see outlier packets
    )
    
    plt.title('Packet Latency by Network Topology')
    plt.ylabel('Time to Root (Seconds)')
    plt.xlabel('Network Topology')
    plt.legend(title='DODAGs')
    plt.tight_layout()
    plt.savefig('8.3.3_Packet_Latency.png', dpi=300)
    plt.show()

def plot_pdr(df):
    """
    8.3.2 Packet Delivery Ratio
    Visualizes PDR % across different topologies.
    """
    print("Generating PDR Bar Chart...")
    plt.figure()
    
    # Bar chart with error bars (95% confidence interval by default)
    sns.barplot(
        data=df,
        x='Network',
        y='PDR',
        hue='DODAGS',
        palette="magma",
        capsize=.1
    )
    
    plt.title('Packet Delivery Ratio (PDR)')
    plt.ylabel('PDR (%)')
    plt.xlabel('Network Topology')
    plt.ylim(0, 105) # PDR is usually 0-100
    plt.legend(title='DODAGs', loc='lower right')
    plt.tight_layout()
    plt.savefig('8.3.2_PDR.png', dpi=300)
    plt.show()

def plot_control_overhead(df):
    """
    8.3.1 Number of RPL Control Packets
    Visualizes the total volume of DIO/DAO packets.
    """
    print("Generating Control Overhead Chart...")
    
    # Melt the data to make it "Long" format for Seaborn
    # From: [Network, DIO_Count, DAO_Count]
    # To:   [Network, Packet_Type, Count]
    df_melted = df.melt(
        id_vars=['Network', 'DODAGS'], 
        value_vars=['DIO_Count', 'DAO_Count', 'DIS_Count'],
        var_name='Packet_Type', 
        value_name='Count'
    )

    plt.figure()
    
    sns.barplot(
        data=df_melted,
        x='Network',
        y='Count',
        hue='Packet_Type',
        palette="Set2"
    )
    
    plt.title('RPL Control Packet Overhead')
    plt.ylabel('Average Packets per Simulation')
    plt.xlabel('Network Topology')
    plt.tight_layout()
    plt.savefig('8.3.1_Control_Packets.png', dpi=300)
    plt.show()

def analyze_data():
    set_style()
    
    # --- LOAD DATA ---
    try:
        # Load the packet-level data (for Latency)
        df_packets = pd.read_csv(PACKET_FILE)
        
        # Ensure DODAGS is categorical for proper coloring
        df_packets['DODAGS'] = df_packets['DODAGS'].astype(str)
        
        # Generate Latency Plot
        plot_latency(df_packets)

        # --- NOTE: The following requires the Summary CSV ---
        # For now, I will create DUMMY data so you can see the code work.
        # In reality, you will load: df_summary = pd.read_csv(SUMMARY_FILE)
        
        print("\n(Simulating Summary Data for PDR and Control Packets...)")
        data = {
            'Network': ['Circular']*10 + ['Random']*10 + ['Grid']*10,
            'DODAGS': (['1']*5 + ['2']*5) * 3,
            'PDR': [95, 96, 94, 95, 97, 88, 85, 89, 87, 88,  # Circular (D1 high, D2 low)
                    80, 82, 78, 81, 79, 70, 72, 68, 71, 69,  # Random
                    99, 98, 99, 98, 99, 95, 94, 96, 95, 95], # Grid
            'DIO_Count': [1200, 1150, 1210, 1180, 1190, 2400, 2350, 2410, 2380, 2390]*3,
            'DAO_Count': [300, 310, 305, 302, 308, 600, 610, 605, 602, 608]*3,
            'DIS_Count': [50, 52, 48, 51, 49, 10, 12, 11, 10, 13]*3
        }
        df_summary = pd.DataFrame(data)
        
        # Generate PDR and Overhead Plots
        plot_pdr(df_summary)
        plot_control_overhead(df_summary)

    except FileNotFoundError as e:
        print(f"Error: {e}")
        print("Please ensure your CSV files are in the same folder.")

if __name__ == "__main__":
    analyze_data()
```

### 3. How to Interpret the Visuals for your Thesis

1.  **Boxplots (Latency):**
    *   Look at the "Interquartile Range" (the colored box). A short box means consistent performance. A tall box means jitter (unpredictable latency).
    *   Compare D1 vs D2 (colors). If D2 is consistently higher, the complexity of 2 DODAGs is adding latency.

2.  **Bar Charts (PDR):**
    *   The black lines on top of the bars are "Error Bars" (Confidence Intervals).
    *   If the error bars of two columns overlap, the difference might not be statistically significant (you can mention this in your results).

3.  **Stacked/Grouped Bars (Control Overhead):**
    *   This is critical for battery life. High DIO counts = high radio usage = low battery.
    *   You expect D2 to have roughly double the overhead of D1. If it's *more* than double, that's an interesting finding (the "Thundering Herd" problem).

### Next Step
Would you like me to write the **Parser Script** that generates the `simulation_summary_metrics.csv` (counting the DIOs, DAOs, and calculating PDR from the log file)?
