# When Language Learned to Move
### Connecting LLMs to Robotic Simulators via MCP — A Complete Exploration Report

**Author:** Pavan Kumar  
**Date:** March 2026  
**Platform:** Ubuntu 24.04 · HP ZBook Power G11 · RTX 2000 Ada (8GB)

---

## Executive Summary

Across multiple sessions in March 2026, we built a working pipeline where natural language commands typed into Claude Code are translated into real-time robot actions across multiple physics simulators. Four simulators were attempted — **Gazebo**, **MuJoCo**, **NVIDIA Isaac Sim**, and **PyBullet** — with three successfully connected to Claude via the Model Context Protocol (MCP).

The final achievement: two robot arms coordinating a block handoff, orchestrated by an LLM that not only executes commands but **reasons about why** it makes each decision.

| Metric | Value |
|--------|-------|
| Simulators Attempted | 4 (Gazebo, MuJoCo, Isaac Sim, PyBullet) |
| Successfully Connected | 3 |
| Live ROS2 Topics | 16+ |
| MCP Pipeline | 1 unified connection |

### The Architecture

```
Natural Language → Claude Code → MCP Server → rosbridge / PyBullet → Robot Simulator
```

---

## Phase 1: ROS2 Foundation + Gazebo

### What Worked

| Step | Command | Notes |
|------|---------|-------|
| Install curl | `sudo apt install -y curl` | Not in Ubuntu 24.04 minimal |
| ROS2 Jazzy | `sudo apt install -y ros-jazzy-desktop` | Jazzy for 24.04, not Humble |
| TurtleBot3 | `sudo apt install -y ros-jazzy-turtlebot3 ros-jazzy-turtlebot3-gazebo ros-jazzy-turtlebot3-teleop ros-jazzy-turtlebot3-msgs ros-jazzy-turtlebot3-description ros-jazzy-turtlebot3-simulations ros-jazzy-turtlebot3-bringup ros-jazzy-turtlebot3-navigation2` | Each package explicitly |
| Gazebo launch | `export TURTLEBOT3_MODEL=burger && ros2 launch turtlebot3_gazebo empty_world.launch.py` | Use empty_world for visibility |
| rosbridge | `sudo apt install -y ros-jazzy-rosbridge-server && ros2 launch rosbridge_server rosbridge_websocket_launch.xml` | Port 9090 |
| Robot movement | `ros2 topic pub /cmd_vel geometry_msgs/msg/TwistStamped "{twist: {angular: {z: 1.0}}}" --rate 10` | Confirmed spinning |

### What Failed

- **ROS2 Humble** — incompatible with Ubuntu 24.04 (GPG key errors)
- **TurtleBot3 glob install** — `ros-jazzy-turtlebot3* ros-jazzy-gazebo-ros-pkgs` failed on gazebo-ros-pkgs and silently rolled back all packages
- **Robot invisible** — commands succeeded but robot had traveled 1,443 meters off screen. Fix: kill all gz processes with `pkill -f gz` and restart fresh
- **Gazebo not restarting** — background gz processes keep old simulation alive. Must `pkill -f gz` before relaunching

---

## Phase 2: MCP Bridge + Claude Code

### What Worked

| Step | Command | Notes |
|------|---------|-------|
| Clone ros-mcp-server | `git clone https://github.com/robotmcp/ros-mcp-server.git` | v3.0.1, FastMCP |
| Install deps | `cd ros-mcp-server && uv venv && source .venv/bin/activate && uv pip install -e .` | Uses pyproject.toml |
| Claude Code | `curl -fsSL https://claude.ai/install.sh \| bash` | Native Linux binary |
| Connect MCP | `claude mcp add ros-mcp-server uv -- --directory ~/ros-mcp-server run server.py` | MCP registered |
| First query | "What ROS topics are available?" | Returned all 13 topics ✅ |

### What Failed

- **Wrong install URL** — `cli.claude.com` doesn't exist. Correct: `claude.ai/install.sh`

---

## Phase 3: MuJoCo Robotic Arm

