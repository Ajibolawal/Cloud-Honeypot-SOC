# Home SOC in the Cloud using Azure ☁️🔐

## Introduction 📖
In this project, I will be setting up a **Home Security Operations Centre (SOC) in the Cloud** using **Azure**. The main idea is to deploy a **honeypot** to attract and log cyberattacks, forward the logs to a **central repository**, and analyse them using **Microsoft Sentinel**.

---

## What is a Honeypot? 🍯🐝
A **honeypot** is a **decoy system** designed to mimic a real vulnerable target. It is deliberately left exposed to **entice attackers**, allowing security professionals to monitor, analyse, and learn from malicious activities.

---

## Project Architecture 🏗️
The following network diagram outlines the setup:

```plaintext
+----------------------------------+
|         Azure Subscription       |
| +------------------------------+ |
| |       Resource Group         | |
| |  +------------------------+  | |
| |  |         VNet          |  | |
| |  |  +------------------+ |  | |
| |  |  | Virtual Machine  | |  | |
| |  |  +------------------+ |  | |
| |  |      (Honeypot)       |  | |
| |  +------------------------+  | |
| | NSG: Open to Public Internet | |
| | Logs → Log Analytics → SIEM  | |
| +------------------------------+ |
+----------------------------------+
```

---

# Step-by-Step Guide 🛠️

## 1. **Creating a Resource Group** 📁
- Head over to the **Azure Portal** and click on **Resource Groups**.
- Pick a **Region** that's closest to you.
- Give it a clear name, like `RG-SOC-LAB`, so you know what it's for.

## 2. **Creating a Virtual Network** 🌐
- Navigate to the **Azure Portal** and find **Virtual Networks**.
- Assign it to the **Resource Group** you just created.
- The **IP address range** will be set automatically.
- A **subnet** will also be created for you.

## 3. **Deploying the Virtual Machine** 🖥️
1. In the **Azure Portal**, go to **Virtual Machines**.
2. Select the same **Resource Group** you created earlier.
3. Give the VM a name, something like `VM-SOC-LAB`.
4. Choose **Windows 10** as the operating system image.
5. Pick an appropriate **size** for your VM.
6. Create an **Administrator username and password**.
7. Tick the box to confirm you're complying with the **licensing terms**.
8. In the **Networking tab**:
   - Select the **VNet** you created earlier.
   - Make sure you check the box to **delete the public IP and NIC when the VM is deleted**.
9. In the **Monitoring tab**, select **Disable** for **Boot Diagnostics**.
10. Finally, click **Deploy** to create the VM.

## 4. **Log in to the VM and Disable the Windows Firewall** 🛑
1. Once the VM is set up, connect to it using **RDP (Remote Desktop Protocol)**.
2. After you're logged in, follow these steps to disable the firewall:
   - Press **Start** and type `wf.msc` to open **Windows Defender Firewall with Advanced Security**.
   - In the firewall window, click **Properties** on the right.
   - For each profile tab (Domain Profile, Private Profile, and Public Profile):
     - Set the **Firewall State** to **Off**.
3. To check if the firewall is off, **ping the VM** from your local machine:
   - Open **PowerShell** or **Command Prompt** (cmd).
   - Type this command:
   ```bash
   ping <VM_Public_IP>
   ```
   - Replace `<VM_Public_IP>` with the actual public IP address of your VM. If the VM responds, it confirms the firewall is disabled and that the machine is accessible on the network (which also makes it more vulnerable to attack).

## 5. **Sign Out and Test Login Attempts** 🚪
1. Sign out of the VM.
2. Try logging in with a different username and use **incorrect credentials** for 4 attempts. This simulates failed login attempts.
3. After 4 failed attempts, log in again using the correct username and password.

## 6. **Viewing Security Logs in Event Viewer** 📜
1. Open **Event Viewer** by pressing **Start** and typing `event viewer`.
2. This tool logs all system activity, including **security events**.
3. Go to **Windows Logs → Security** to view all **security-related logs** on the system.
4. These logs will provide information about **successful and failed login attempts** as well as other security-related events.

## 7. **Forwarding Logs to Azure for Analysis** 📡
To forward your security logs to Azure, follow these steps:

### 7.1. **Create a Log Analytics Workspace in Azure**
1. Go to the **Azure Portal**.
2. Create a new **Log Analytics Workspace**:
   - Assign it to the same **Resource Group** as your VM.
   - Name it something clear, like `LAW-SOC-LAB`.

### 7.2. **Create a Sentinel Instance in Azure**
1. Set up a **Microsoft Sentinel** instance in Azure.
2. Add the **Log Analytics Workspace** (`LAW-SOC-LAB`) to this instance.
3. This will link your Log Analytics Workspace to **Microsoft Sentinel**, allowing you to access and analyse the logs through the SIEM.

