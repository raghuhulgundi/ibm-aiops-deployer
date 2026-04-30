# RobotShop NYC to Orlando Network Scenario

## Overview
This scenario simulates a network saturation issue where the RobotShop application's Catalogue and Ratings services (deployed in NYC) are accessing a MySQL database in Orlando through a complex network infrastructure including fiber optic connections, routers, and switches.

## Architecture

### Network Topology
```
NYC Data Center
├── Web Service (K8s Cluster)
├── Catalogue Service (Separate Network - VM)
│   └── Connected via NYC-SWITCH-001
└── Ratings Service (Separate Network - VM)
    └── Connected via NYC-SWITCH-001
    
NYC-ROUTER-001
    └── NYC-FIBER-DEVICE (Fiber Optic)
        └── NYC to Orlando Fiber Link
            └── ORL-FIBER-DEVICE (Fiber Optic)
                └── ORL-ROUTER-001
                    └── ORL-SWITCH-001 (CRITICAL POINT)
                        └── ORL-SQL-SWITCH-PORT (SATURATION POINT)
                            └── ORL-SQL-VM
                                └── MySQL Database
```

### Key Components

#### NYC Infrastructure
- **NYC-K8S-CLUSTER**: Kubernetes cluster hosting web service
- **NYC-CATALOGUE-VM**: VM hosting catalogue service (10.10.1.50)
- **NYC-RATINGS-VM**: VM hosting ratings service (10.10.1.51)
- **NYC-SWITCH-001**: Network switch connecting VMs
- **NYC-ROUTER-001**: Core router connecting to fiber network
- **NYC-FIBER-DEVICE**: Fiber optic device for long-distance connectivity

#### Fiber Optic Network
- **NYC-FIBER-CONNECTION**: NYC to Orlando fiber link
- **ORL-FIBER-CONNECTION**: Orlando fiber termination
- Provides 10Gbps bandwidth between locations

#### Orlando Infrastructure
- **ORL-FIBER-DEVICE**: Fiber optic device receiving from NYC
- **ORL-ROUTER-001**: Core router in Orlando
- **ORL-SWITCH-001**: Critical switch connecting to database (BOTTLENECK)
- **ORL-SQL-SWITCH-PORT**: 1Gbps port connecting to SQL VM (SATURATION POINT)
- **ORL-SQL-VM**: VM hosting MySQL database (10.20.1.100)
- **mysql-db**: MySQL database serving ratings and catalogue data

## Problem Scenario

### Root Cause
The switch port (ORL-SQL-SWITCH-PORT) connecting to the SQL database VM is saturated at 95-98% utilization. This 1Gbps port is the bottleneck in an otherwise high-capacity network.

### Symptom Chain
1. **Network Layer**: Switch port saturation (95-98% utilization)
2. **Infrastructure Layer**: High latency on Orlando switch (250ms vs normal 5ms)
3. **Database Layer**: Connection timeouts and slow queries
4. **Application Layer**: 
   - Ratings service database connection errors
   - Catalogue service slow response times (5000ms vs normal 200ms)
5. **User Experience**: Web service degradation (8000ms page load vs normal 1500ms)

### Alert Sequence
The alerts are designed to show the cascading failure:
1. Switch port saturation alerts (Critical - Severity 6)
2. Network performance degradation (High - Severity 5)
3. Database connection timeouts (Critical - Severity 6)
4. VM performance issues (High - Severity 5)
5. Application errors (Critical - Severity 6)
6. User experience degradation (Medium - Severity 4)

## Files

### 1. Topology File
**Location**: `ansible/roles/ibm-aiops-demo-content/templates/topology/robotshop-nyc-orlando-network.txt`

**Purpose**: Defines the complete network topology from NYC to Orlando including:
- Application services (web, catalogue, ratings)
- VMs and bare metal servers
- Network devices (switches, routers)
- Fiber optic infrastructure
- Database components

**Format**: File Observer format for CP4AIOps
- Each line starts with `V:` for vertex (node)
- JSON format with topology attributes
- `_references` define relationships between components

### 2. Alert Ingestion File
**Location**: `tools/01_demo/INCIDENT_FILES/robot-shop-network/events_rest/events_rest_nyc_orlando_switch_saturation.json`

**Purpose**: Contains alerts simulating the switch port saturation incident

**Format**: Newline-delimited JSON (one alert per line)
- Each alert has unique ID, timestamp placeholders
- Severity levels: 3-6 (3=Low, 4=Medium, 5=High, 6=Critical)
- Includes golden signal insights for AI correlation
- Links to mock monitoring dashboards

## Installation Instructions

### Step 1: Import Topology File

1. **Copy the topology file** to the CP4AIOps file observer location:
   ```bash
   # The file is already in the correct location:
   # ansible/roles/ibm-aiops-demo-content/templates/topology/robotshop-nyc-orlando-network.txt
   ```

2. **Import via CP4AIOps UI**:
   - Navigate to: Data and tool connections > File Observer
   - Create new File Observer job
   - Upload: `robotshop-nyc-orlando-network.txt`
   - Run the observer job

3. **Verify topology**:
   - Go to: Topology > Resource Management
   - Search for: "robotshop" or "ORL-SQL-SWITCH-PORT"
   - Verify all components are visible