### What Worked

| Step | Command | Notes |
|------|---------|-------|
| MuJoCo install | `pip install mujoco --break-system-packages` | v3.5.0 |
| Custom Python bridge | `python3 ~/mujoco_ros_bridge.py` | 90-line script replaces C++ package |
| Passive viewer | `mujoco.viewer.launch_passive()` with `viewer.sync()` | Thread-safe |
| Dual control | Claude controlling TurtleBot3 + MuJoCo arm simultaneously | Through one MCP |

### What Failed

- **mujoco_ros2_control** — C++ API mismatch with Jazzy's ros2_control ResourceManager
- **Missing dependencies** — needed `ros-jazzy-controller-manager`, `libglfw3-dev`
- **Viewer segfault** — `mujoco.viewer.launch()` not thread-safe. Fix: `launch_passive()`

---

## Phase 4: NVIDIA Isaac Sim (Attempted)

### What Partially Worked

| Step | Command | Notes |
|------|---------|-------|
| Python 3.11 venv | `sudo apt install -y python3.11 python3.11-venv && python3.11 -m venv ~/isaac_env` | Isaac requires 3.11 exactly |
| GCC 11 | `sudo apt install -y gcc-11 g++-11` | Isaac requires GCC 11 |
| Isaac Sim install | `pip install isaacsim && pip install "isaacsim[all,extscache]" --extra-index-url https://pypi.nvidia.com` | v5.1.0 |

### What Failed

- **SimulationApp** — returns NoneType, extension not found
- **GUI** — opens but crashes/closes repeatedly
- **ROS2 bridge** — no pip package, `--enable isaacsim.ros2.bridge` produces no topics
- **Root causes:** Ubuntu 24.04 not fully supported, 8GB VRAM at minimum threshold

---

## Phase 5: Multi-Arm Coordination

### MuJoCo Two-Arm Demo

| Step | Status | Notes |
|------|--------|-------|
| Scene loaded | ✅ | Two 5-DOF arms, grippers, table, 3 blocks, handoff/target zones |
| Touch sensor fix | ✅ | Removed broken `arm_a_touch_site_l` references from XML |
| Visual buffer fix | ✅ | Added `<visual><global offwidth="1920" offheight="1080"/></visual>` |
| Folder structure | ✅ | Created controllers/, planner/, mcp_bridge/, models/ subdirectories |
| Local demo | ✅ | Full pick → handoff → place with reasoning logs |
| FastMCP server | ✅ | Claude Code observes scene, picks blocks, commands arms |
| IK precision | ⚠️ | Arms move but don't reach exact positions — blocks not gripped |

### PyBullet Two-Arm Demo

| Step | Status | Notes |
|------|--------|-------|
| Install | ✅ | `pip install pybullet --break-system-packages` |
| Two Panda arms | ✅ | Built-in franka_panda/panda.urdf, facing each other |
| Built-in IK | ✅ | `p.calculateInverseKinematics` — one function call |
| Block picking | ✅ | Arm picks blocks using `p.createConstraint` |
| Handoff/placement | ⚠️ | Block slips during transfer — timing needs tuning |

---

## AI Reasoning Discoveries

The most valuable output was Claude's reasoning log — showing non-obvious optimizations:

> **Block Ordering:** "Picking block_green second (not block_blue) because block_blue's cylindrical shape might roll and knock into block_green, toppling it. Removing the tall, unstable block_green first reduces risk of cascade failures."

> **Handoff Positioning:** "Choosing handoff at x=-0.05 instead of center because the loaded arm does more work — shifting the handoff 5cm toward Arm A reduces its loaded travel distance. Arm B travels slightly further but is unloaded, so total energy is lower."

> **Shape-Aware Transfer:** "For the cylinder, raising handoff height to z=0.57. Higher handoff reduces the chance of the cylinder slipping during transfer — gravity assists the receiving gripper's closure."

> **Placement Strategy:** "Placing at x=0.42 (slightly inside target zone, not at center x=0.45) to leave room for subsequent blocks. Starting the placement pattern from the inner edge ensures all 3 blocks fit without collision."

