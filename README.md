# n8n-PatchMon-Semaphore-patching-workflow
n8n workflows that are part of patching system that utilizes Semaphore to create snapshots before patching. Can distinguish between security-only and other updates. Utilizes groups, exceptions for patching, locks of hosts when being patched by using Proxmox tags and disabling of alarms during patching using Uptimekuma.
