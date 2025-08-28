### **Phase 0: Groundwork & Environment Setup**

**Goal:** Establish a stable, repeatable hardware and software foundation for a single Cerberus node. This phase is about getting off of your development machine and onto the target hardware.

**Key Activities:**
1.  **Hardware Selection & Procurement:**
    *   **SBC (Single Board Computer):** Finalize selection. A Raspberry Pi 4/5 is excellent for this phase. Procure at least **three** units to enable mesh testing later.
    *   **Power System:** Procure a reliable 5V power supply for bench testing and a drone-grade BEC (Battery Eliminator Circuit) for future integration.
    *   **Core Peripherals:** Procure necessary SD cards, cables, and a basic case/mount for bench work.
2.  **Software Environment:**
    *   **OS Installation:** Flash a lightweight, headless Linux distribution (e.g., Raspberry Pi OS Lite) onto the SD cards.
    *   **Environment Baselining:** Create a setup script (`setup.sh`) that installs all system dependencies, Python libraries (`requirements.txt`), and configures the OS (e.g., enables SSH, sets hostname). This ensures every node is identical.
    *   **Deployment Workflow:** Establish a simple workflow for deploying code from your Git repository to the SBCs (e.g., a `git pull` script or a basic Fabric/Ansible playbook).
3.  **Validation:**
    *   Run the *entire existing simulation suite* on one of the SBCs. It should pass exactly as it does on your development machine. This validates the environment.

**Deliverables:**
*   A finalized Bill of Materials (BOM) for a single node.
*   A fully documented, scripted setup process for a new node.
*   A "Golden Image" for the SBC's OS and software.
*   Successful execution of the simulation suite on the target hardware.

**Primary Risks & Mitigation:**
*   **Risk:** Hardware supply chain delays.
    *   **Mitigation:** Order all Phase 0/1 hardware immediately. Do not wait.
*   **Risk:** Inconsistent behavior between development machine and SBC.
    *   **Mitigation:** The validation step is designed to catch this. Freeze Python package versions to avoid dependency drift.

---

### **Phase 1: Core Mesh Networking - The First Real Signal**

**Goal:** Achieve reliable, secure, multi-hop communication between two or more physical nodes using Bluetooth LE, replacing the core BitChat simulation.

**Key Activities:**
1.  **Hardware Abstraction Layer (HAL) Development:**
    *   Create a `hardware/` directory. Inside, create `mesh_ble.py`.
    *   This module will contain a `MeshBLE` class with methods like `start_advertising()`, `scan_for_peers()`, `connect()`, `send()`, `receive()`.
    *   Use a modern library like `bleak` for the underlying implementation.
2.  **Protocol Porting:**
    *   Port the fundamental logic from BitChat (and your Python stubs like `onion_routing.py` and `proof_of_relay.py`) into a `protocols/` layer that uses the new `MeshBLE` HAL. This is a pure software task, but it's complex. Start with basic multi-hop forwarding before implementing the full onion routing.
3.  **Integration and Testing:**
    *   Modify the `main_mesh_simulation.py` to import and use the real `MeshBLE` HAL instead of the mock objects.
    *   **Benchtop Test:**
        *   **Test 1 (2 Nodes):** Verify that two SBCs can send and receive encrypted messages directly.
        *   **Test 2 (3 Nodes):** Place nodes A, B, and C. Move A and C out of direct range of each other. Verify that a message from A to C is successfully relayed through B.
    *   **Performance Baselining:** Measure latency and reliability for single-hop and multi-hop messages. Log this data.

**Deliverables:**
*   A functional `mesh_ble.py` HAL driver.
*   A Python implementation of the secure mesh protocol.
*   A successful demonstration of multi-hop message relay between three physical nodes.
*   A baseline performance report for the mesh network.