---

## Final Status Matrix

| Component | Status | Version | Key Insight |
|-----------|--------|---------|-------------|
| ROS2 Jazzy | ✅ Working | Jazzy Jalisco | Use Jazzy for Ubuntu 24.04 |
| Gazebo + TurtleBot3 | ✅ Working | Harmonic | Kill bg processes between restarts |
| rosbridge | ✅ Working | Port 9090 | Single point connecting MCP ↔ ROS2 |
| ROS MCP Server | ✅ Working | 3.0.1 | FastMCP + stdio to Claude Code |
| Claude Code | ✅ Working | v2.1.81 | Native Linux installer |
| MuJoCo (single arm) | ✅ Working | 3.5.0 | launch_passive() for thread safety |
| MuJoCo (two-arm) | ⚠️ Partial | 3.5.0 | Reasoning works, IK imprecise |
| PyBullet (two-arm) | ⚠️ Partial | Latest | Built-in IK, handoff needs tuning |
| NVIDIA Isaac Sim | ❌ Failed | 5.1.0 | Ubuntu 24.04 unsupported, 8GB insufficient |

---

## Constraints

- **Ubuntu 24.04 is too new** — ecosystem targets 22.04 (ROS2 Humble, Isaac Sim, mujoco_ros2_control)
- **RTX 2000 Ada (8GB)** — cannot run Isaac Sim reliably, minimum 16GB recommended
- **No Claude Desktop for Linux** — CLI-only via Claude Code
- **MuJoCo viewer not thread-safe** — requires launch_passive() pattern
- **Custom IK unreliable** — use simulators with built-in IK (PyBullet)
- **Isaac Sim requires Python 3.11 exactly** — not 3.10, not 3.12
- **rosbridge is single point of failure** — all MCP-to-ROS communication flows through one WebSocket

## What's Now Possible

- Natural language robot control — "move forward, pick the red block"
- Multi-robot orchestration through one LLM
- AI reasoning about physical actions — explaining why, not just executing
- Any MCP-compatible LLM controls any ROS robot — zero code changes
- Shape-aware and physics-aware strategies discovered by AI
- Voice → LLM → MCP → Robot pipeline within reach
- Extensible with Blender MCP, Home Assistant MCP, and more

---

## Key Takeaways

**1. The protocol layer is the breakthrough.** MCP between LLM and simulator means any AI can control any robot. Intelligence and hardware are fully decoupled.

**2. Python bridges beat C++ for AI-driven robotics.** 90-line Python script replaced a failing C++ package. PyBullet's one-line IK replaced custom Jacobian solver. Speed of iteration > speed of execution.

**3. The reasoning log is more valuable than the motion.** When Claude explains "I pick the cube first because it's the most stable base" — that's the AlphaFold moment. The environment isn't the breakthrough; what AI discovers inside it is.

**4. Ubuntu 22.04 remains the safe bet.** Every compatibility issue traced to 24.04.

---

## Hardware

| Component | Specification |
|-----------|--------------|
| Machine | HP ZBook Power 16" G11 |
| GPU | NVIDIA RTX 2000 Ada — 8GB VRAM |
| Driver | NVIDIA 580.95.05 — CUDA 13.0 |
| OS | Ubuntu 24.04 LTS (Noble) |
| ROS2 | Jazzy Jalisco |
| Python | 3.12 (system) + 3.11 (Isaac Sim) |

---

## Video Demos

📹 **Gazebo Demo** — Claude Code controlling TurtleBot3 mobile robot via MCP  
📹 **MuJoCo Demo** — Claude Code controlling robotic arm via MCP  
📹 **Multi-Arm Demo** — Two-arm block pick and handoff coordination  

*(Videos recorded separately with OBS Studio)*

---

*Built across multiple exploratory sessions · March 2026*  
*LLM → MCP → Simulator → Robot Motion → AI Reasoning*  
*The robot doesn't just move — it reasons.*
