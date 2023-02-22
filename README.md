# Fujitsu-s500-scan-to-google-drive-with-ocr

A snapshot of files that I use on a Raspberry Pi v4 connected to a Fujitsu S500 scanner.

ScanBD 1.4.4 is installed on the PI along with ocrmypdf, img2pdf, and rclone

I followed this page to set up scanbd - I only looked at the local sections, I didn't bother with the sane.d or xinetd steps : https://sodocumentation.net/raspberry-pi/topic/6701/create-a-scan-station-with-scanbd--raspbian-

rclone is configured to connect to my google drive instance : https://medium.com/@artur.klauser/mounting-google-drive-on-raspberry-pi-dd15193d8138

scanbd uses the action.scan script in this repo when the scan button is pressed on my scanner - the output is a pdf of all sheets of a scan produced by img2pdf. img2pdf is fast enough that the Pi is ready to action the next Scanner button press as soon as the next document is loaded into the scanner. The scanned pdf is moved to a _to_be_ocred directory

incrontab is monitoring 2 directories
- watch _to_be_ocred and call ocrmypdf when a new pdf arrives. ocrmypdf is realtively slow and would block the Pi from actioning a new scan if I included ocrmypdf in the action.scan script. Decoupling with this process with this directory and watching with incrontab allows scans to run quickly and the Pi catches up with the OCR action. When ocrmypdf is finished the incrontab entry moves the pdf to _to_mv_to_gdrive
- the second incrontab entry tests to ensure the rclone mount of my google drive is up (I've seen it fall over sometimes) and if up will move ocred pdf to my google drive. If the test fails (if the rclone mount is down) then the ocred pdf sits in the _to_mv_to_gdrive folder to ensure it is not lost. On the next scan the test will be performed again, and if passes then all ocred pdfs will be moved (new and waiting pdfs) ... or I can manually establish the rclone mount and move the files from _to_mv_to_gdrive myself ... but this way I know that none of my scan pdfs will be lost/overwritten ... especially important because I shred the originals!
