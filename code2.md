import re
import base64
import subprocess
from datetime import datetime
from openpyxl import Workbook

def decode_base64(s):
    """Decode a string using base64."""
    return base64.b64decode(s.encode('utf-8')).decode('utf-8')

def read_credentials(filename):
    """Read credentials from a file."""
    credentials = []
    with open(filename, 'r') as file:
        block = {}
        for line in file:
            line = line.strip()
            if line:
                key, value = line.split('=', 1)
                block[key] = value.strip('\"')
            elif block:
                credentials.append(block)
                block = {}
        if block: # to handle no newline at the end
            credentials.append(block)
    return credentials

all_creds = read_credentials('/root/automation/cred.txt')

# Generate timestamp for the report filename
timestamp = datetime.now().strftime("%Y-%m-%d_%H-%M-%S")
excel_filename = f'status-{timestamp}.xlsx'
wfilename = f'Healthcheck-{timestamp}.txt'

# Create a new workbook
print("Creating Excel workbook...")
wb = Workbook()
ws = wb.active

# Writing headers
ws.append(["IP", "Disk Utilization Status", "Available Memory Status", "Uptime Status", "Chrony Status", "Node Status", "Pod Status"])

print("\t\t*****HealthCheck Report*****")
with open(wfilename, 'w') as wfile:
    for creds in all_creds:
        IP = creds['IP']
        USERNAME = creds['USERNAME']
        PASSWORD = decode_base64(creds.get('PASSWORD', ''))
        K8s_Master = creds['K8s_Master']
        
        print(f"Performing health check for {IP}...")
        
        disk_status = "Pass"
        memory_status = "Pass"
        uptime_status = "Pass"
        leap_status = "Pass"

        # Disk utilization check
        disk_cmd = f'sshpass -p "{PASSWORD}" ssh -o StrictHostKeyChecking=no {USERNAME}@{IP} "df -h"'
        disk_util = subprocess.check_output(disk_cmd, shell=True, stderr=subprocess.STDOUT).decode('utf-8')
        pattern = r'^\S+\s+\S+\s+\S+\s+\S+\s+(\d+)%\s+\S+$' # Updated pattern to match the "Use%" value

        # Iterate over each line of the output
        for line in disk_util.split('\n'):
            match = re.search(pattern, line)
            if match:
                utilization = int(match.group(1))
                if utilization >= 90:
                    disk_status = "Fail"
                    break # If any line meets the condition, we can break out of the loop

        # Memory check
        mem_cmd = f'sshpass -p "{PASSWORD}" ssh -o StrictHostKeyChecking=no {USERNAME}@{IP} "free -g"'
        mem_info = subprocess.check_output(mem_cmd, shell=True, stderr=subprocess.STDOUT).decode('utf-8')
        pattern = r'^Mem:\s+(\d+)\s+\d+\s+\d+\s+\d+\s+\d+\s+\d+$' # Match total memory
        match = re.search(pattern, mem_info, re.MULTILINE)
        if match:
            total_memory = int(match.group(1))

        pattern = r'^Mem:\s+\d+\s+\d+\s+\d+\s+\d+\s+\d+\s+(\d+)$' # Match available memory
        match = re.search(pattern, mem_info, re.MULTILINE)
        if match:
            available_memory = int(match.group(1))
            if available_memory < 0.5 * total_memory:
                memory_status = "Fail"

        # Uptime check
        uptime_cmd = f'sshpass -p "{PASSWORD}" ssh -o StrictHostKeyChecking=no {USERNAME}@{IP} "uptime"'
        uptime_info = subprocess.check_output(uptime_cmd, shell=True, stderr=subprocess.STDOUT).decode('utf-8')
        pattern = r'up\s+(\d+)\s+day'
        uptime_match = re.search(pattern, uptime_info)
        if uptime_match:
            days = int(uptime_match.group(1))
            if days <= 5:
                uptime_status = "Fail"

        #NTP configuration
        chrony_cmd = f'sshpass -p "{PASSWORD}" ssh -o StrictHostKeyChecking=no {USERNAME}@{IP} "chronyc tracking"'
        chrony_info = subprocess.check_output(chrony_cmd, shell=True, stderr=subprocess.STDOUT).decode('utf-8')
        leap_pattern = r'Leap status\s+:\s+(\w+)'
        leap_match = re.search(leap_pattern, chrony_info)
        if leap_match:
            leap_status = "Fail" if leap_match.group(1) != "Normal" else "Pass" 
    
        # CPU Info
        cpu_cmd = f'sshpass -p "{PASSWORD}" ssh -o StrictHostKeyChecking=no {USERNAME}@{IP} "lscpu"'
        cpu_info = subprocess.check_output(cpu_cmd, shell=True, stderr=subprocess.STDOUT).decode('utf-8')

        # Print results
        wfile.write("\t\tHealthcheck status of " + IP + "\n")
        wfile.write("Disk utilization:\n" + disk_util + "\n")
        wfile.write("CPU info:\n" + cpu_info + "\n")
        wfile.write("Memory info:\n" + mem_info + "\n")
        wfile.write("Uptime info:\n" + uptime_info + "\n")
        wfile.write("Chrony Status:\n" + chrony_info + "\n")
        #ws.append([IP, disk_status, memory_status, uptime_status])
