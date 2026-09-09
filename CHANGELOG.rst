=======================================================
specsnl.specsops Collection Changelog 0.1 Release Notes
=======================================================

.. contents:: Topics

v0.1.0
======

Release Summary
---------------

First release of ``specsnl.specsops``. The collection provisions and hardens Specs Ubuntu 24.04 (noble) hosts: base system, hardening, firewall, unattended upgrades, swap, log rotation, image cleanup and PostgreSQL.

Major Changes
-------------

- Initial release of the ``specsnl.specsops`` collection — eight roles for provisioning and hardening Specs Ubuntu 24.04 (noble) hosts.

Minor Changes
-------------

- base - baseline system setup: apt update/upgrade, core packages, locale, timezone and ``sysctl`` tuning. Configures ``needrestart``, ``DPkg::Lock::Timeout`` and ``force-confdef,force-confold`` so an unattended apt upgrade cannot hang on a prompt or fail on a held lock. Optional ``base_prefer_ipv4`` prefers IPv4 in ``/etc/gai.conf``.
- cleanup - build-time image cleanup — apt autoremove/clean and temp directory wipe.
- firewall - ufw baseline — default deny incoming, allow outgoing, and configurable rules.
- hardening - OS hardening: an sshd drop-in that disables password and keyboard-interactive authentication, plus fail2ban. Installs ``openssh-server`` and prepares the host keys and ``/run/sshd`` itself, so the role is self-contained on minimal images.
- logrotate - caps rotated log size and enables compression globally in ``/etc/logrotate.conf``.
- postgresql - PostgreSQL installation from PGDG with tuning, ``listen_addresses``, ``port``, timezone and ``pg_hba.conf`` entries. Only opens a ufw rule when the server listens beyond loopback.
- swap - creates, formats, persists and activates a swap file, and applies the related sysctl tuning.
- unattended_upgrades - automatic security updates via unattended-upgrades and chrony, including the periodic download, autoclean and Ubuntu ESM origins.
