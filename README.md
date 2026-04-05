THINKPAD T480 FINGERPRINT SCANNER SETUP - FEDORA 43 KDE
========================================================

STEP 1 - INSTALL PACKAGES
--------------------------
sudo dnf copr enable sneexy/python-validity
sudo dnf install open-fprintd fprintd-clients fprintd-clients-pam python3-validity


STEP 2 - DISABLE PREDESKTOP AUTHENTICATION IN BIOS (CRITICAL)
--------------------------------------------------------------
- Shut down completely
- Press F1 at the Lenovo logo to enter BIOS
- Go to Security -> Fingerprint
- Set "Predesktop Authentication" to Disabled
- Save with F10 and boot back in

NOTE: Skipping this step will cause the service to crash with
"Exception: Failed: 0401" and nothing will work.


STEP 3 - CREATE RUNTIME DIRECTORY AND DOWNLOAD FIRMWARE
--------------------------------------------------------
sudo mkdir -p /var/run/python-validity
sudo validity-sensors-firmware
sudo cp /var/run/python-validity/*.xpfwext /usr/share/python-validity/


STEP 4 - MAKE RUNTIME DIRECTORY PERSIST ACROSS REBOOTS
-------------------------------------------------------
echo 'd /var/run/python-validity 0755 root root -' | sudo tee /etc/tmpfiles.d/python-validity.conf


STEP 5 - FACTORY RESET THE SENSOR
-----------------------------------
sudo systemctl mask python3-validity
sudo systemctl stop python3-validity
sudo pkill -9 -f dbus-service
sudo python3 /usr/share/python-validity/playground/factory-reset.py

NOTE: Masking the service before factory reset is critical — if systemd
restarts it in the background it will grab the USB device and the reset
will fail with "Resource busy".

A successful factory reset prints nothing. Any traceback means it failed.


STEP 6 - START AND ENABLE SERVICES
------------------------------------
sudo systemctl unmask python3-validity
sudo systemctl enable python3-validity
sudo systemctl start python3-validity
sudo systemctl add-wants multi-user.target open-fprintd.service
sleep 8
sudo systemctl start open-fprintd


STEP 7 - FIX SERVICE ORDERING
------------------------------
Ensures open-fprintd always waits for python3-validity to fully initialize:

sudo mkdir -p /etc/systemd/system/open-fprintd.service.d
sudo tee /etc/systemd/system/open-fprintd.service.d/after-validity.conf << EOF
[Unit]
After=python3-validity.service
Requires=python3-validity.service
EOF
sudo systemctl daemon-reload


STEP 8 - ENROLL YOUR FINGERPRINT
----------------------------------
fprintd-enroll

Follow the prompts and swipe your finger several times.


STEP 9 - ENABLE FINGERPRINT FOR SUDO AND LOGIN
------------------------------------------------
sudo authselect enable-feature with-fingerprint
sudo authselect apply-changes

Test it:
sudo echo "works"


STEP 10 - FIX FINGERPRINT AFTER SUSPEND/RESUME
------------------------------------------------
sudo tee /usr/lib/systemd/system-sleep/fprint_wakeup << EOF
#!/bin/bash
if [ "\${1}" = "post" ]; then
    systemctl restart python3-validity
    systemctl restart open-fprintd
fi
EOF
sudo chmod +x /usr/lib/systemd/system-sleep/fprint_wakeup


STEP 11 - OPTIONAL: INCREASE SUDO TIMEOUT
------------------------------------------
sudo visudo

Add this line:
    Defaults timestamp_timeout=60

Value is in minutes. Use -1 to never ask again (not recommended).


TROUBLESHOOTING
---------------
Problem:  validity-sensors-firmware fails with "Is a directory" error
Fix:      sudo mkdir -p /var/run/python-validity

Problem:  factory-reset.py fails with "Resource busy"
Fix:      sudo systemctl mask python3-validity
          sudo systemctl stop python3-validity
          sudo pkill -9 -f dbus-service
          Then retry factory reset.

Problem:  factory-reset.py fails with "Exception: Failed: 0401" or "0404"
Fix:      Disable Predesktop Authentication in BIOS (Step 2).
          Then do a full shutdown (not reboot), wait 15 seconds, power on,
          and retry the factory reset before starting any services.

Problem:  fprintd-enroll returns "NoSuchDevice"
Fix:      python3-validity probably hasn't finished initializing.
          Check: journalctl -u python3-validity -n 20 --no-pager
          Look for "Manager is back online, registering"
          Then: sudo systemctl restart open-fprintd

Problem:  Fingerprint stops working after suspend
Fix:      Make sure Step 10 (fprint_wakeup script) is in place.