### Step 2: Configure Alert Ingestion

1. **Locate the alert file**:
   ```bash
   cd /Users/raghuhulgundi/Desktop/ibm-aiops-deployer/tools/01_demo/INCIDENT_FILES/robot-shop-network/events_rest/
   ```

2. **The file can be used with the existing incident simulation scripts**:
   - Modify `robotshop_incident_network.sh` to use this new alert file
   - Or create a new script following the pattern

### Step 3: Create Incident Simulation Script

Create a new script: `tools/01_demo/robotshop_incident_nyc_orlando.sh`

```bash
#!/bin/bash

export APP_NAME=robot-shop-nyc-orlando
export LOG_TYPE=elk
export EVENTS_TYPE=noi
export EVENTS_SKEW="-120M"
export LOGS_SKEW="-90M"
export METRICS_SKEW="+5M"

# Use the NYC-Orlando alert file
export EVENTS_FILE="./tools/01_demo/INCIDENT_FILES/robot-shop-network/events_rest/events_rest_nyc_orlando_switch_saturation.json"

# Rest of the script follows the pattern from robotshop_incident_network.sh
```

## Testing the Scenario

### Pre-requisites
1. CP4AIOps installed and configured
2. Topology imported and visible
3. Event manager configured
4. AI models trained (optional but recommended)

### Running the Scenario

1. **Import the topology** (one-time setup)
2. **Run the incident simulation script**:
   ```bash
   cd /Users/raghuhulgundi/Desktop/ibm-aiops-deployer/tools/01_demo
   ./robotshop_incident_nyc_orlando.sh
   ```

3. **Expected Results**:
   - Story created with ~15 correlated alerts
   - Root cause identified: ORL-SQL-SWITCH-PORT saturation
   - Blast radius showing affected services:
     - mysql-db (direct impact)
     - ratings-id (connection errors)
     - catalogue-id (slow response)
     - web-id (user experience degradation)
   - Network path visualization from NYC to Orlando

### Validation Points

1. **Topology Correlation**:
   - Alerts should correlate to topology entities
   - Network path should be visible: NYC → Fiber → Orlando → Switch → DB

2. **Golden Signals**:
   - Saturation signal on switch port (cause)
   - Availability signal on database (cause)
   - Latency signal on catalogue (effect)
   - Error signal on ratings (effect)

3. **Probable Cause**:
   - AI should identify switch port saturation as root cause
   - Database timeout as secondary cause

## Customization

### Adjusting Severity
Edit the JSON file to change alert severity:
- Severity 6: Critical (red)
- Severity 5: High (orange)
- Severity 4: Medium (yellow)
- Severity 3: Low (blue)

### Adding More Alerts
Follow the JSON format:
```json
{ "id": "unique-id-MY_ID", "occurrenceTime": "MY_TIMESTAMP", "summary": "Alert description", "severity": 6, "type": { "eventType": "problem", "classification": "Category" }, "expirySeconds": 6000000, "sender": { "type": "host", "name": "Source System", "sourceId": "source" }, "resource": { "type": "resource_type", "name": "resource_name", "application": "robot-shop" }, "details": { }}
```

### Modifying Topology
Edit `robotshop-nyc-orlando-network.txt`:
- Add new components with `V:` prefix
- Define relationships in `_references` array
- Use appropriate `entityTypes` (vm, switch, router, database, etc.)
- Include geolocation for map visualization

## Troubleshooting

### Topology Not Showing
- Verify file observer job completed successfully
- Check for JSON syntax errors in topology file
- Ensure `matchTokens` are unique

### Alerts Not Correlating
- Verify resource names match topology entities
- Check `matchTokens` and `mergeTokens` alignment
- Ensure application tag is consistent ("robot-shop")

### No Story Created
- Verify event manager is running
- Check temporal grouping is enabled
- Ensure sufficient alerts are generated (minimum 3-5)

## Demo Script

### Narrative
"We have a distributed RobotShop application where the web interface runs in NYC, but the catalogue and ratings services need to access a MySQL database located in Orlando. The services communicate over a fiber optic network spanning multiple routers and switches.

The problem started when the switch port connecting to the SQL database VM became saturated at 95% utilization. This 1Gbps port became a bottleneck, causing:
- Database connection timeouts
- Slow query performance
- Application errors in the ratings service
- Degraded user experience on the web interface

CP4AIOps correlated all these symptoms and identified the root cause as the saturated switch port in Orlando, showing the complete network path and blast radius of the issue."

## Additional Notes

- **Geolocation**: NYC coordinates: -73.97721, 40.74505 | Orlando: -81.37924, 28.53834
- **Network Distance**: ~1,200 miles via fiber
- **Normal Latency**: <10ms end-to-end
- **Problem Latency**: 250ms+ due to saturation
- **Bandwidth**: 10Gbps fiber, 1Gbps switch port (bottleneck)

## Support

For issues or questions:
- Review existing RobotShop scenarios in `tools/01_demo/INCIDENT_FILES/robot-shop-network/`
- Check CP4AIOps documentation for file observer format
- Refer to the main repository README for general troubleshooting

---
**Created**: 2026-04-30  
**Author**: Custom scenario for NYC-Orlando network demonstration  
**Version**: 1.0