#################### K8s Check ################################

# Execute additional commands for Kubernetes master
        node_status = "NA"
        pod_status = "NA"
        component_status = "NA"
        if K8s_Master.lower() == 'yes':
            node_status = "Pass"
            pod_status = "Pass"

            kubectl_get_nodes_cmd = f'sshpass -p "{PASSWORD}" ssh -o StrictHostKeyChecking=no {USERNAME}@{IP} "kubectl get nodes"'
            kubectl_get_nodes_output = subprocess.check_output(kubectl_get_nodes_cmd, shell=True, stderr=subprocess.STDOUT).decode('utf-8')
            # Define pattern to match node status
            pattern = re.compile(r'^\S+\s+(\S+)\s+\S+\s+\S+\s+\S+')
            # Search for node status in the output
            for line in kubectl_get_nodes_output.split('\n')[1:]:
                if line.strip():
                    match = pattern.match(line)
                    if match:
                        x = match.group(1)
                        if x != "Ready":
                            node_status = "Fail"
                            break
            # Write node status to the file
            wfile.write("Nodes:\n" + kubectl_get_nodes_output + "\n")

            kubectl_get_pod_cmd = f'sshpass -p "{PASSWORD}" ssh -o StrictHostKeyChecking=no {USERNAME}@{IP} "kubectl get pod -n kube-system"'
            kubectl_get_pod_output = subprocess.check_output(kubectl_get_pod_cmd, shell=True, stderr=subprocess.STDOUT).decode('utf-8')
            pod_status_pattern = re.compile(r'^\S+\s+\S+\s+(\S+)\s+\S+\s+\S+\s+\S+')
            # Iterate over each line of the output
            for line in kubectl_get_pod_output.split('\n'):
                match = pod_status_pattern.match(line)
                if match:
                    pod_status_value = match.group(1)
                    if pod_status_value != "Running":
                        pod_status = "Fail"
                        break
    
            wfile.write("Pods in all namespaces:\n" + kubectl_get_pod_output + "\n")
            kubectl_get_pvc_cmd = f'sshpass -p "{PASSWORD}" ssh -o StrictHostKeyChecking=no {USERNAME}@{IP} "kubectl get pvc --all-namespaces"'
            kubectl_get_pvc_output = subprocess.check_output(kubectl_get_pvc_cmd, shell=True, stderr=subprocess.STDOUT).decode('utf-8')
            wfile.write("PVCs in all namespaces:\n" + kubectl_get_pvc_output + "\n")

            # Execute the command to get component statuses
            kubectl_get_componentstatuses_cmd = f'sshpass -p "{PASSWORD}" ssh -o StrictHostKeyChecking=no {USERNAME}@{IP} "kubectl get componentstatuses"'
            kubectl_get_componentstatuses_output = subprocess.check_output(kubectl_get_componentstatuses_cmd, shell=True, stderr=subprocess.STDOUT).decode('utf-8')
            wfile.write("Component Statuses:\n" + kubectl_get_componentstatuses_output + "\n")

            wfile.write("##########################################################################################\n")
    # Write data to Excel
        ws.append([IP, disk_status, memory_status, uptime_status, leap_status, node_status, pod_status])

wb.save(excel_filename)
print("Excel workbook saved successfully.")
print("Health check report has been saved in", wfilename, "file.")

