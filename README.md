# Bridging Language & Machines
### An End-to-End Experiment: LLM → MCP → ROS2 → Robot

**Author:** Pavan Kumar  
**Date:** March 2026  
**Platform:** Ubuntu 24.04 · HP ZBook Power G11 · NVIDIA RTX 2000 Ada (8GB)

---

## Executive Summary

In a single session, we constructed a working pipeline where natural language typed into an LLM is translated into real-time robot commands across multiple physics simulators. The pipeline chains **Claude Code** (via Anthropic's Claude) through the **Model Context Protocol (MCP)** to a **ROS2 bridge**, which dispatches commands to both a **Gazebo mobile robot** and a **MuJoCo robotic arm** — simultaneously.

NVIDIA Isaac Sim was installed but encountered compatibility issues on Ubuntu 24.04.

| Metric | Value |
|--------|-------|
| Simulators Connected | 2 (Gazebo + MuJoCo) |
| Live ROS2 Topics | 13+ |
| MCP Pipeline | 1 unified connection |

### The Working Pipeline

```
Claude Code → MCP Server → rosbridge (9090) → ROS2 Jazzy → Gazebo / MuJoCo
```

---

## The Build Journey

### Phase 1 — ROS2 Foundation

The first challenge: Ubuntu 24.04 (Noble) ships with Python 3.12 and is not compatible with ROS2 Humble (which targets Ubuntu 22.04). The pivot to **ROS2 Jazzy** resolved the distribution mismatch.

| Step | Status | Command | Notes |
|------|--------|---------|-------|
| Install curl | ✅ Fixed | `sudo apt install -y curl` | Not included in Ubuntu 24.04 minimal |
| ROS2 Humble | ❌ Failed | — | Incompatible with Ubuntu 24.04, GPG key errors |
| ROS2 Jazzy | ✅ Working | `sudo apt install -y ros-jazzy-desktop` | Full desktop install succeeded |
| TurtleBot3 (first attempt) | ❌ Failed | `sudo apt install -y ros-jazzy-turtlebot3* ros-jazzy-gazebo-ros-pkgs` | gazebo-ros-pkgs not available for Jazzy, rolled back all packages silently |
| TurtleBot3 (second attempt) | ✅ Working | `sudo apt install -y ros-jazzy-turtlebot3 ros-jazzy-turtlebot3-gazebo ros-jazzy-turtlebot3-teleop ros-jazzy-turtlebot3-msgs ros-jazzy-turtlebot3-description ros-jazzy-turtlebot3-simulations ros-jazzy-turtlebot3-bringup ros-jazzy-turtlebot3-navigation2` | Installed each package explicitly |
| Gazebo launch | ✅ Working | `export TURTLEBOT3_MODEL=burger && ros2 launch turtlebot3_gazebo turtlebot3_world.launch.py` | Gazebo Harmonic with TurtleBot3, 13 ROS2 topics active |
| rosbridge | ✅ Working | `sudo apt install -y ros-jazzy-rosbridge-server && ros2 launch rosbridge_server rosbridge_websocket_launch.xml` | WebSocket on port 9090 |

### Phase 2 — MCP Server + Claude Code

The ROS MCP Server by robotmcp translates LLM tool calls into ROS2 WebSocket commands.

| Step | Status | Command | Notes |
|------|--------|---------|-------|
| Clone ros-mcp-server | ✅ Working | `git clone https://github.com/robotmcp/ros-mcp-server.git` | v3.0.1, FastMCP-based |
| Install dependencies | ✅ Working | `uv venv && source .venv/bin/activate && uv pip install -e .` | Uses pyproject.toml, not requirements.txt |
| Claude Code install (first attempt) | ❌ Failed | `curl -fsSL https://cli.claude.com/install.sh \| sh` | Wrong URL — cli.claude.com doesn't exist |
| Claude Code install (second attempt) | ✅ Working | `curl -fsSL https://claude.ai/install.sh \| bash` | Native binary, no Node.js required |
| Connect MCP to Claude Code | ✅ Working | `claude mcp add ros-mcp-server uv -- --directory /home/pavan-kumar/ros-mcp-server run server.py` | MCP server registered |
| First LLM → Robot command | ✅ Working | Asked "What ROS topics are available?" | Claude returned all 13 topics with message types |

### Phase 3 — MuJoCo Robotic Arm

Added a second simulator to demonstrate multi-robot control through one MCP connection.

| Step | Status | Command | Notes |
|------|--------|---------|-------|
| MuJoCo install | ✅ Working | `pip install mujoco --break-system-packages` | Version 3.5.0 |
| mujoco_ros2_control build (dependencies) | ❌ Failed | `colcon build` | Missing controller_manager, glfw3 |
| Install missing deps | ✅ Fixed | `sudo apt install -y ros-jazzy-controller-manager ros-jazzy-ros2-control ros-jazzy-ros2-controllers libglfw3-dev` | — |
| mujoco_ros2_control build (API) | ❌ Failed | `colcon build` | C++ ResourceManager constructor changed in Jazzy's ros2_control — API incompatibility |
| Custom Python bridge | ✅ Working | `python3 ~/mujoco_ros_bridge.py` | 90-line script: MuJoCo sim → ROS2 topics |
| MuJoCo viewer (threaded) | ❌ Crashed | `mujoco.viewer.launch()` in thread | Segfault — not thread-safe with ROS2 spin |
| MuJoCo viewer (passive) | ✅ Working | `mujoco.viewer.launch_passive()` with `viewer.sync()` | Thread-safe viewer with ROS2 integration |
| Dual-simulator control | ✅ Working | — | Claude controlling TurtleBot3 + MuJoCo arm simultaneously |

### Phase 4 — NVIDIA Isaac Sim (Attempted)

| Step | Status | Command | Notes |
|------|--------|---------|-------|
| Python version mismatch | ❌ Failed | `pip install isaacsim` in Python 3.12 | Isaac Sim requires Python 3.11 exactly |
| Python 3.11 venv | ✅ Fixed | `sudo apt install -y python3.11 python3.11-venv && python3.11 -m venv ~/isaac_env` | — |
| Isaac Sim pip install | ✅ Working | `pip install isaacsim` then `pip install "isaacsim[all,extscache]" --extra-index-url https://pypi.nvidia.com` | Version 5.1.0 installed, EULA accepted |
| GCC 11 for Ubuntu 24.04 | ✅ Working | `sudo apt install -y gcc-11 g++-11` | Isaac Sim requires GCC 11, not 12+ |
| SimulationApp Python API | ❌ Failed | `SimulationApp({'headless': False})` | Returns NoneType — extension not found |
| Isaac Sim GUI | ⚠️ Partial | `isaacsim isaacsim.exp.full.kit` | Opens but keeps closing/crashing |
| ROS2 bridge | ❌ Failed | `--enable isaacsim.ros2.bridge` | No ROS2 topics published. No standalone pip package for the bridge. |

---

## Final Status Matrix

| Component | Status | Version | Notes |
|-----------|--------|---------|-------|
| ROS2 Jazzy | ✅ Working | Jazzy Jalisco | Full desktop on Ubuntu 24.04 |
| Gazebo + TurtleBot3 | ✅ Working | Harmonic | Mobile robot with LiDAR, IMU, odometry |
| rosbridge | ✅ Working | Port 9090 | WebSocket bridge MCP ↔ ROS2 |
| ROS MCP Server | ✅ Working | 3.0.1 | FastMCP, stdio transport |
| Claude Code | ✅ Working | Native installer | LLM client with MCP on Linux |
| MuJoCo | ✅ Working | 3.5.0 | 3-DOF arm via custom Python bridge |
| mujoco_ros2_control | ❌ Failed | — | C++ API mismatch with Jazzy |
| NVIDIA Isaac Sim | ⚠️ Partial | 5.1.0 | Installed, GUI unstable, ROS2 bridge non-functional |

---

## Working Architecture

Four simultaneous terminals are required:

```bash
# Terminal 1 — Gazebo Simulation
source /opt/ros/jazzy/setup.bash
export TURTLEBOT3_MODEL=burger
ros2 launch turtlebot3_gazebo turtlebot3_world.launch.py

# Terminal 2 — rosbridge WebSocket
source /opt/ros/jazzy/setup.bash
ros2 launch rosbridge_server rosbridge_websocket_launch.xml

# Terminal 3 — MuJoCo Arm
source /opt/ros/jazzy/setup.bash
python3 ~/mujoco_ros_bridge.py

# Terminal 4 — Claude Code with MCP
cd ~/ros-mcp-server
claude
```

### ROS2 Topics Available

```
/clock              /cmd_vel                 /imu
/joint_states       /odom                    /scan
/tf                 /tf_static               /robot_description
/mujoco/joint_states    /mujoco/joint_commands    /mujoco/end_effector_pos
/parameter_events   /rosout                  /client_count
/connected_clients
```

---

## Constraints & Limitations

### Hardware & Platform

- **Ubuntu 24.04 is too new** for many robotics packages — ROS2 Humble, Isaac Sim, and mujoco_ros2_control all target 22.04
- **RTX 2000 Ada with 8GB VRAM** is at minimum spec for Isaac Sim — complex scenes require 16GB+
- **No official Claude Desktop for Linux** — used Claude Code CLI as the MCP client
- **MuJoCo viewer thread-safety** requires `launch_passive()` pattern, not `launch()`

### Software Ecosystem

- Isaac Sim 5.1 pip package requires **Python 3.11 exactly** — not 3.10, not 3.12
- The mujoco_ros2_control C++ package hasn't been updated for Jazzy's ros2_control API
- Isaac Sim ROS2 bridge has **no standalone pip package** — tightly coupled to Omniverse runtime
- rosbridge is the **single point of failure** — all MCP-to-ROS communication flows through one WebSocket

---

## What This Makes Possible

### Immediate Capabilities (Working Today)

- Natural language control of a mobile robot — "move forward 1 meter, turn left 90 degrees"
- Natural language control of a robotic arm — "set shoulder to 1.5 radians"
- Real-time sensor reading via conversation — "what does the LiDAR see?"
- Multi-robot orchestration — control Gazebo and MuJoCo robots simultaneously
- Robot debugging via natural language — browse topics, inspect message types, call services

### Near-Term Extensions

- **Voice-controlled robotics** — pipe speech-to-text into Claude Code
- **Vision-language reasoning** — subscribe to camera topics, send frames to VLMs
- **Autonomous task planning** — LLM decomposes "explore the room" into motor commands
- **Anomaly detection** — LLM monitors sensor streams, alerts on unusual readings
- **Multi-tool orchestration** — add Blender MCP (3D design) + Home Assistant MCP (smart devices)

---

## Key Takeaways

**1. The MCP abstraction is powerful.** By placing an MCP server between the LLM and ROS2, we decouple the intelligence layer from the robot layer. Any MCP-compatible LLM (Claude, GPT, Gemini) can control any ROS robot without modification.

**2. Ubuntu 22.04 is still the safe bet for robotics.** Every compatibility issue we hit — ROS2 Humble, Isaac Sim, mujoco_ros2_control — traces back to being on 24.04. The ecosystem hasn't fully migrated yet.

**3. Python bridges beat C++ packages for rapid prototyping.** When the compiled mujoco_ros2_control package failed, a 90-line Python script achieved the same result in minutes. For LLM-driven robotics, the speed of iteration matters more than the speed of execution.

**4. The ecosystem is nascent but accelerating.** The ros-mcp-server project went from first release in September 2025 to 825+ GitHub stars by March 2026. Research papers on "Agentic Embodied AI" via MCP are appearing monthly. This is early, but the direction is clear.

---

## Hardware

| Component | Specification |
|-----------|--------------|
| Machine | HP ZBook Power 16 inch G11 Mobile Workstation |
| GPU | NVIDIA RTX 2000 Ada Generation — 8GB VRAM |
| Driver | NVIDIA 580.95.05 — CUDA 13.0 |
| OS | Ubuntu 24.04 LTS (Noble Numbat) |
| ROS2 | Jazzy Jalisco |
| Python | 3.12 (system) + 3.11 (Isaac Sim venv) |

---

*Built during a single exploratory session · March 9–10, 2026*  
*LLM → MCP → ROS2 → Robot*