**Primary Risks & Mitigation:**
*   **Risk:** Python Bluetooth libraries are buggy or perform poorly.
    *   **Mitigation:** Allocate significant time for this phase. Test multiple libraries if necessary. This is a known high-risk area.
*   **Risk:** Porting the security protocol introduces vulnerabilities.
    *   **Mitigation:** Write extensive unit tests for the cryptographic functions. Perform peer code reviews focusing on the security implementation.

---

### **Phase 2: Infrastructure Connectivity - The 5G Uplink**

**Goal:** Give a single node the ability to connect to the internet reliably and securely over a 5G cellular network.

**Key Activities:**
1.  **Hardware Integration:**
    *   Select and procure a 5G modem compatible with your SBC (M.2 or USB).
    *   Physically connect the modem and its antennas. Install necessary Linux drivers and tools (e.g., `ModemManager`, `mmcli`).
2.  **HAL Development:**
    *   Create `hardware/five_g_modem.py`.
    *   This driver will be responsible for low-level modem interaction. It will likely use `subprocess` to call command-line tools like `mmcli` or interact with a serial port (`/dev/ttyUSB*`).
    *   Implement methods to `check_status()`, `get_signal_strength()`, and `connect()`.
3.  **Secure Connection Logic:**
    *   Implement the real logic for `secure_5g_module.py`.
    *   **Double Encryption:** Implement a real VPN tunnel. Using a lightweight solution like WireGuard is ideal. The module will be responsible for starting the tunnel.
    *   **IMSI Privacy:** While true IMSI rotation is a network feature, you can implement best practices at the device level, such as MAC address randomization if the modem supports it.
    *   **Base Station Auth:** This is the hardest to implement fully. Start by logging all visible cell tower information (`cell_id`, `mcc`, `mnc`). A real implementation might query an external database (like OpenCelliD) to validate the tower's location.
4.  **Testing:**
    *   Verify the node can reliably connect to the 5G network on boot.
    *   Verify all traffic is being routed through the VPN tunnel.
    *   Run a 24-hour soak test to check for connection drops.

**Deliverables:**
*   A functional `five_g_modem.py` HAL driver.
*   A security module that can establish a VPN connection over 5G.
*   A report detailing the modem's performance (signal strength vs. throughput) and stability.

**Primary Risks & Mitigation:**
*   **Risk:** 5G modem drivers are unstable or poorly documented.
    *   **Mitigation:** Choose a modem with proven Linux community support (e.g., Quectel, Sierra Wireless). Budget extra time for driver debugging.
*   **Risk:** Power consumption from the 5G modem is too high.
    *   **Mitigation:** Measure power draw under different loads (idle, connecting, high traffic). This data is critical for the final drone integration.

---

### **Phase 3: System Intelligence - The Communication Manager**

**Goal:** Enable a single node to autonomously and intelligently switch between the real mesh and 5G communication channels based on live environmental data.

**Key Activities:**
1.  **Driver Integration:**
    *   Modify `CommunicationManager` to use the real HAL drivers from Phases 1 and 2.
    *   The manager must now handle real-world failure states from the drivers (e.g., connection failed, modem not found).
2.  **Real-Time Data Polling:**
    *   Implement background threads or asynchronous tasks that continuously poll the HAL drivers for metrics: 5G signal strength, mesh peer count, link quality, etc.
3.  **Decision Logic Tuning:**
    *   The `_select_optimal_mode` logic is currently conceptual. Tune the thresholds based on the performance data gathered in previous phases. For example, determine the exact RSSI value below which a 5G connection becomes less reliable than a 2-hop mesh message.
4.  **Scenario-Based Testing:**
    *   **Test 1 (5G Failover):** Start with the node connected to 5G. Place the node in an RF-shielded box (or an area with no cell service). Verify it detects the loss of 5G and activates the mesh.
    *   **Test 2 (5G Recovery):** Remove the node from the shielded box. Verify it detects the return of a strong 5G signal and switches back from mesh to 5G.
    *   **Test 3 (Threat Response):** Manually trigger a mock "HIGH" threat level. Verify the manager forces a switch to the mesh network, even if 5G is available.