### 7.3. **Configuring the Azure Monitoring Agent (AMA) for the Security Event Connector**
1. In Azure Sentinel, go to **Content Management → Content Hub**.
2. Search for **Windows Security Event**, select it, and click **Install**.

### 7.4. **Configure the Windows Security Event Connector**
1. After installation, go to **Manage → Windows Security Events via AMA**.
2. Click **Open Connector Page** and then click **Create a Data Collection Rule**.
3. Name the rule something like `DCR-SOC-LAB` and assign it to the same **Resource Group**.
4. Under the **Resources** tab, select the **Virtual Machine** you created earlier.
5. Under the **Collect** tab, select **All Security Events**.
6. Click **Save** to finish the setup.

## 8. **Verifying the Log Forwarding and Querying Logs** 📊
1. Return to the **Log Analytics Workspace** and select **Logs**.
2. In the query box, type `SecurityEvents` and run the query.
3. The query will return the **Windows security events**, showing all activities logged from the VM.

You can now monitor and analyse these logs in **Microsoft Sentinel**.

## 9. **Using KQL to Query SecurityEvent Data** 🔍🔐
Now, you can use KQL (Kusto Query Language) to run queries on the **SecurityEvent** data. Below is a breakdown of how to query the security logs and filter the data as per the required conditions.

### **Collating Data from the SecurityEvent Table:**

```kusto
SecurityEvent
```
**Explanation:**
This line collates all the data from the SecurityEvent table into a single query. It retrieves all available logs related to security events stored within Microsoft Sentinel.

### **Filtering by Account and Projecting Relevant Columns:**

```kusto
SecurityEvent
| where Account == "\\ADMINUSER"
| project TimeGenerated, Account, Computer, EventID, Activity, IpAddress
```
**Explanation:**
- `| where Account == "\\ADMINUSER"`: Filters the logs to include only those where the Account is \\ADMINUSER. This is typically used to focus on security events related to specific user accounts (in this case, an administrative account).
- `| project TimeGenerated, Account, Computer, EventID, Activity, IpAddress`: Specifies the columns you want to include in the query results. In this case, the query will return the TimeGenerated, Account, Computer, EventID, Activity, and IpAddress for the filtered logs.
- This query is useful for tracking specific login events, particularly administrative account access.

### **Identifying Failed Login Attempts:**

```kusto
SecurityEvent
| where EventID == 4625
| project TimeGenerated, Account, Computer, EventID, Activity, IpAddress
```
**Explanation:**
- `| where EventID == 4625`: Filters the logs to only include events with EventID 4625, which corresponds to failed login attempts.
- `| project TimeGenerated, Account, Computer, EventID, Activity, IpAddress`: Specifies the fields to be displayed, showing the timestamp, account, computer, event ID, activity, and IP address for the failed login attempts.
- This query is used to identify and analyse failed login attempts across the environment.

### **Filtering Failed Login Attempts in the Last 5 Minutes:**
```kusto
SecurityEvent
| where EventID == 4625
| where TimeGenerated > ago(5m)
| project TimeGenerated, Account, Computer, EventID, Activity, IpAddress
```
**Explanation:**
- `| where EventID == 4625`: Filters to include only EventID 4625, indicating failed login attempts.
- `| where TimeGenerated > ago(5m)`: Filters to show only the events that occurred in the past 5 minutes. The ago(5m) function dynamically gets the logs from the last 5 minutes relative to the current time.
- `| project TimeGenerated, Account, Computer, EventID, Activity, IpAddress`: Specifies the columns to display, including time, account, computer, event ID, activity, and IP address.
- This query is particularly useful for real-time monitoring of failed login attempts.


## 10. **Geo-location Attack Map in Sentinel** 🌍

1. To create the geo-location attack map, go into the Sentinel instance, then:
    - Go to **Configuration** → **Watchlist** → **Click New**.

