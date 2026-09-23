# 🤖 RootAgent

[![Platform](https://img.shields.io/badge/Platform-Android-green.svg)](https://github.com/Machine-farmer/RootAgent)
[![Agent](https://img.shields.io/badge/Agent%20Ready-Yes-blue.svg)](https://github.com/Machine-farmer/RootAgent)
[![OS](https://img.shields.io/badge/Tested%20on-Kali%20%7C%20Ubuntu-orange.svg)](#-prerequisites)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](http://makeapullrequest.com)

> **⚠️ EDUCATIONAL & DEVELOPMENT PURPOSE ONLY**: These prompts are designed to automate Android rooting for development and testing. Use responsibly.

A collection of battle-tested, highly-constrained LLM agent prompts designed to automate the process of rooting Android devices and emulators. Built for AI agents capable of executing shell commands, these runbooks guarantee minimal user interaction, strict safety constraints (zero data loss), and reliable rooting via Magisk.

Currently tested and verified on **Kali Linux** and **Ubuntu** (Debian variants). 

---

## 📋 Table of Contents

- [About The Project](#-about-the-project)
- [Features](#-features)
- [Available Prompts](#-available-prompts)
- [Prerequisites](#-prerequisites)
- [Usage](#-usage)
- [Safety & Constraints](#-safety--constraints)
- [Contributing](#-contributing)
- [Contact & Support](#-contact--support)
- [License](#-license)

---

## 🔍 About The Project

Rooting Android devices manually involves executing a series of precise `adb` and `fastboot` commands, fetching the correct factory images, and patching boot images. One wrong command (like wiping `vbmeta`) can lead to unrecoverable data loss or a bricked device.

This repository provides **Agent-Ready Prompts** — meticulous instructions designed to be fed directly to an autonomous AI coding agent. The prompts enforce strict constraints on the agent, preventing it from making dangerous assumptions and forcing it to verify checksums and build numbers before taking action.

---

## ✨ Features

- **🤖 AI-Native Runbooks** - Instructions optimized for LLM comprehension and execution.
- **🛡️ Strict Safety Gates** - Agents are explicitly forbidden from wiping data or touching `vbmeta`.
- **✅ Verification First** - Enforces SHA-256 checksum validation for all downloaded factory images.
- **📱 Zero-Touch Rooting** - Fully automates Magisk boot image patching and flashing via the shell.
- **🔄 Auto-Recovery** - Built-in troubleshooting rules for the agent to handle common ADB/Fastboot errors.

---

## 📱 Available Prompts

Currently, the repository includes two battle-tested automation prompts:

| Prompt File | Target Device | Android Version | Root Method |
|------|-----------|------------|--------|
| `Android-studio-emulator-root.md` | Android Studio Emulator (x86_64) | Android 16 (API 36) | Magisk + rootAVD (FAKEBOOTIMG) |
| `pixel-6a-root.md` | Google Pixel 6a (`bluejay`) | Android 17 / SDK 37 | Magisk (boot patching) |

*These are just the start! More devices coming soon. Contributions welcome!*

---

## 🔧 Prerequisites

Your host machine (where the AI agent runs) must have:

- **OS**: Ubuntu, Kali Linux, or any Debian variant (macOS/Windows untested).
- **Android SDK Platform-Tools**: `adb` and `fastboot` must be installed.
- **Host Tools**: `curl`, `unzip`, `strings`, `sha256sum`.
- **An AI Agent**: A CLI-based LLM agent with shell execution capabilities (e.g., Aider, OpenInterpreter, Claude Engineer, etc.).

---

## 🚀 Usage

1. **Clone the repository:**
   ```bash
   git clone https://github.com/Machine-farmer/RootAgent.git
   cd RootAgent
   ```

2. **Connect your device / Start your emulator:**
   Ensure your device is connected via USB with **USB Debugging enabled**, or your Android Studio emulator is ready to be launched.

3. **Feed the prompt to your AI agent:**
   Using your preferred CLI agent, pass the markdown file as the system prompt or initial instruction.

   *Example using a generic agent CLI:*
   ```bash
   agent-cli --prompt-file pixel-6a-root.md
   ```

   *Alternatively, just copy the contents of the `.md` file and paste it into your AI assistant chat if it has access to your local terminal.*

4. **Supervise the Agent:**
   The agent will execute the commands in the background. It will ask for your interaction only when strictly necessary (e.g., granting Superuser access on the device screen).

---

## 🔒 Safety & Constraints

These prompts are engineered with strict "Hard Rules" to prevent bricking or data loss:

- **Never touch `vbmeta`**: Bypassing verification via `vbmeta` is a common cause of forced data wipes. The agent is strictly forbidden from doing this.
- **Never wipe**: Explicit commands forbidding `fastboot -w` or `erase userdata`.
- **Test before flash**: The agent is instructed to use `fastboot boot` to prove the patched image works *before* writing it permanently to the active slot.
- **Query, don't assume**: The agent dynamically fetches `ro.build.id` and slot suffixes instead of relying on hardcoded values.

---

## 🤝 Contributing

Contributions are highly encouraged! Since this is a new and emerging concept, community help is needed to expand device support.

### How to Contribute

1. **Fork** the repository
2. **Create** a new branch for your device (`git checkout -b feature/DeviceName-Root`)
3. **Write** your prompt following the strict constraint style of existing files. Ensure you include safety gates and validation steps.
4. **Test** it thoroughly on your local Linux machine.
5. **Open** a Pull Request

### Contribution Ideas
- Add prompts for OnePlus, Samsung, or Xiaomi devices
- Expand and verify support for Windows/macOS host toolchains
- Improve existing prompts with better error handling

---

## 📞 Contact & Support

- **Issues**: [GitHub Issues](https://github.com/Machine-farmer/RootAgent/issues)
- **Discussions**: [GitHub Discussions](https://github.com/Machine-farmer/RootAgent/discussions)

---

## 📜 License

This project is licensed under the **MIT License**.

---

<div align="center">

**Made for Automation & Security Research**

*Remember: Unlocking your bootloader and rooting your device carries inherent risks. While these scripts are designed to be safe, you assume all responsibility for your hardware.*

</div>
