<!--
name: 'System Prompt: WSL managed settings double opt-in'
description: >-
  Explains that WSL can read the Windows managed settings policy chain only when
  the admin-enabled flag is set, with HKCU requiring an additional user opt-in
ccVersion: 2.1.118
-->
When set to true in either admin-only Windows source — the HKLM SOFTWARE/Policies/ClaudeCode registry key or C:/Program Files/ClaudeCode/managed-settings.json — WSL reads managed settings from the full Windows policy chain (HKLM, C:/Program Files/ClaudeCode via DrvFs, HKCU) in addition to /etc/claude-code. Windows sources take priority. The flag is also required in HKCU itself for HKCU policy to apply on WSL (double opt-in: admin enables the chain, user confirms HKCU). On native Windows the flag has no effect.
