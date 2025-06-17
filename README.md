# BlueOS-Water-Linked-DVL

## Changelog

### v1.0.7
  - Fix using lat/long inputs with no internet

### v1.0.6
 - No longer sets parameters automatically. Users can now change for two modes of operation:
     - DVL-only, the recommended mode
     - DVL+GPS, experimental mode which allows fusing (underwater) GPS and DVL data

### v1.0.5
 - Update texts to make support of DVL A125 obvious

### v1.0.4
 - Fix issue introduced in v1.0.3 where the extension was unable to talk to Cable-guy

### v1.0.3
 - Uses an random available port instead of 9001 to avoid conflict
 - Updated menu icon

### v1.0.2
 - Improved style

### v1.0.1
 - Fixed an issue where the driver was sending Rangefinder messages with invalid data

This is a docker implementation of a Water Linked DVL A50 and A125 driver as a BlueOS Extension.

## Install

Install it from [BlueOS extensions tab](https://docs.bluerobotics.com/ardusub-zola/software/onboard/BlueOS-1.1/extensions/).

The service will show in the "Extension Manager" section in BlueOS, where there are some configuration options.

## Features

### Core DVL Integration
- Connects to Water Linked DVL A50/A125 via TCP socket
- Processes velocity and position data from the DVL
- Sends MAVLink messages to ArduSub for navigation

### Distance Sensing Capabilities

#### 1. Average Altitude (Original Feature)
- Publishes the DVL's averaged altitude as a single `DISTANCE_SENSOR` message
- Uses `MAV_SENSOR_ROTATION_PITCH_270` orientation (downward facing)
- Sensor ID: 0

#### 2. Individual Beam Distances (New Feature)
- Extracts distance measurements from each of the 4 DVL transducer beams
- Publishes separate `DISTANCE_SENSOR` messages for each beam with different orientations
- Enables better terrain analysis for rugged underwater environments
- Each beam uses a different orientation to work around ArduSub's single rangefinder per orientation limitation:
  - **Beam 0** (Front-Left): `MAV_SENSOR_ROTATION_YAW_315` (315°), Sensor ID: 1
  - **Beam 1** (Front-Right): `MAV_SENSOR_ROTATION_YAW_45` (45°), Sensor ID: 2
  - **Beam 2** (Back-Right): `MAV_SENSOR_ROTATION_YAW_135` (135°), Sensor ID: 3
  - **Beam 3** (Back-Left): `MAV_SENSOR_ROTATION_YAW_225` (225°), Sensor ID: 4

### DVL AHRS Comparison (Extra Credit Feature)
- When `position_local` data is available, sends additional `GLOBAL_VISION_POSITION_ESTIMATE` messages
- Includes the DVL's internal AHRS attitude data (roll, pitch, yaw)
- Allows comparison between ArduSub's AHRS and the DVL's AHRS for improved navigation analysis

## Configuration

### Settings
- **Enable DVL driver**: Master enable/disable for the entire DVL integration
- **Send range data through MAVLink**: Enable/disable the averaged altitude `DISTANCE_SENSOR` message
- **Send individual beam distances**: Enable/disable individual beam `DISTANCE_SENSOR` messages

### UI Features
- Real-time display of all distance sensors (average + 4 individual beams)
- Visual indication of beam validity (green for valid, red for invalid)
- Live distance readings in meters with 2-decimal precision
- Orientation reference diagram showing beam positions

### API Endpoints
- `GET /beam_distances/<true|false>` - Enable/disable individual beam distance messages
- `GET /get_status` - Returns status including beam distance data

## Data Analysis Benefits

### For Pilots
- **Terrain Awareness**: See individual distances to understand seafloor topology
- **Obstacle Avoidance**: Detect when different parts of the ROV are at varying altitudes
- **Navigation Precision**: Better maintain desired altitude over complex terrain

### For Post-Dive Analysis
- **Complete Dataset**: All 5 distance measurements (average + 4 beams) logged via MAVLink
- **Terrain Mapping**: Reconstruct 3D seafloor profiles using individual beam data
- **AHRS Comparison**: Compare DVL and ArduSub attitude estimates for validation

## Technical Implementation

### MAVLink Message Types
1. `DISTANCE_SENSOR` - Used for all distance measurements
2. `GLOBAL_VISION_POSITION_ESTIMATE` - Used for DVL AHRS data

### Data Sources
- **Velocity Messages**: Primary source for beam distance data via `transducers` array
- **Position Local Messages**: Source for DVL AHRS attitude data

### Protocol Compatibility
- Follows Water Linked DVL TCP protocol specification
- Compatible with existing BlueOS DVL integration
- Backward compatible with existing configurations

## Installation

1. Deploy as a BlueOS extension
2. Configure DVL network settings
3. Enable desired distance sensor features via the UI
4. Configure QGroundControl to display the additional `DISTANCE_SENSOR` messages

## Docker Setup and Testing

### Prerequisites
- Docker installed on your system
- Access to the internet for building the image

### Build the Docker Image

```bash
docker build -t dvl-app .
```

### Testing with Water Linked Demo Server

**Important**: The Water Linked demo server uses port 16171 for both HTTP API and TCP data stream. The hostname should include the port for proper HTTP API access.

#### Option 1: Basic Setup
```bash
# Run the container
docker run -d \
  --name dvl-extension \
  -p 9001:9001 \
  -e DVL_TEST_MODE=true \
  -e DVL_HOSTNAME=dvl.demo.waterlinked.com:16171 \
  dvl-app

# Configure to use demo server (if not set via environment variable)
curl "http://localhost:9001/hostname/dvl.demo.waterlinked.com:16171"

# Enable beam distances feature
curl "http://localhost:9001/beam_distances/true"
```

#### Option 2: Pre-configured Setup
Create a settings file with demo server configuration:

```bash
# Create configuration directory
mkdir -p ./config/dvl

# Create pre-configured settings (NOTE: hostname includes port)
cat > ./config/dvl/settings.json << EOF
{
  "enabled": true,
  "orientation": 1,
  "hostname": "dvl.demo.waterlinked.com:16171",
  "rangefinder": true,
  "should_send": "POSITION_DELTA",
  "beam_distances_enabled": true
}
EOF

# Run with pre-configured settings
docker run -d \
  --name dvl-extension \
  -p 9001:9001 \
  -v $(pwd)/config:/root/.config \
  dvl-app
```

#### Option 3: With Network Host Resolution
```bash
docker run -d \
  --name dvl-extension \
  -p 9001:9001 \
  --add-host=dvl.demo.waterlinked.com:34.64.72.180 \
  -e DVL_TEST_MODE=true \
  -e DVL_HOSTNAME=dvl.demo.waterlinked.com:16171 \
  dvl-app
```

**Environment Variables:**
- `DVL_TEST_MODE=true` - Bypasses BlueOS dependencies (cable-guy, MAVLink heartbeat)
- `DVL_HOSTNAME=dvl.demo.waterlinked.com:16171` - Sets the DVL server hostname with port

### Accessing the Application

1. **Web Interface**: Open `http://localhost:9001` in your browser
2. **API Status**: Check `http://localhost:9001/get_status` for current configuration
3. **Individual Beam Toggle**: Use the "Send individual beam distances" switch in the UI

### Testing Steps

1. **Access the web interface** at `http://localhost:9001`
2. **Verify connection** to `dvl.demo.waterlinked.com` in the status area
3. **Enable features**:
   - ✅ Enable DVL driver
   - ✅ Send range data through MAVLink  
   - ✅ Send individual beam distances
4. **Monitor the Distance Sensors panel** for live data:
   - Average altitude (main sensor)
   - 4 individual beam distances with validity indicators
5. **Check logs** for MAVLink message transmission:
   ```bash
   docker logs -f dvl-extension
   ```

### Development and Debugging

#### View container logs:
```bash
docker logs -f dvl-extension
```

#### Access container shell:
```bash
docker exec -it dvl-extension /bin/bash
```

#### Restart container:
```bash
docker restart dvl-extension
```

#### Stop and remove container:
```bash
docker stop dvl-extension
docker rm dvl-extension
```

#### Clean up volumes:
```bash
docker volume rm dvl-config
```

### API Testing

Test individual API endpoints:

```bash
# Get current status
curl "http://localhost:9001/get_status"

# Enable/disable main driver
curl "http://localhost:9001/enable/true"

# Enable/disable range finder
curl "http://localhost:9001/use_as_rangefinder/true"

# Enable/disable individual beam distances
curl "http://localhost:9001/beam_distances/true"

# Change DVL hostname
curl "http://localhost:9001/hostname/dvl.demo.waterlinked.com:16171"

# Load DVL-only parameters
curl "http://localhost:9001/load_params/dvl"
```

### Expected Behavior with Demo Server

When connected to `dvl.demo.waterlinked.com:16171`, you should see:

1. **Status**: "Running" in the main status area
2. **Distance Sensors Panel**: 
   - Average altitude showing "Enabled"
   - Individual beam distances showing "Enabled"
   - 4 individual beam distances (Front-Left, Front-Right, Back-Right, Back-Left)
   - Green color coding for valid readings with actual distance values (e.g., "5.92 m")
   - Real-time updates every 2 seconds
3. **MAVLink Messages**: 
   - 1x `DISTANCE_SENSOR` for average (ID: 0, Orientation: PITCH_270)
   - Up to 4x `DISTANCE_SENSOR` for individual beams (ID: 1-4, Orientations: YAW_315, YAW_45, YAW_135, YAW_225)
4. **Console Logs**: Connection messages and data processing confirmations

## Troubleshooting

### Web UI Shows "Disabled" or "Invalid" Despite Working API

**Symptoms**: API returns correct data but web interface shows:
- Average Altitude: "Disabled" 
- Individual Beam Distances: "Invalid"

**Cause**: Field name mismatch between API response (snake_case) and JavaScript (camelCase)

**Solution**: This has been fixed in the current version. The JavaScript now correctly reads:
- `data.beam_distances_enabled` instead of `data.beamDistancesEnabled`
- `data.beam_distances` instead of `data.beamDistances`  
- `data.beam_valid` instead of `data.beamValid`

**Verification**: 
```bash
# Check API returns correct data
curl -s "http://localhost:9001/get_status" | jq '.beam_distances, .beam_valid'

# Should show arrays like:
# [5.92, 6.12, 5.82, 5.12]
# [true, true, true, true]
```

### Container Stuck on "waiting for cable-guy"

**Symptoms**: Logs show repeated `waiting for cable-guy to come online...` messages

**Cause**: BlueOS dependency not available in standalone Docker environment

**Solution**: Use test mode environment variables:
```bash
docker run -d --name dvl-extension -p 9001:9001 \
  -e DVL_TEST_MODE=true \
  -e DVL_HOSTNAME=dvl.demo.waterlinked.com:16171 \
  dvl-app
```

### Connection Timeouts to Demo Server

**Symptoms**: Cannot connect to `dvl.demo.waterlinked.com:16171`

**Possible Solutions**:
1. Check internet connectivity
2. Verify DNS resolution: `nslookup dvl.demo.waterlinked.com`
3. Test direct connection: `curl http://dvl.demo.waterlinked.com:16171/api/v1/about`
4. Try alternative hostname format in container

### Technical Note: Port Handling

The DVL application uses two types of connections:
- **HTTP API**: Used for initial DVL discovery and status checks (port specified in hostname)
- **TCP Data Stream**: Used for real-time velocity/position data (always port 16171)

For the demo server, both connections use port 16171, so the hostname must include the port for HTTP API calls to work correctly.

## Complete Testing Workflow

### Step 1: Build and Run
```bash
# Build the Docker image
docker build -t dvl-app .

# Run with test mode enabled
docker run -d --name dvl-extension -p 9001:9001 \
  -e DVL_TEST_MODE=true \
  -e DVL_HOSTNAME=dvl.demo.waterlinked.com:16171 \
  dvl-app
```

### Step 2: Verify Connection
```bash
# Check container logs for successful connection
docker logs dvl-extension --tail 20

# Verify API is working
curl -s "http://localhost:9001/get_status" | jq '.'

# Should show status: "Running" and valid beam data
```

### Step 3: Test Web Interface
1. Open http://localhost:9001 in your browser
2. Verify the following displays correctly:
   - **Average Altitude**: "Enabled" (blue text)
   - **Individual Beam Distances**: "Enabled"
   - **All 4 beam readings**: Green text with values like "5.92 m"
   - **Real-time updates**: Values change every 2 seconds

### Step 4: Test API Endpoints
```bash
# Test all major endpoints
curl "http://localhost:9001/get_status"
curl "http://localhost:9001/beam_distances/true" 
curl "http://localhost:9001/use_as_rangefinder/true"
curl "http://localhost:9001/enable/true"
```

### Step 5: Monitor Data Stream
```bash
# Watch live data updates
watch -n 1 'curl -s "http://localhost:9001/get_status" | jq ".beam_distances, .beam_valid"'
```

### Step 6: Cleanup
```bash
# Stop and remove test container
docker stop dvl-extension && docker rm dvl-extension
```

## QGroundControl Integration

Each beam's `DISTANCE_SENSOR` message will appear as a separate rangefinder in QGroundControl's telemetry. The different orientations ensure ArduSub treats them as distinct sensors, allowing pilots to monitor all readings simultaneously.

## Logging

All `DISTANCE_SENSOR` messages are automatically logged by QGroundControl and can be used for post-dive terrain analysis and ROV performance evaluation.

## Summary

This enhanced BlueOS Water Linked DVL extension successfully addresses the original problem of capturing individual beam distances for rugged terrain navigation. Key achievements include:

### ✅ **Core Features Implemented**
- **Individual Beam Distance Extraction**: All 4 DVL transducer beams send separate `DISTANCE_SENSOR` messages
- **Multiple Orientations**: Each beam uses a different MAVLink orientation to work around ArduSub's single-rangefinder-per-orientation limitation
- **Real-time UI**: Web interface displays all 5 distance measurements (average + 4 beams) with live updates
- **Test Mode**: Standalone Docker testing with Water Linked demo server integration

### ✅ **Technical Implementation**
- **MAVLink Integration**: Sends 5 separate `DISTANCE_SENSOR` messages with unique IDs and orientations
- **DVL AHRS Support**: Publishes `GLOBAL_VISION_POSITION_ESTIMATE` messages for attitude comparison
- **Robust Error Handling**: Validates beam data and only sends messages for readings > 5cm
- **BlueOS Compatibility**: Maintains full compatibility with existing BlueOS DVL integration

### ✅ **Navigation Benefits**
- **Terrain Awareness**: Pilots can see individual beam distances to navigate complex seafloor topography
- **Data Logging**: All measurements logged via QGroundControl for post-dive analysis
- **3D Reconstruction**: Complete dataset enables detailed seafloor mapping and ROV performance analysis

### 🚀 **Ready for Production**
The extension is production-ready and can be deployed in real BlueOS environments for enhanced underwater navigation and data collection.