2. In the **Name** field, enter **geoip**.
3. In the **Alias** field, enter **geoip**.
4. In the **Source** tab, upload the file that contains the allowed IP locations from [this link](https://drive.google.com/file/d/13EfjM_4BohrmaxqXZLB5VUBIz2sv9Siz/view). (Make sure to save it locally first).
5. For the **Search Key**, select **network**.
6. Wait for it to upload.
7. Once uploaded, go back to the query and type the following:

    ```kql
    SecurityEvent
    _GetWatchlist("geoip")
    ```

    **Explanation**:
    - This is used to look up IP address locations to see where the failed login attempts are coming from.

---

## 11. **Querying the Geo-location Data with KQL** 🔍

1. Now, create a KQL query to investigate events associated with the IP addresses.

    ```kql
    let GeoIPDB_FULL = _GetWatchlist("geoip");
    let WindowsEvents = SecurityEvent
        | where IpAddress == "103.165.213.50"
        | where EventID == 4625
        | order by TimeGenerated desc
        | evaluate ipv4_lookup(GeoIPDB_FULL, IpAddress, network);
    WindowsEvents
    ```

    **Explanation**:
    - `let GeoIPDB_FULL = _GetWatchlist("geoip");`: Retrieves the **geoip** watchlist, which contains the allowed IP locations.
    - `let WindowsEvents = SecurityEvent ...`: This part defines the query to retrieve the security events related to the IP address `"103.165.213.50"`.
    - `| where IpAddress == "103.165.213.50"`: Filters the events where the IP address matches `"103.165.213.50"`.
    - `| where EventID == 4625`: Filters the events for **failed login attempts**.
    - `| order by TimeGenerated desc`: Orders the results by the most recent event first.
    - `| evaluate ipv4_lookup(GeoIPDB_FULL, IpAddress, network);`: Looks up the location of the IP address from the geo-location database.
    - `WindowsEvents`: Displays the result of the query.

---

2. To further expand on this, add the following query to display results:

    ```kql
    WindowsEvents
    |project TimeGenerated, Computer, AttackerIp = IpAddress, cityname, countryname, latitude, longitude
    ```

    **Explanation**:
    - `| project ...`: Projects (or selects) specific columns for the output: **TimeGenerated**, **Computer**, **AttackerIp**, **cityname**, **countryname**, **latitude**, and **longitude**.

---

## 12. **Visualising Attack Data on a Map** 📍

1. To visualise the data, go to **Sentinel**, then select **Threat Management** → **Workbooks** → **Add New Workbook**.
2. Click **Edit** and remove the pre-populated data.
3. Click on **Add New Query** → **Advanced Editor**, then clear the existing text and paste the following JSON:

    ```json
    {
        "type": 3,
        "content": {
            "version": "KqlItem/1.0",
            "query": "let GeoIPDB_FULL = _GetWatchlist(\"geoip\");\nlet WindowsEvents = SecurityEvent;\nWindowsEvents | where EventID == 4625\n| order by TimeGenerated desc\n| evaluate ipv4_lookup(GeoIPDB_FULL, IpAddress, network)\n| summarize FailureCount = count() by IpAddress, latitude, longitude, cityname, countryname\n| project FailureCount, AttackerIp = IpAddress, latitude, longitude, city = cityname, country = countryname,\nfriendly_location = strcat(cityname, \" (\", countryname, \")\");",
            "size": 3,
            "timeContext": {
                "durationMs": 2592000000
            },
            "queryType": 0,
            "resourceType": "microsoft.operationalinsights/workspaces",
            "visualization": "map",
            "mapSettings": {
                "locInfo": "LatLong",
                "locInfoColumn": "countryname",
                "latitude": "latitude",
                "longitude": "longitude",
                "sizeSettings": "FailureCount",
                "sizeAggregation": "Sum",
                "opacity": 0.8,
                "labelSettings": "friendly_location",
                "legendMetric": "FailureCount",
                "legendAggregation": "Sum",
                "itemColorSettings": {
                    "nodeColorField": "FailureCount",
                    "colorAggregation": "Sum",
                    "type": "heatmap",
                    "heatmapPalette": "greenRed"
                }
            }
        },
        "name": "query - 0"
    }
    ```

    **Explanation of the JSON**:
    - The **query** in the JSON is similar to the KQL query written earlier, but it now includes an additional step to **summarize** the failure count by IP address, city, country, and location.
    - **visualization**: Specifies that the output should be a **map**.
    - **mapSettings**: Customizes how the map is displayed, including what data to show and how to visualise the attack locations.

4. Click **Done Editing** to view the map.
5. Save the workbook as **Windows VM Attack Map** and select the **UK South** resource group and location.

---

## Tools Used 🛠️
- **Azure Virtual Machines** (Honeypot deployment)
- **Azure Network Security Group (NSG)** (Intentional exposure)
- **Log Analytics Workspace** (Centralised log collection)
- **Microsoft Sentinel** (SIEM for log analysis)
- **Kusto Query Language (KQL)** (Threat hunting and analysis)

---

## Skills Gained 🚀
✅ Deploying and managing **Azure resources**
✅ Setting up **honeypots** and monitoring real-world attacks
✅ Configuring **log forwarding** and analysis
✅ Using **Microsoft Sentinel for SIEM**
✅ Writing **KQL queries** to detect attack patterns
✅ Creating **visual dashboards** for cybersecurity analytics

---

## Conclusion 🎯
This **Home SOC project** provides hands-on experience with cloud security, **threat intelligence**, and SIEM operations. By running a honeypot, I can analyse real-world threats and enhance my cybersecurity skills.

---

## Future Enhancements 🔮
- Automate alerts using **Azure Logic Apps**
- Integrate **Threat Intelligence Feeds**
- Implement **Machine Learning-based anomaly detection**
- Add more honeypots for broader analysis
- Improve dashboards with **real-time visualisations**

---

