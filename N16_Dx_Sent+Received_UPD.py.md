You are absolutely correct to question the matching logic.

### The Problem
The issue isn't the Timestamp in the key. The issue is in the **Regex for the Payload**.

1.  **Send lines** look like this: `... Payload: 'Inst 30 Msg 0'`
2.  **Receive lines** look like this: `... length 14: 'Inst 30 Msg 0'`

My previous script looked for `Payload: 'Inst...`. Because the **Receive** lines do not contain the word "Payload", the script was skipping them entirely before it even checked if it was a "Data received" event.

### The Fix
1.  **Relax the Regex:** match `'Inst X Msg Y'` directly, ignoring the word "Payload".
2.  **Keep the Timestamp in the Key:** You actually **do** want the `filename_timestamp` in the key. This timestamp represents the **Simulation Run ID** (e.g., Run 1 vs Run 50). If you remove it, a packet sent in Simulation 1 could accidentally match with a receive event in Simulation 2 if the Node and Message numbers happen to be the same.

Here is the corrected script.

```python
import csv
import re
import os

# ================= CONFIGURATION =================
LOG_FILENAME = 'N16_Dx_Sent+Received_UPD.txt'
SEED_FILENAME = '../CoojaRuns.txt'
OUTPUT_FILENAME = 'simulation_data_with_time.csv'
# =================================================

def parse_time_to_seconds(time_str):
    try:
        minutes, seconds = time_str.split(':')
        return (float(minutes) * 60) + float(seconds)
    except ValueError:
        return 0.0

def load_seed_map(seed_file_path):
    seed_map = {}
    timestamp_pattern = re.compile(r'Timestamp:\s*(\d{14})')
    seed_pattern = re.compile(r'with seed\s+(\d+)')
    
    print(f"--- Loading seeds from {seed_file_path} ---")
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
        print(f"Error: Seed file '{seed_file_path}' not found.")
    return seed_map

def process_log():
    seed_dictionary = load_seed_map(SEED_FILENAME)
    header = ['Network','DODAGS','Timestamp','Node','Seed','Instance','Message','Time']
    
    extracted_rows = []
    
    # Key: (Simulation_Timestamp, Sender_Node_ID, Instance, Message_Num)
    # Value: Send_Time_Seconds
    sent_packets = {}
    
    # Counters
    count_sends = 0
    count_recvs = 0
    count_matched = 0

    # 1. Filename Regex
    filename_re = re.compile(r'N\d+_D(\d+)_([A-Za-z]+)-(\d{14})')
    
    # 2. Payload Regex - CHANGED
    # Removed "Payload:" prefix requirement. matches 'Inst 30 Msg 0' anywhere.
    payload_re = re.compile(r'\'Inst\s+(\d+)\s+Msg\s+(\d+)\'')
    
    # 3. Source IP Regex
    # Matches the last hex block before "on port"
    rx_source_re = re.compile(r'Data received from .*?([a-fA-F0-9]+)\s+on port')

    print(f"--- Processing {LOG_FILENAME} ---")

    try:
        with open(LOG_FILENAME, 'r') as f:
            for line in f:
                
                # Split line to separate Metadata from Message
                parts = line.split(" Node:")
                if len(parts) < 2: continue
                
                metadata_part = parts[0]
                message_part = parts[1] # e.g., "6 :[INFO..."
                
                # --- A. Parse Timestamp (Time of event) ---
                time_match = re.search(r'(\d{2}:\d{2}\.\d{3})$', metadata_part.strip())
                if not time_match: continue
                current_time_sec = parse_time_to_seconds(time_match.group(1))

                # --- B. Parse Filename (Simulation ID) ---
                filepath_end_index = metadata_part.find('.txt')
                if filepath_end_index == -1: continue
                
                full_path_chunk = metadata_part[:filepath_end_index+4]
                filename = os.path.basename(full_path_chunk)
                
                file_match = filename_re.search(filename)
                if not file_match: continue
                
                dodags = file_match.group(1)
                network_type = file_match.group(2)
                sim_timestamp = file_match.group(3) # Unique ID for this simulation run

                # --- C. Parse Payload (Instance / Msg Num) ---
                payload_match = payload_re.search(message_part)
                if not payload_match: 
                    # If there is no 'Inst X Msg Y', we don't care about this line
                    continue
                
                instance = payload_match.group(1)
                msg_num = payload_match.group(2)
                
                # --- D. Logic: Send vs Receive ---
                
                # CASE 1: SENDING
                if "Sending to Root" in message_part:
                    # The node logging this message IS the sender
                    log_node_id = message_part.split()[0].strip().replace(':', '')
                    
                    # Key includes sim_timestamp to ensure we don't mix up runs
                    key = (sim_timestamp, log_node_id, instance, msg_num)
                    sent_packets[key] = current_time_sec
                    count_sends += 1
                    
                # CASE 2: RECEIVING
                elif "Data received from" in message_part:
                    count_recvs += 1
                    
                    # Identify who sent this packet by parsing the IPv6 address
                    src_match = rx_source_re.search(message_part)
                    if src_match:
                        hex_id = src_match.group(1)
                        sender_node = str(int(hex_id, 16)) # Hex to Int (b -> 11)
                        
                        key = (sim_timestamp, sender_node, instance, msg_num)
                        
                        # Look for the matching SENT packet
                        if key in sent_packets:
                            send_time = sent_packets[key]
                            time_taken = round(current_time_sec - send_time, 3)
                            
                            seed_val = seed_dictionary.get(sim_timestamp, "N/A")
                            
                            row = [network_type, dodags, sim_timestamp, sender_node, seed_val, instance, msg_num, time_taken]
                            extracted_rows.append(row)
                            count_matched += 1
                            
                            # Clean up dictionary to keep it small (optional)
                            # del sent_packets[key] 

        # Write CSV
        with open(OUTPUT_FILENAME, 'w', newline='') as csvfile:
            writer = csv.writer(csvfile)
            writer.writerow(header)
            writer.writerows(extracted_rows)

        print(f"\n--- Results ---")
        print(f"Sends Found:    {count_sends}")
        print(f"Recvs Found:    {count_recvs}")
        print(f"Matches Made:   {count_matched}")
        print(f"File Written:   {OUTPUT_FILENAME}")

    except Exception as e:
        print(f"An error occurred: {e}")

if __name__ == "__main__":
    process_log()
```
