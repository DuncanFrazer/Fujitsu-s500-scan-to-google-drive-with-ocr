# Fujitsu-s500-scan-to-google-drive-with-ocr

A snapshot of files that I use on a Raspberry Pi v4 connected to a Fujitsu S500 scanner.

ScanBD 1.4.4 is installed on the PI along with ocrmypdf, img2pdf, incron, and rclone

I followed this page to set up scanbd - I only looked at the local sections, I didn't bother with the sane.d or xinetd steps : https://sodocumentation.net/raspberry-pi/topic/6701/create-a-scan-station-with-scanbd--raspbian-

rclone is configured to connect to my google drive instance : https://medium.com/@artur.klauser/mounting-google-drive-on-raspberry-pi-dd15193d8138

scanbd uses the action.scan script in this repo when the scan button is pressed on my scanner - the output of the script in action.scan is a pdf of all sheets of a scan produced by img2pdf. img2pdf is fast enough that the Pi is ready to action the next Scanner button press as soon as the next document is loaded into the scanner. The scanned pdf is moved to a `_to_be_ocred` directory on my Pi

incrontab is monitoring 2 directories
- watch `_to_be_ocred` and call ocrmypdf when a new pdf arrives. ocrmypdf is realtively slow and would block the Pi from actioning a new scan if I included ocrmypdf in the action.scan script. Decoupling the process of OCRing the PDF with incron monitoring this directory allows scans to run quickly and the Pi catches up with the OCR action. When ocrmypdf is finished, the incron entry moves the pdf to a second directory `_to_mv_to_gdrive`
- the second incron entry tests to ensure the rclone mount of my google drive is up (I've seen it fall over sometimes) and if it is then move ocred pdfs to my mounted google drive. If the test fails (if the rclone mount is down) then the ocred pdf sits in the `_to_mv_to_gdrive` folder to ensure it is not lost. On the next scan the test will be performed again, and if passes then all ocred pdfs will be moved (new and waiting pdfs) ... or I can manually establish the rclone mount and move the files from _to_mv_to_gdrive myself ... but this way I know that none of my scan pdfs will be lost/overwritten ... especially important because I shred the originals!

### 1. Configure USB Permissions for `saned`

Create a custom udev rule so that `saned` retains access to the Fujitsu S500 (`04c5:10fe`) after `scanbd` drops root privileges:

```bash
# Create the udev rule file
sudo bash -c 'cat <<EOF> /etc/udev/rules.d/99-fujitsu-sane.rules
SUBSYSTEM=="usb", ATTRS{idVendor}=="04c5", ATTRS{idProduct}=="10fe", MODE="0666", GROUP="lp"
EOF'

# Add the saned user to hardware groups
sudo usermod -aG scanner,lp,plugdev saned

# Reload udev rules
sudo udevadm control --reload-rules
sudo udevadm trigger
```
---

### 2. Configure Systemd Service & Auto-Recovery

To ensure `scanbd` runs headlessly at boot and automatically recovers if the USB bus resets or the scanner enters power-saving mode, set up a systemd service override:

```bash
# Create systemd override directory
sudo mkdir -p /etc/systemd/system/scanbd.service.d

# Write override configuration
sudo bash -c 'cat <<EOF> /etc/systemd/system/scanbd.service.d/override.conf
[Service]
ExecStart=
ExecStart=/usr/sbin/scanbd -f -c /etc/scanbd/scanbd.conf
Restart=always
RestartSec=5s
EOF'

# Reload systemd, enable service on boot, and start
sudo systemctl daemon-reload
sudo systemctl enable scanbd
sudo systemctl restart scanbdd
```
---

## Troubleshooting

### Button Presses Not Triggering
If `scanbd` detects the scanner on USB reconnect but ignores button presses:
1. Verify `saned` group membership (`groups saned` should include `lp` and `scanner`).
2. Run `scanbd` manually in high-debug mode to inspect button polling:
   ```bash
   sudo systemctl stop scanbd
   sudo /usr/local/sbin/scanbd -f -d7 -c /usr/local/etc/scanbd/scanbd.conf
   ```