**Deliverables:**
*   A fully operational `CommunicationManager` running on a physical node.
*   A successful demonstration of autonomous failover and recovery between 5G and mesh.
*   A documented set of tuned decision-making rules and thresholds.

**Primary Risks & Mitigation:**
*   **Risk:** "Flapping" - the system rapidly switches back and forth between modes.
    *   **Mitigation:** Implement hysteresis in the decision logic. For example, don't switch back to 5G unless the signal has been "good" for at least 30 seconds.
*   **Risk:** The transition period is too slow, causing packet loss.
    *   **Mitigation:** Refine the `ModeTransitionController` to implement a "make-before-break" connection strategy where possible.

---

### **Phase 4: Advanced Resilience & Final Integration**

**Goal:** Integrate the satellite and emergency beacon failsafe channels and physically prepare the system for drone mounting.

**Key Activities:**
1.  **Failsafe Hardware Integration:**
    *   Procure and integrate a satellite module (e.g., Iridium, Swarm) and a LoRa radio module.
    *   Develop the corresponding HAL drivers (`satellite_modem.py`, `emergency_radio.py`). These are typically simpler serial interfaces.
2.  **Final Manager Integration:**
    *   Incorporate these last two modules into the `CommunicationManager`'s logic as the lowest-priority fallback options.
    *   Integrate the `CommunicationBlackoutDetector` with the real drivers.
3.  **Physical Hardening & Form Factor:**
    *   Design and 3D print/fabricate a compact, vibration-dampened enclosure for the entire electronics package (SBC, modems, antennas).
    *   Finalize the wiring harness using reliable connectors.
4.  **Full System Soak Test:**
    *   Run the complete, assembled node for 48-72 hours straight, simulating random link failures (by temporarily disconnecting antennas) to ensure the entire decision tree works as expected.

**Deliverables:**
*   A complete, self-contained, and ruggedized Cerberus node.
*   Successful demonstration of failover to all four communication modes.
*   A final power consumption report.

**Primary Risks & Mitigation:**
*   **Risk:** RF interference between the different antennas (5G, GPS, LoRa, BLE).
    *   **Mitigation:** Follow best practices for antenna placement (physical separation, ground planes). This may require significant experimentation.

---

### **Phase 5: Field Validation & Operationalization**

**Goal:** Test the hardened Cerberus node in a real-world environment on a drone platform and build the tools for operational monitoring.

**Key Activities:**
1.  **Drone Integration:**
    *   Mount the Cerberus package onto a drone.
    *   Integrate the power system with the drone's main battery.
2.  **Progressive Field Testing:**
    *   **Walk Test:** Carry the drone around an area with mixed 5G/no-5G coverage. Log all mode switches and verify they happen as expected.
    *   **Hover Test:** Fly the drone to a low altitude and verify that vibration and motor noise do not interfere with communications.
    *   **Range Test:** Fly the drone to the edge of 5G coverage and test the failover to mesh with another node on the ground.
3.  **Implement Operator Dashboard:**
    *   Develop the `RealTimeDashboard`. A simple web server (e.g., Flask) running on the node that serves a webpage showing current status, signal strength, and active threats would be a powerful tool.
4.  **Final Documentation:**
    *   Create an "Operator's Manual" detailing pre-flight checks, operational procedures, and troubleshooting steps.

**Deliverables:**
*   Video evidence and data logs from successful field tests.
*   A functional real-time monitoring dashboard.
*   A complete set of operational documentation.

**Primary Risks & Mitigation:**
*   **Risk:** Unforeseen environmental factors (weather, RF environment) negatively impact performance.
    *   **Mitigation:** This is the purpose of field testing. Log everything. Be prepared to go back to Phase 3 to retune the decision logic based on real-world data.
