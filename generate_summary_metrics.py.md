That is a good question.

**Here is the risk:** Your file is named `N16_Dx_Sent+Received_UPD.txt`. This implies it might have been filtered (using `grep` or similar) to *only* show application "Sent" and "Received" lines.

If that file **does not** contain lines that look like `Sending multicast DIO` or `Sending DAO`, then we cannot count the control overhead from this specific file.

**However**, I have written the script below to be "optimistic."
1.  It calculates **PDR** (Packet Delivery Ratio) perfectly using the data we already have.
2.  It *attempts* to count **DIO/DAO/DIS**. If your log file doesn't have those lines, those columns will just be 0, but the PDR part will still work.

### The Extended Parser Script

Save this as `generate_summary_metrics.py`.

```python
import csv
import re
import os
from collections import defaultdict

# ================= CONFIGURATION =================
LOG_FILENAME = 'N16_Dx_Sent+Received_UPD.txt'
SEED_FILENAME = '../CoojaRuns.txt'
OUTPUT_FILENAME = 'simulation_summary_metrics.csv'
# =================================================

def load_seed_map(seed_file_path):
    """Loads the Map of Timestamp -> Seed Value"""
    seed_map = {}
    timestamp_pattern = re.compile(r'Timestamp:\s*(\d{14})')
    seed_pattern = re.compile(r'with seed\s+(\d+)')
    
    try:
        with open(seed_file_path, 'r') as f:
            for line in f:
                ts_match = timestamp_pattern.search(line)
                if ts_match:
                    timestamp = ts_match.group(1)
                    seed_match = seed_pattern.search(line)
                    seed_val = seed_match.group(1) if seed_match else 'N/A'
                    seed_map[timestamp] = seed_val
    except FileNotFoundError:
        print(f"Warning: Seed file '{seed_file_path}' not found.")
    return seed_map

def parse_full_metrics():
    seed_map = load_seed_map(SEED_FILENAME)
    
    # Dictionary to hold aggregated stats per simulation run
    # Key: Simulation_Timestamp
    # Value: Dictionary of counters
    sim_stats = defaultdict(lambda: {
        'Network': 'Unknown',
        'DODAGS': '0',
        'NodeCount': '0',
        'App_Sent': 0,
        'App_Recv': 0,
        'DIO_Count': 0,
        'DAO_Count': 0,
        'DIS_Count': 0
    })

    # Regex to extract Simulation Metadata from Filename
    # Matches: N16_D1_Circular-20260226160016.txt
    filename_re = re.compile(r'N(\d+)_D(\d+)_([A-Za-z]+)-(\d{14})')

    # Regex for Application Payload (to count Sends)
    # We look for 'Inst X Msg Y' to ensure it's a data packet
    payload_re = re.compile(r'\'Inst\s+(\d+)\s+Msg\s+(\d+)\'')

    print(f"--- Scanning {LOG_FILENAME} for metrics ---")

    try:
        with open(LOG_FILENAME, 'r') as f:
            for line in f:
                # 1. Parse Metadata from Filename inside the log line
                # We need this to group stats by the specific Simulation Run
                parts = line.split(" Node:")
                if len(parts) < 2: continue
                
                metadata_part = parts[0]
                message_part = parts[1]

                # Extract filename
                filepath_end_index = metadata_part.find('.txt')
                if filepath_end_index == -1: continue
                
                full_path_chunk = metadata_part[:filepath_end_index+4]
                filename = os.path.basename(full_path_chunk)
                
                file_match = filename_re.search(filename)
                if not file_match: continue
                
                # Extract Keys
                node_count = file_match.group(1)
                dodags = file_match.group(2)
                network_type = file_match.group(3)
                sim_timestamp = file_match.group(4)

                # Initialize this run's static data if not set
                if sim_stats[sim_timestamp]['Network'] == 'Unknown':
                    sim_stats[sim_timestamp]['Network'] = network_type
                    sim_stats[sim_timestamp]['DODAGS'] = dodags
                    sim_stats[sim_timestamp]['NodeCount'] = node_count

                # 2. Analyze the Message Content
                
                # --- A. Count Application Packets (Sent vs Received) ---
                if payload_re.search(message_part):
                    if "Sending to Root" in message_part:
                        sim_stats[sim_timestamp]['App_Sent'] += 1
                    elif "Data received from" in message_part:
                        sim_stats[sim_timestamp]['App_Recv'] += 1

                # --- B. Count RPL Control Packets ---
                # This assumes your log file contains lines like "Sending DIO", "Sending DAO"
                # If your file is filtered to only UDP app packets, these will remain 0.
                else:
                    # We check specifically for RPL control messages
                    # Note: Contiki-NG logs usually say "Sending [multicast] DIO"
                    if "Sending" in message_part:
                        if "DIO" in message_part:
                            sim_stats[sim_timestamp]['DIO_Count'] += 1
                        elif "DAO" in message_part:
                            sim_stats[sim_timestamp]['DAO_Count'] += 1
                        elif "DIS" in message_part:
                            sim_stats[sim_timestamp]['DIS_Count'] += 1

        # 3. Calculate PDR and Write to CSV
        header = ['Network', 'DODAGS', 'NodeCount', 'Seed', 'Timestamp', 
                  'App_Sent', 'App_Recv', 'PDR', 'DIO_Count', 'DAO_Count', 'DIS_Count']
        
        output_rows = []
        
        for timestamp, stats in sim_stats.items():
            sent = stats['App_Sent']
            recv = stats['App_Recv']
            
            # Calculate Packet Delivery Ratio (Percentage)
            if sent > 0:
                pdr = round((recv / sent) * 100, 2)
            else:
                pdr = 0.0
            
            # Get Seed
            seed = seed_map.get(timestamp, 'N/A')
            
            output_rows.append([
                stats['Network'],
                stats['DODAGS'],
                stats['NodeCount'],
                seed,
                timestamp,
                sent,
                recv,
                pdr,
                stats['DIO_Count'],
                stats['DAO_Count'],
                stats['DIS_Count']
            ])

        # Write to file
        with open(OUTPUT_FILENAME, 'w', newline='') as csvfile:
            writer = csv.writer(csvfile)
            writer.writerow(header)
            writer.writerows(output_rows)

        print(f"Successfully aggregated {len(output_rows)} simulation runs.")
        print(f"Data written to {OUTPUT_FILENAME}")
        
        # Quick Debug check
        total_dio = sum(r[8] for r in output_rows)
        if total_dio == 0:
            print("\n[NOTE] Total DIO count is 0.")
            print("If you expected control packets, check if your input text file contains")
            print("lines with 'Sending DIO' or 'Sending DAO'. If it was filtered for UDP only,")
            print("you will need to re-generate the log file to include RPL messages.")

    except Exception as e:
        print(f"An error occurred: {e}")

if __name__ == "__main__":
    parse_full_metrics()
```

### What this script does differently

1.  **Aggregates per Run:** Instead of writing a line for every packet, it sums up everything for one specific Simulation Run (e.g., Circular, D1, Seed 5).
2.  **Calculates PDR:** It takes `(Total Received / Total Sent) * 100` for that specific run and gives you a percentage (e.g., 98.5%).
3.  **Counts Control Packets:** It looks for lines containing "Sending" + "DIO/DAO/DIS".
4.  **Extracts Node Count:** It pulls the "16" from `N16...` so you can sort your future graphs by network size (50, 100, 200).

### Instructions

1.  Run this script.
2.  Look at the console output.
    *   If it says `Total DIO count is 0`, your current text file was filtered too aggressively (only kept App packets). You can still analyze Latency and PDR, but you won't be able to do Section 8.3.1 (Control Packets) without the full logs.
    *   If it shows numbers > 0 for DIO/DAO, you are golden!
